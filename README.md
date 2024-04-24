# GPG SSH YubiKey for MacOS

#### This manual provides detailed instructions on configuring YubiKey to operate as an advanced smart card, enabling high-security encryption, digital signing, and authentication mechanisms. 
#### YubiKey can be programmed to require manual tactile confirmation for executing cryptographic functions, substantially reducing the potential for unauthorized access to credentials.

## Compatible YubiKey

#### All current models of [YubiKey](https://www.yubico.com/pl/store/compare/), with the exception of the FIDO-only Security Key Series and Bio Series YubiKeys, are compatible with the protocols outlined in this guide.

## Set up the system environment

#### Download and install [Homebrew](https://brew.sh/) and the following packages:

```bash
brew install \
  gnupg yubikey-personalization ykman pinentry-mac wget
```
