# GPG SSH YubiKey for MacOS

#### This manual provides detailed instructions on configuring YubiKey to operate as an advanced smart card, enabling high-security encryption, digital signing, and authentication mechanisms. 
#### YubiKey can be programmed to require manual tactile confirmation for executing cryptographic functions, substantially reducing the potential for unauthorized access to credentials.

## Compatible YubiKey

#### All current models of [YubiKey](https://www.yubico.com/pl/store/compare/), with the exception of the FIDO-only Security Key Series and Bio Series YubiKeys, are compatible with the protocols outlined in this guide.

## Set up the system environment

#### Download and install [Homebrew](https://brew.sh/) and the following packages:

```bash
brew install --cask gpg-suite
```

```bash
brew install \
    yubikey-personalization ykman pinentry-mac wget
```

## Configure the GPG system

#### Create a temporary directory that will be deleted after a system reboot and assign it as the directory for GPG:

### CHANGE YOUR USER NAME!

```bash
export GNUPGHOME=/Users/macos/.gnupg
```

```bash
cd $GNUPGHOME
wget https://raw.githubusercontent.com/kacpowsky/<your_repository_name>/<branch_name>/<file_name>.<extension_name>
```