+++ 
draft = true
date = 2025-07-19T12:00:00-04:00
title = "GnuPG and Yubikeys"
description = ""
slug = ""
authors = ['Aaron Bauman']
tags = []
categories = ['GnuPG', 'Yubikey', 'Security']
externalLink = ""
series = []
+++

## Introduction
Public key cryptography is, thankfully, becoming more common and accessible. One
of my favorite setups is using a [Yubikey](https://www.yubico.com) paired with
[GnuPG](https://www.gnupg.org/). This is ideal for contributing to Free and Open
Source Software (FOSS), personal use, and really anywhere you intend to add a
layer of security.

### Hardware Keys
There are many options available in the hardware key world. There are many
aspects of choosing which key is right for the individual; however, I weigh
several areas that are most important to me, overall security, performance, and
open source availability. As a former Gentoo developer, the
[Foundation](https://bugs.gentoo.org/659620) provided Nitrokeys to all
developers. If my memory is correct, the Nitrokeys met all requirements for
available cryptography and open source, but lacked in performance compared to
Yubico. Overall, weigh each of these to ensure you choose the best for you.

### Yubikey
Yubico offers a lot of hardware keys with various configurations and levels of
testing. While this article is not meant to dive too deep, Yubico does offer
FIPS compliance with their Series 5 which is ideal for regulated industries and
the key I would recommend to those considering.

### GnuPG
[GnuPG](https://www.gnupg.org/) has a long history, is RFC compliant, is Free and
Open Source Software, and cross platform. Further, it has great support across
many frontends and is integrated via many libraries. The downside is managing
GPG keys can be a burden dependent upon implementation. This article will manage
GPG keys via the command line.

## Configuration
This article will cover two parts of implementation. Generating the respective
keys within GnuPG and adding those to the hardware key. I will be focusing on
MacOS but dependencies and configuration will generally be the same on your
favorite Linux distro.

### GnuPG
```bash
brew install gnupg pinentry-mac

```

```bash
echo "use-agent cert-digest-algo SHA512" > ~/.gnupg/gpg.conf
```

The second configuration item sets SHA-512 as the preferred digest algorithm.

Before generating keys, please ensure you understand the specifications of the
hardware token. That is, what cryptography it can handle, etc. For me, I am
using a [Yubikey
4](https://support.yubico.com/hc/en-us/articles/360013714599-YubiKey-4?gad_source=1&gad_campaignid=22540972794)
which is capable of 4096 bits (e.g. RSA 4096) with OpenPGP (i.e. GnuPG).

#### Generating Master Key
The first step is to generate a master key. This will be followed up by
generating subkeys which will be moved to the hardware key. Finally, we will
backup the master key, store it offline, and delete it from the keyring to enhance
our security posture.

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
Your selection? 8

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

As seen above, GnuPG offers (4) actions allowed by our GnuPG key, Sign, Certify,
Authenticate, and Encrypt. By default, (3) of these are enabled. We will toggle
off Sign and Encrypt leaving Certify as the only option.

As this is the master key, certifying is the only action it will be used
for. Certifying is used to sign others GPG keys, generate new subkeys, extend
expiry, and revoking the key. Given this set of actions, accessing this key is
considered a high threat which may lead to compromising access to critical
infrastructure, code bases, and other places the key is used.

The other option and choice here is to set the expiry for the key. I have chosen
1 year and advocate this for several reasons. Should an individual lose access
to the key or revocation certificate, the key will eventually expire. Ideally,
this is handled through proper revocation. The other aspect is, we can extend
the expiry of the key as it's used. This allows the individual to maintain
proficiency in managing the key.

#### Revocation Certificate
Highly important to this process is generating the revocation certificate. This
certificate should be treated equally to the master key. Backed up and stored
offline.

```bash
gpg --gen-revoke 70334C09AD28CF3CC05F38CAE854050599A57489 >
~/key-revocation-certificate.asc
```

The above will generate the revocation certificate. Feel free to save it to
whatever file makes sense to you with a .asc extension.

#### Subkey Generation
In this phase, we will generate subkeys for daily use on the hardware key. There
will be (3) keys generated for Signing, Authentication, and Encryption.

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
gpg> save
```

The final key generated is a bit different, in that we must set our own
capabilities. Hence, the toggling of features, so pay extra attention as you
generate it! 

Finally, ensure you save the key.

I prefer to generate all keys within GnuPG and back them up. This also allows
you to use multiple hardware keys and not experience issues decrypting,
authenticating, etc. It simply makes it easier. You may see other articles
written that suggest generating the Signing and Authentication keys on the
hardware key. Additionally, should a hardware key break, you are able to simply
purchase a new one and load the keys.

#### Export Secret Key(s)
This is absolutely critical. Ensuring you have a proper backup of keys is paramount.

```bash
gpg --export-secret-key 70334C09AD28CF3CC05F38CAE854050599A57489 >
~/70334C09AD28CF3CC05F38CAE854050599A57489-secret.pgp

gpg --export-secret-subkeys --armor  70334C09AD28CF3CC05F38CAE854050599A57489 > ~/subkeys.asc
```

#### Copying keys to hardware key
*Note: This is a destructive process!*

When you copy a key to the card it will permanently remove it from the GnuPG
keyring. If you have not backed up your keys, per previous step, then do not proceed.

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
Take note of the * next to the Encryption key which ensures we selected the
proper key and are moving it to the hardware key.

Repeat this process for the Signing and Authentication keys.

If you are making use of a second hardware key, then you will need to restore
the backup to the keychain.

### Offline Master Key
Now that all keys have been generated and moved to the Yubikey, the master key
can be removed from the local keychain. Again, do not proceed here if you have
not properly backed up the master key as described above.

The below command will list the keygrips which are needed to remove the key.

```bash
sec   rsa4096 2025-07-19 [C] [expires: 2026-07-19]
      70334C09AD28CF3CC05F38CAE854050599A57489
      Keygrip = 7F2A5095D3CBFB74199596DFB39EBE4D4A734A7A
uid           [ultimate] John Doe <johnny@doe.com>
```

```bash
gpg-connect-agent "DELETE_KEY 7F2A5095D3CBFB74199596DFB39EBE4D4A734A7A" /bye
```

### Publishing Keys
There are multiple public keyservers to share your key with. I prefer using
[OpenPGP](https://keys.openpgp.org/) servers. Here are a couple of examples to
upload keys to the servers. I personally prefer the second example.

```bash
gpg --send-keys --keyserver hkps://keys.openpgp.org
7F2A5095D3CBFB74199596DFB39EBE4D4A734A7A
```

```bash
gpg --export johnny@doe.com | curl -T - https://keys.openpgp.org
```

That's all for this post. I may cover more in depth discussions around GnuPG and
hardware in other posts. I hope this helps!
