# GPG SSH YubiKey for MacOS (2024)

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

## Environment configuration

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
gpg --batch --pinentry-mode=loopback --passphrase "$CERTIFY_PASS" --quick-add-key "$KEYFP" ed25519 sign "$EXPIRATION"
```
```bash
gpg --batch --pinentry-mode=loopback --passphrase "$CERTIFY_PASS" --quick-add-key "$KEYFP" cv25519 encrypt "$EXPIRATION"
```
```bash
gpg --batch --pinentry-mode=loopback --passphrase "$CERTIFY_PASS" --quick-add-key "$KEYFP" ed25519 auth "$EXPIRATION"
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

## Configure YubiKey

Connect YubiKey and confirm its status:

```bash
gpg --card-status
```

Set PINs manually or generate them

```bash
ADMIN_PIN=$(LC_ALL=C tr -dc '0-9' < /dev/urandom | fold -w8 | head -1)

USER_PIN=$(LC_ALL=C tr -dc '0-9' < /dev/urandom | fold -w6 | head -1)

printf "\nAdmin PIN: %12s\nUser PIN: %13s\n\n" "$ADMIN_PIN" "$USER_PIN"
```

Change the Admin PIN:

```bash
gpg --command-fd=0 --pinentry-mode=loopback --change-pin <<EOF
3
12345678
$ADMIN_PIN
$ADMIN_PIN
q
EOF
```

Change the User PIN:

```bash
gpg --command-fd=0 --pinentry-mode=loopback --change-pin <<EOF
1
123456
$USER_PIN
$USER_PIN
q
EOF
```

Remove and re-insert YubiKey.

Connect YubiKey and confirm its status:

```bash
gpg --card-status
```

You can modify the number of retry attempts:

```bash
ykman openpgp access set-retries 5 5 5 -f -a $ADMIN_PIN
```

Set the smart card attributes with admin mode:

```bash
gpg --command-fd=0 --pinentry-mode=loopback --edit-card <<EOF
admin
login
$IDENTITY
$ADMIN_PIN
quit
EOF
```

Run `gpg --card-status` to verify results.

## Transfer Subkeys

### Signature key

```bash
gpg --edit-key $KEYID
```
`key 1`

`keytocard`

select `1` for Signature key

confirm with CERTIFY_PASS

confirm with ADMIN_PIN

`save`

### Encryption key

```bash
gpg --edit-key $KEYID
```
`key 2`

`keytocard`

select `2` for Encryption key

confirm with CERTIFY_PASS

confirm with ADMIN_PIN

`save`

### Authentication key

```bash
gpg --edit-key $KEYID
```
`key 3`

`keytocard`

select `3` for Authentication key

confirm with CERTIFY_PASS

confirm with ADMIN_PIN

`save`

## Verify Subkeys

Verify Subkeys have been moved to YubiKey with `gpg -K` and look for `ssb>`, for example:

```bash
sec   ed25519/0xF0F2CFEB04341FB5 2024-05-01 [C]
      Key fingerprint = 4E2C 1FA3 372C BA96 A06A  C34A F0F2 CFEB 0434 1FB5
uid                   [ultimate] YubiKey User <yubikey@example>
ssb>  ed25519/0xB3CD10E502E19637 2024-05-01 [S] [expires: 2026-05-01]
ssb>  cv25519/0x30CBE8C4B085B9F7 2024-05-01 [E] [expires: 2026-05-01]
ssb>  ed25519/0xAD9E24E1B8CB9600 2024-05-01 [A] [expires: 2026-05-01]
```

If you have `>` after a tag indicates the key is on a smart card.

### Using YubiKey

```bash
cd ~/.gnupg
```
```bash
touch scdaemon.conf

echo "disable-ccid" >>scdaemon.conf
```

```bash
gpg --card-edit
```

```bash
gpg/card> fetch

gpg/card> quit
```

Remove and re-insert YubiKey.

Verify the status with `gpg --card-status` which will list the available Subkeys:

```bash
Reader ...........: Yubico YubiKey OTP FIDO CCID 00 00
Application ID ...: D27600012401020100060578934512
Application type .: OpenPGP
Version ..........: 3.4
Manufacturer .....: Yubico
Serial number ....: 28926784
Name of cardholder: YubiKey User
Language prefs ...: en
Salutation .......:
URL of public key : [not set]
Login data .......: example@example
Signature PIN ....: not forced
Key attributes ...: ed25519 cv25519 ed25519
Max. PIN lengths .: 127 127 127
PIN retry counter : 5 0 5
Signature counter : 0
KDF setting ......: on
Signature key ....: CF5A 305B 808B 7A0F 230D  A064 B3CD 10E5 02E1 9637
      created ....: 2024-01-01 12:00:00
Encryption key....: A5FA A005 5BED 4DC9 889D  38BC 30CB E8C4 B085 B9F7
      created ....: 2024-01-01 12:00:00
Authentication key: 570E 1355 6D01 4C04 8B6D  E2A3 AD9E 24E1 B8CB 9600
      created ....: 2024-01-01 12:00:00
General key info..: sub  ed25519/0xB3CD10E502E19637 2024-05-01 YubiKey User <example@example>
sec#  ed25519/0xF0F2CFEB04341FB5  created: 2024-05-01  expires: never
ssb>  ed25519/0xB3CD10E502E19637  created: 2024-05-01  expires: 2026-05-01
                                  card-no: 0006 05553211
ssb>  cv25519/0x30CBE8C4B085B9F7  created: 2024-05-01  expires: 2026-05-01
                                  card-no: 0006 05553211
ssb>  ed25519/0xAD9E24E1B8CB9600  created: 2024-05-01  expires: 2026-05-01
                                  card-no: 0006 05553211
```


### Configure touch

Encryption:

```bash
ykman openpgp keys set-touch dec on
```
Versions of YubiKey Manager before 5.1.0 use `enc` instead of `dec` for encryption:

```bash
ykman openpgp keys set-touch enc on
```

Signature:

```bash
ykman openpgp keys set-touch sig on
```

Authentication:

```bash
ykman openpgp keys set-touch aut on
```

## SSH

```bash
cd ~/.gnupg

wget https://raw.githubusercontent.com/kacpowsky/gpgYubiKey/config/gpg-agent.conf
```

Add the following to the shell rc file:

for example: `./zshrc`

```bash
nano ./zshrc
```

Add

```bash
export GPG_TTY="$(tty)"
export SSH_AUTH_SOCK=$(gpgconf --list-dirs agent-ssh-socket)
gpgconf --launch gpg-agent
```

and

```bash
export SSH_AUTH_SOCK=$(gpgconf --list-dirs agent-ssh-socket)
```

```bash
source ./zshrc
```

### Last Step

Reload gpg-agent:

```bash
gpg-connect-agent killagent /bye

gpg-connect-agent /bye
```

Reload ssh-agent:

```bash
eval $(ssh-agent -s)
```

Remove and re-insert YubiKey.

Export the SSH public key:

```bash
gpg --export-ssh-key <public key id>
```

Check your ssh agent:

```bash
$ ssh-add -L
ssh-ed25519 AAAAB4NzaC1yc2EAAAADAQABAAACAz[...]zreOKM+HwpkHzcy9DQcVG2Nw== cardno:000605553211
```

 If you see your SSH public key, you have completed all the steps correctly! 

### You can now enjoy your GPG SSH YubiKey authentication

# Additional resources

- [GPG Suite](https://gpgtools.org/)
- [EdDSA](https://en.wikipedia.org/wiki/EdDSA)
- [Yubico - PGP](https://developers.yubico.com/PGP/)
- [Yubico - Yubikey Personalization](https://developers.yubico.com/yubikey-personalization/)