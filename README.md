# GPG SSH YubiKey for MacOS

#### This manual provides detailed instructions on configuring YubiKey to operate as an advanced smart card, enabling high-security encryption, digital signing, and authentication mechanisms. 
#### YubiKey can be programmed to require manual tactile confirmation for executing cryptographic functions, substantially reducing the potential for unauthorized access to credentials.

## Compatible YubiKey

All current models of [YubiKey](https://www.yubico.com/pl/store/compare/), with the exception of the FIDO-only Security Key Series and Bio Series YubiKeys, are compatible with the protocols outlined in this guide.

## Set up the system environment

Download and install [Homebrew](https://brew.sh/) and the following packages:

```bash
brew install --cask gpg-suite
```

```bash
brew install \
    yubikey-personalization ykman pinentry-mac wget
```

## Configure the GPG system

Create a temporary directory that will be deleted after a system reboot and assign it as the directory for GPG:


```bash
export GNUPGHOME=/Users/macos/.gnupg
```

```bash
cd $GNUPGHOME
wget https://raw.githubusercontent.com/kacpowsky/gpgYubiKey/config/gpg.conf
```

The options will look similar to:

```bash
$ grep -ve "^#" $GNUPGHOME/gpg.conf
auto-key-retrieve
#no-emit-version
personal-cipher-preferences AES256 AES192 AES
personal-digest-preferences SHA512 SHA384 SHA256
personal-compress-preferences ZLIB BZIP2 ZIP Uncompressed
default-preference-list SHA512 SHA384 SHA256 AES256 AES192 AES ZLIB BZIP2 ZIP Uncompressed
cert-digest-algo SHA512
s2k-digest-algo SHA512
s2k-cipher-algo AES256
charset utf-8
no-comments
no-emit-version
no-greeting
keyid-format 0xlong
list-options show-uid-validity
verify-options show-uid-validity
with-fingerprint
require-cross-certification
no-symkey-cache
armor
use-agent
throw-keyids
```

## Configuration of environment variables

### IDENTITY

The default options ask for a "Real name", "Email address" and optional "Comment":

```bash
IDENTITY="YubiKey User <yubikey@example>"
```
or 

```bash
IDENTITY="My super cool name 2024"
```

### KEY

This guide recommends ed25519

```bash
KEY_TYPE=ed25519
```

### Expiration

```bash
EXPIRATION=2y
```

### Passphrase

```bash
CERTIFY_PASS=$(LC_ALL=C tr -dc 'A-Z1-9' < /dev/urandom | \
  tr -d "1IOS5U" | fold -w 30 | sed "-es/./ /"{1..26..5} | \
  cut -c2- | tr " " "-" | head -1) ; echo "$CERTIFY_PASS"
```

Write the passphrase in a secure location, ideally separate from the portable storage device used for key material, or memorize it.

## Generate a certificate key

The primary key to be generated is the Certify key, which is responsible for issuing Subkeys for encryption, signing, and authentication operations.

Do not set an expiration date for the Certify key.

Generate the Certify key:

```bash
gpg --batch --passphrase "$CERTIFY_PASS" \
    --quick-generate-key "$IDENTITY" "$KEY_TYPE" cert never
```
Configure and retrieve the identifier and fingerprint of the Certify key for future reference:

```bash
KEYID=$(gpg -k --with-colons "$IDENTITY" | awk -F: '/^pub:/ { print $5; exit }')

KEYFP=$(gpg -k --with-colons "$IDENTITY" | awk -F: '/^fpr:/ { print $10; exit }')

printf "\nKey ID: %40s\nKey FP: %40s\n\n" "$KEYID" "$KEYFP"
```

## Create Subkeys

```bash
gpg --batch --pinentry-mode=loopback --passphrase "$CERTIFY_PASS"     --quick-add-key "$KEYFP" ed25519 sign "$EXPIRATION"
```
```bash
gpg --batch --pinentry-mode=loopback --passphrase "$CERTIFY_PASS"     --quick-add-key "$KEYFP" cv25519 encrypt "$EXPIRATION"
```
```bash
gpg --batch --pinentry-mode=loopback --passphrase "$CERTIFY_PASS"     --quick-add-key "$KEYFP" ed25519 auth "$EXPIRATION"
```

## Verify keys

```bash
gpg -K
```

The output will display [C]ertify, [S]ignature, [E]ncryption and [A]uthentication keys:

```bash
sec   ed25519/0xF0F2CFEB04341FB5 2024-05-01 [C]
      Key fingerprint = 4E2C 1FA3 372C BA96 A06A  C34A F0F2 CFEB 0434 1FB5
uid                   [ultimate] YubiKey User <yubikey@example>
ssb   ed25519/0xB3CD10E502E19637 2024-05-01 [S] [expires: 2026-05-01]
ssb   cv25519/0x30CBE8C4B085B9F7 2024-05-01 [E] [expires: 2026-05-01]
ssb   ed25519/0xAD9E24E1B8CB9600 2024-05-01 [A] [expires: 2026-05-01]
```

## Backup keys

```bash
gpg --output $GNUPGHOME/$KEYID-Certify.key \
    --batch --pinentry-mode=loopback --passphrase "$CERTIFY_PASS" \
    --armor --export-secret-keys $KEYID

gpg --output $GNUPGHOME/$KEYID-Subkeys.key \
    --batch --pinentry-mode=loopback --passphrase "$CERTIFY_PASS" \
    --armor --export-secret-subkeys $KEYID

gpg --output $GNUPGHOME/$KEYID-$(date +%F).asc \
    --armor --export $KEYID
```

