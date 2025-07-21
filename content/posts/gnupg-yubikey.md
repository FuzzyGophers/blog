+++
draft = false
date = 2025-07-19T12:00:00-04:00
title = "GnuPG and Yubikey"
description = "Using GnuPG generated keys on a Yubikey hardware token"
slug = ""
authors = ['Aaron Bauman']
tags = []
categories = ['GnuPG', 'Yubikey', 'Security']
externalLink = ""
series = []
+++

## Introduction

Public key cryptography is increasingly common and accessible, providing robust
security for various applications. Combining a [Yubikey](https://www.yubico.com)
with [GnuPG](https://gnupg.org/) offers a powerful setup for securing Free and
Open Source Software (FOSS) contributions, personal communications, and other
use cases requiring enhanced security. This guide outlines the process of
generating GnuPG keys and integrating them with a Yubikey hardware token.

### Hardware Tokens

Numerous hardware token options exist, each with unique characteristics. Key
considerations for selecting a token include overall security, performance
(e.g., speed of cryptographic operations), and open source availability. For
example, the [Gentoo Foundation](https://bugs.gentoo.org/659620) provided
Nitrokeys to developers, which met cryptography and open source requirements but
were less performant compared to Yubico’s offerings. Evaluate these factors to
choose the most suitable token for specific needs.

### Yubikey

Yubico provides a range of hardware tokens with varying configurations and
compliance levels. The Yubikey 5 Series, with FIPS compliance, is recommended
for regulated industries due to its robust security features. This guide
references the
[Yubikey 4](https://support.yubico.com/hc/en-us/articles/360013714599-YubiKey-4),
which supports RSA 4096-bit keys with OpenPGP, though newer models like the
Yubikey 5 are preferred.

### GnuPG

[GnuPG](https://gnupg.org/) is a long-standing, RFC-compliant, Free and Open
Source Software (FOSS) tool that operates across multiple platforms. It
integrates well with various frontends and libraries, making it versatile for
secure communication and authentication. However, managing GPG keys can be
complex depending on the implementation. This guide focuses on command-line key
management.

## Prerequisites

Before proceeding, ensure the following:

- A configured Yubikey hardware token (e.g., Yubikey 4 or 5) is available.
- Basic familiarity with terminal commands is assumed.
- For macOS, Homebrew is installed (`brew`). For Linux, a package manager like
  `portage` or `dnf` is required. For Windows, tools like Gpg4win are
  recommended.
- An air-gapped system (optional but recommended) for generating and backing up
  the keys to minimize exposure.

## Configuration

This guide covers two primary tasks: generating GnuPG keys and transferring them
to a Yubikey. The instructions focus on macOS, but dependencies and
configurations are generally similar for Linux distributions. Windows users may
need to adapt commands using tools like Gpg4win.

### GnuPG

```bash
brew install gnupg pinentry-mac

```

Set the preferred digest algorithm to SHA-512

```bash
echo "use-agent cert-digest-algo SHA512" > ~/.gnupg/gpg.conf
```

Ensure the Yubikey supports the chosen cryptography (e.g., RSA 4096 for Yubikey
4). RSA is used here for compatibility, though ECC is the default in modern
GnuPG due to its efficiency.

#### Generating Master Key

Generate a master key for certification purposes, followed by subkeys for daily
use on the Yubikey. Back up the master key securely and remove it from the local
keyring to enhance security.

**Optional: Perform all operations on an air-gapped machine until all keys are
backed up and deleted from the keyring to minimize exposure.**

Generate key in expert mode to set custom capabilities:

```bash
captainamerica@tater:~$ gpg --full-generate-key --expert
gpg (GnuPG) 2.4.8; Copyright (C) 2025 g10 Code GmbH
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.

Please select what kind of key you want:
   (1) RSA and RSA
   (2) DSA and Elgamal
   (3) DSA (sign only)
   (4) RSA (sign only)
   (7) DSA (set your own capabilities)
   (8) RSA (set your own capabilities)
   (9) ECC (sign and encrypt) *default*
  (10) ECC (sign only)
  (11) ECC (set your own capabilities)
  (13) Existing key
  (14) Existing key from card
```

Choose "RSA (set your own capabilities)":

```bash
Your selection? 8
```

Toggle off Signing and Encryption, leaving Certify:

```bash
Possible actions for this RSA key: Sign Certify Encrypt Authenticate
Current allowed actions: Sign Certify Encrypt

   (S) Toggle the sign capability
   (E) Toggle the encrypt capability
   (A) Toggle the authenticate capability
   (Q) Finished

Your selection? S

Possible actions for this RSA key: Sign Certify Encrypt Authenticate
Current allowed actions: Certify Encrypt

   (S) Toggle the sign capability
   (E) Toggle the encrypt capability
   (A) Toggle the authenticate capability
   (Q) Finished

Your selection? E

Possible actions for this RSA key: Sign Certify Encrypt Authenticate
Current allowed actions: Certify

   (S) Toggle the sign capability
   (E) Toggle the encrypt capability
   (A) Toggle the authenticate capability
   (Q) Finished

Your selection? Q
```

Set key length to 4096 bits and expiration to 1 year:

```bash
RSA keys may be between 1024 and 4096 bits long.
What keysize do you want? (3072) 4096
Requested keysize is 4096 bits
Please specify how long the key should be valid.
         0 = key does not expire
      <n>  = key expires in n days
      <n>w = key expires in n weeks
      <n>m = key expires in n months
      <n>y = key expires in n years
Key is valid for? (0) 1y
Key expires at Sun Jul 19 17:11:46 2026 EDT
Is this correct? (y/N) y
```

Enter identifying information:

```bash
GnuPG needs to construct a user ID to identify your key.

Real name: John Doe
Email address: johnny@doe.com
Comment:
You selected this USER-ID:
    "John Doe <johnny@doe.com>"

Change (N)ame, (C)omment, (E)mail or (O)kay/(Q)uit? O
We need to generate a lot of random bytes. It is a good idea to perform
some other action (type on the keyboard, move the mouse, utilize the
disks) during the prime generation; this gives the random number
generator a better chance to gain enough entropy.
gpg: directory '/Users/aaronbauman/.gnupg/openpgp-revocs.d' created
gpg: revocation certificate stored as '/Users/aaronbauman/.gnupg/openpgp-revocs.d/70334C09AD28CF3CC05F38CAE854050599A57489.rev'
public and secret key created and signed.

pub   rsa4096 2025-07-19 [C] [expires: 2026-07-19]
      70334C09AD28CF3CC05F38CAE854050599A57489
uid                      John Doe <johnny@doe.com>
```

The master key is configured for certification only, used for signing other
keys, generating subkeys, extending expiration, or revoking keys. This
restricted role minimizes risk, as compromising the master key could affect
critical infrastructure or trusted code bases. A 1-year expiration ensures the
key becomes invalid if lost, though it can be extended as needed. Regular
interaction with the key maintains proficiency in its management.

#### Revocation Certificate

Generate a revocation certificate to allow key revocation, if needed. Store it
securely, treating it with the same care as the master key:

```bash
gpg --gen-revoke 70334C09AD28CF3CC05F38CAE854050599A57489 >
~/key-revocation-certificate.asc
```

#### Subkey Generation

Generate three subkeys for signing, encryption, and authentication to handle
daily operations on the Yubikey.

Edit the key:

```bash
captainamerica@tater:~$ gpg --expert --edit-key 70334C09AD28CF3CC05F38CAE854050599A57489
gpg (GnuPG) 2.4.8; Copyright (C) 2025 g10 Code GmbH
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.

Secret key is available.

sec  rsa4096/E854050599A57489
     created: 2025-07-19  expires: 2026-07-19  usage: C
     trust: ultimate      validity: ultimate
[ultimate] (1). John Doe <johnny@doe.com>
```

Generate the Encryption subkey:

```bash
gpg> addkey
Please select what kind of key you want:
   (3) DSA (sign only)
   (4) RSA (sign only)
   (5) Elgamal (encrypt only)
   (6) RSA (encrypt only)
  (10) ECC (sign only)
  (12) ECC (encrypt only)
  (14) Existing key from card
Your selection? 6
RSA keys may be between 1024 and 4096 bits long.
What keysize do you want? (3072) 4096
Requested keysize is 4096 bits
Please specify how long the key should be valid.
         0 = key does not expire
      <n>  = key expires in n days
      <n>w = key expires in n weeks
      <n>m = key expires in n months
      <n>y = key expires in n years
Key is valid for? (0) 1y
Key expires at Sun Jul 19 17:31:17 2026 EDT
Is this correct? (y/N) y
Really create? (y/N) y
We need to generate a lot of random bytes. It is a good idea to perform
some other action (type on the keyboard, move the mouse, utilize the
disks) during the prime generation; this gives the random number
generator a better chance to gain enough entropy.

sec  rsa4096/E854050599A57489
     created: 2025-07-19  expires: 2026-07-19  usage: C
     trust: ultimate      validity: ultimate
ssb  rsa4096/08FC38926722D573
     created: 2025-07-19  expires: 2026-07-19  usage: E
[ultimate] (1). John Doe <johnny@doe.com>
```

Generate the Signing subkey:

```bash
gpg> addkey
Please select what kind of key you want:
   (3) DSA (sign only)
   (4) RSA (sign only)
   (5) Elgamal (encrypt only)
   (6) RSA (encrypt only)
  (10) ECC (sign only)
  (12) ECC (encrypt only)
  (14) Existing key from card
Your selection? 4
RSA keys may be between 1024 and 4096 bits long.
What keysize do you want? (3072) 4096
Requested keysize is 4096 bits
Please specify how long the key should be valid.
         0 = key does not expire
      <n>  = key expires in n days
      <n>w = key expires in n weeks
      <n>m = key expires in n months
      <n>y = key expires in n years
Key is valid for? (0) 1y
Key expires at Sun Jul 19 17:32:21 2026 EDT
Is this correct? (y/N) y
Really create? (y/N) y
We need to generate a lot of random bytes. It is a good idea to perform
some other action (type on the keyboard, move the mouse, utilize the
disks) during the prime generation; this gives the random number
generator a better chance to gain enough entropy.

sec  rsa4096/E854050599A57489
     created: 2025-07-19  expires: 2026-07-19  usage: C
     trust: ultimate      validity: ultimate
ssb  rsa4096/08FC38926722D573
     created: 2025-07-19  expires: 2026-07-19  usage: E
ssb  rsa4096/16F2BF1AF721EC4D
     created: 2025-07-19  expires: 2026-07-19  usage: S
[ultimate] (1). John Doe <johnny@doe.com>
```

Generate the Authentication subkey, setting custom capabilities:

```bash
gpg> addkey
Please select what kind of key you want:
   (3) DSA (sign only)
   (4) RSA (sign only)
   (5) Elgamal (encrypt only)
   (6) RSA (encrypt only)
   (7) DSA (set your own capabilities)
   (8) RSA (set your own capabilities)
  (10) ECC (sign only)
  (11) ECC (set your own capabilities)
  (12) ECC (encrypt only)
  (13) Existing key
  (14) Existing key from card
Your selection? 8

Possible actions for this RSA key: Sign Encrypt Authenticate
Current allowed actions: Sign Encrypt

   (S) Toggle the sign capability
   (E) Toggle the encrypt capability
   (A) Toggle the authenticate capability
   (Q) Finished

Your selection? A

Possible actions for this RSA key: Sign Encrypt Authenticate
Current allowed actions: Sign Encrypt Authenticate

   (S) Toggle the sign capability
   (E) Toggle the encrypt capability
   (A) Toggle the authenticate capability
   (Q) Finished

Your selection? S

Possible actions for this RSA key: Sign Encrypt Authenticate
Current allowed actions: Encrypt Authenticate

   (S) Toggle the sign capability
   (E) Toggle the encrypt capability
   (A) Toggle the authenticate capability
   (Q) Finished

Your selection? E

Possible actions for this RSA key: Sign Encrypt Authenticate
Current allowed actions: Authenticate

   (S) Toggle the sign capability
   (E) Toggle the encrypt capability
   (A) Toggle the authenticate capability
   (Q) Finished

Your selection? Q
RSA keys may be between 1024 and 4096 bits long.
What keysize do you want? (3072) 4096
Requested keysize is 4096 bits
Please specify how long the key should be valid.
         0 = key does not expire
      <n>  = key expires in n days
      <n>w = key expires in n weeks
      <n>m = key expires in n months
      <n>y = key expires in n years
Key is valid for? (0) 1y
Key expires at Sun Jul 19 17:40:57 2026 EDT
Is this correct? (y/N) Y
Really create? (y/N) Y
We need to generate a lot of random bytes. It is a good idea to perform
some other action (type on the keyboard, move the mouse, utilize the
disks) during the prime generation; this gives the random number
generator a better chance to gain enough entropy.

sec  rsa4096/E854050599A57489
     created: 2025-07-19  expires: 2026-07-19  usage: C
     trust: ultimate      validity: ultimate
ssb  rsa4096/08FC38926722D573
     created: 2025-07-19  expires: 2026-07-19  usage: E
ssb  rsa4096/16F2BF1AF721EC4D
     created: 2025-07-19  expires: 2026-07-19  usage: S
ssb  rsa4096/010C39E986CEF8EB
     created: 2025-07-19  expires: 2026-07-19  usage: A
[ultimate] (1). John Doe <johnny@doe.com>
```

Finally, save changes!

```bash
gpg> save
```

Generating keys in GnuPG allows backups and supports multiple hardware tokens,
simplifying recovery if a Yubikey is lost or damaged. Alternatively, generating
keys directly on the Yubikey ensures they never leave the device but prevents
backups, complicating recovery and usability.

#### Export Secrets

Export the master key secret:

```bash
gpg --export-secret-key 70334C09AD28CF3CC05F38CAE854050599A57489 >
~/70334C09AD28CF3CC05F38CAE854050599A57489-secret.pgp
```

Export the subkeys secrets:

```bash
gpg --export-secret-subkeys --armor  70334C09AD28CF3CC05F38CAE854050599A57489 > ~/subkeys.asc
```

#### Copying keys to hardware token

**Note: This is a destructive process!**

Ensure backups are complete before proceeding as transferring keys to the
Yubikey removes them from the local GnuPG keyring.

```bash
gpg --expert --edit-key 70334C09AD28CF3CC05F38CAE854050599A57489

gpg (GnuPG) 2.4.8; Copyright (C) 2025 g10 Code GmbH
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.

Secret key is available.

sec  rsa4096/E854050599A57489
     created: 2025-07-19  expires: 2026-07-19  usage: C
     trust: ultimate      validity: ultimate
ssb  rsa4096/08FC38926722D573
     created: 2025-07-19  expires: 2026-07-19  usage: E
ssb  rsa4096/16F2BF1AF721EC4D
     created: 2025-07-19  expires: 2026-07-19  usage: S
ssb  rsa4096/010C39E986CEF8EB
     created: 2025-07-19  expires: 2026-07-19  usage: A
[ultimate] (1). John Doe <johnny@doe.com>

gpg> key 1
```

This will select the second listed key (Encryption key).

```bash
sec  rsa4096/E854050599A57489
     created: 2025-07-19  expires: 2026-07-19  usage: C
     trust: ultimate      validity: ultimate
ssb* rsa4096/08FC38926722D573
     created: 2025-07-19  expires: 2026-07-19  usage: E
ssb  rsa4096/16F2BF1AF721EC4D
     created: 2025-07-19  expires: 2026-07-19  usage: S
ssb  rsa4096/010C39E986CEF8EB
     created: 2025-07-19  expires: 2026-07-19  usage: A
[ultimate] (1). John Doe <johnny@doe.com>

gpg> keytocard
```

The asterisk (\*) indicates the selected key. Ensure only one key is selected.
Repeat for Signing and Authentication subkeys.

For multiple Yubikeys, restore subkey secrets from subkeys.asc to the keyring
before transferring to additional devices:

Example import of subkeys (this restores secrets to the local keychain):

```bash
gpg --import ~/subkeys.asc
```

### Offline Master Key

After transferring subkeys to the Yubikey, remove the master key from the local
keyring to enhance security. Ensure a backup exists before proceeding.

List keygrips to identify the master key:

```bash
gpg --list-keys --with-keygrip

sec   rsa4096 2025-07-19 [C] [expires: 2026-07-19]
      70334C09AD28CF3CC05F38CAE854050599A57489
      Keygrip = 7F2A5095D3CBFB74199596DFB39EBE4D4A734A7A
uid           [ultimate] John Doe <johnny@doe.com>
```

Delete the master key (this is permanent):

```bash
gpg-connect-agent "DELETE_KEY 7F2A5095D3CBFB74199596DFB39EBE4D4A734A7A" /bye
```

Verify the secret key is removed:

```bash
gpg --list-secret-keys
```

Confirm the master key stub (sec#):

```bash
-------------------------------------
sec#  rsa4096
```

Confirm the subkey stubs (ssb>):

```bash
ssb>  rsa4096
ssb>  rsa4096
ssb>  rsa4096
```

### Master Key Storage

Securely store the master and revocation keys to enable annual expiration
updates or revocation. Options include a USB drive in a safe, cloud storage with
multiple authentication layers (e.g., MFA), or a printed backup using Paperkey.
Combining methods is also recommended. Retain multiple backups across several
locations. The possibilities are endless, be creative based on your needs.

### Publishing Keys

Share the public key via keyservers like [OpenPGP](https://keys.openpgp.org/),
which requires email verification for privacy:

Alternative keyservers include pgp.mit.edu or keyserver.ubuntu.com. You may
choose to use multiple for redundancy. Ensure you know which keyservers you
chose in case of revocation. Most servers are not federated and will require
individual calls to revoke.

```bash
gpg --send-keys --keyserver hkps://keys.openpgp.org
70334C09AD28CF3CC05F38CAE854050599A57489
```

Preferred method:

```bash
gpg --export johnny@doe.com | curl -T - https://keys.openpgp.org
```
