<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->
**Table of Contents**

- [GPG Quick Guide](#gpg-quick-guide)
  - [Not covered](#not-covered)
  - [Prerequisites](#prerequisites)
  - [General recommendations](#general-recommendations)
  - [Definitions](#definitions)
      - [Algorithm List](#algorithm-list)
      - [Key related abbrs](#key-related-abbrs)
  - [Getting ready for key generation](#getting-ready-for-key-generation)
    - [Configuration File](#configuration-file)
  - [Testing/sandboxing](#testingsandboxing)
  - [Create Keypairs](#create-keypairs)
      - [Small summary of steps for key generation](#small-summary-of-steps-for-key-generation)
    - [Why do we need subkeys](#why-do-we-need-subkeys-sup2debiansup)
    - [Generating Master Key](#generating-master-key)
    - [Add User IDs and subkeys](#add-user-ids-and-subkeys)
      - [Add user ids](#add-user-ids)
      - [Add Subkeys](#add-subkeys)
      - [Wrapping up](#wrapping-up)
      - [Sign keys](#sign-keys)
    - [Move your master key to safe place](#move-your-master-key-to-safe-place)
      - [Export keys to external device](#export-keys-to-external-device)
      - [Leave subkeys only](#leave-subkeys-only-sup4perfect-keypair-mastersup)
    - [Revocation Certificate for a key](#revocation-certificate-for-a-key)
    - [Backing up](#backing-up)
  - [Web of trust](#web-of-trust)
    - [verifying keys](#verifying-keys)
    - [Key servers](#key-servers)
  - [References](#references)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

# GPG Quick Guide

This guide shows only one of the setups that is possible with GPG. Actually
you can have different setup with different algorithms and many other variations.

## Not covered
  - How to install gpg and related software.
  - gpg-agent usage and configuration.
  - dirmngr - Key server management.
  - gpgsm - S/MIME mail processing and mail apps.
  - scdaemon - smart card management.
  - GPG usage with different hardwares.

## Prerequisites
  - GPG version 2.
  - Access to shell.

## General recommendations
  - Use encrypted hard drives.
  - Use encrypted USB drive. (If using one)
  - Use HW devices.

## Definitions
  - `Master Key` or `Primary Key` - First key that we generate. 

#### Algorithm List

| cipher              | digest           | compression | AEAD      |
| --                  | --               | --          | --        |
| S0 - NONE           |                  | Z0 - NONE   | A0 - NONE |
| S1 - IDEA           | H1 - MD5         | Z1 - ZIP    | A1 - EAX  |
| S2 - 3DES           | H2 - SHA1        | Z2 - ZLIB   | A2 - OCB  |
| S3 - CAST5          | H3 - RIPEMD160   | Z3 - BZIP2  |           |
| S4 - BLOWFISH (128) | *H4 - reserved.* |             |           |
| *S5 - reserved.*    | *H5 - reserved.* |             |           |
| *S6 - reserved.*    | *H6 - reserved.* |             |           |
| S7 - AES (128)      | *H6 - reserved.* |             |           |
| S8 - AES192         | *H7 - reserved.* |             |           |
| S9 - AES256         | H8 - SHA256      |             |           |
| S10 - TWOFISH (256) | H9 - SHA384      |             |           |
| S11 - CAMELLIA128   | H10 - SHA512     |             |           |
| S12 - CAMELLIA192   | H11 - SHA224     |             |           |
| S13 - CAMELLIA256   |                  |             |           |

#### Key related abbrs

| abbr | Key types     |
| --   | --            |
| sec  | Secret Key    |
| ssb  | Secret Subkey |
| pub  | Public Key    |
| sub  | Public Subkey |

| flag | Capability         |
| --   | --                 |
| S    | Signing key        |
| C    | Certifying key     |
| E    | Encryption key     |
| A    | Authentication key |

*e.g. `SC` is Signing and Certifying key.*

## Getting ready for key generation

Before generating keys, we need to ensure we have things setup correctly.
We need to make sure GPG uses right preferences for us.

### Configuration File
Things we will be configuring are cipher, digest, compression and general options.  
what will be preferred when communicating with others. Listing and other configs.

You can check list of configurations on [gnupg documentation options][gnupg-configs]

```ini
## Even though we don't want to rely on this, it's still good to have longer format.
keyid-format 0xlong

## Show fingerprint
with-fingerprint

## Charset is used for metadata
charset utf-8

## s2k (str2key) configs used for symmetric encryption
# E.g. keys on your machine
s2k-cipher-algo AES256
s2k-digest-algo SHA512
# Mangle count times (0 = no mangle, 1 = adds a salt, 3 - iterates count times)
s2k-mode 3
# Number of passphrase mangling for symmetric encryption. (used with mode = 3)
# Count range is 1024 - 65011712, not all numbers can be used but it will get round up
# to valid value.
# Higher number will result in slower symmetric encryption.
s2k-count 65011712

# when multiple algorithms are supported by all recipients, choose the strongest one:
# Others will get appended to the list
# When signing to multiple parties common algorithms will be used (Which might weaken security)
personal-digest-preferences SHA512 SHA384 SHA256 SHA224 SHA1
personal-cipher-preferences AES256 AES192 AES
personal-compress-preferences ZLIB BZIP2 ZIP Uncompressed

cert-digest-algo SHA512

## List of preferences
# SHA512 SHA384 SHA256 SHA224 SHA1
# AES256 AES192 AES
# ZLIB BZIP2 ZIP Uncompressed
default-preference-list H10 H9 H8 H11 H2 S9 S8 S7 S3 Z2 Z3 Z1 Z0

## passphrase agent config...

## default keyring setup...

## primary keyring...

## List Trusted keys...
# trust 0xkey

## Key server configs
```

**NOTE: s2k configurations in gpg.conf don't affect key export. GPG-Agent is in charge
of exporting keys, but GPG-agent accepts s2k-count config only, it will still use SHA1
with AES128. By default GPG-agent calibrates s2k-count to take around 100ms.**
<sup>[[1]][gnupg-export-issue]</sup>  
*it should be possible to encrypt exported key manually that will actually
respect gpg.conf options.*

## Testing/sandboxing
Before generating actual keys, you might want to experiment with setups.
You can use GNUPGHOME env variable to different location and generate keys there.

## Create Keypairs
Keypairs can be generated and managed multiple ways.
You can have master keypair for signing and certifying and sub keypair
for encryption. (default setup)

By default, gpg will generate (2018):
  - rsa 2k - master key pair with `signing` and `certifying` capabilities.
  - rsa 2k - sub key for `encryption`

All keypairs have their capabilities which define what these keys are used for (See table above):
  - `certifying` - Signing subkeys and other keys.
  - `signing` - Signing messages.
  - `encryption` - Decryption and Encryption of messages.
  - `authentication` - Authentication.

We will be using master key only for signing and have separate
sub keys for other functionality:
  - Master key - Certifying
  - Subkey 1 - Signing
  - Subkey 2 - Encryption
  - Subkey 3 - Authentication

This makes management easier and gives us more options for security.

**If you don't want to use this structure for generating your keys,
you can skip to usage part.**

*Note: Sub keys are not technically different from master key, they are
like separate key pair, but associated with your main key pair.*

#### Small summary of steps for key generation
  - First we generate our first key - master/primary key.
  - Generate revocation certificate.
  - Add signing sub key.
  - Add encryption sub key.
  - Add authentication sub key.
  - We attach uids to it. (User IDs, e.g. `Nodari Chkuaselidze <nodar.chkuaselidze@gmail.com>`)
  - Double check our keys are correct.
  - Remove primary key from device, move it to safer place.
  - Move rest of keys to other devices.

### Why do we need subkeys <sup>[[2]][debian]</sup>
Primary key helps you keep your identity safe, you need to make sure that Master private key
is extremeley secure, it's mostly offline and you should not need to use it a lot. It should
have only certifying capabilities, where you sign your subkeys and other peoples pubkeys.
That does not occur often, so you should not need to use it everyday and it's not too
inconvinient. If you have subkeys for signing, encryption, decryption and authentication,
you can move more freely, because you can revoke these pubkeys with your master key that is
safe. Also your reputation and Web of trust is bound to your master key and if your
subkey gets compromised, you wont have to rebuild your reputation from scratch.

You only need to use master key only in few cases:
  - When you sign someone else's key or revoke an existing signature.
  - When you add a new UID or mark an existing UID as primary.
  - When you create a new subkey.
  - When you revoke an existing UID or subkey.
  - When you change preferences on a UID.
  - When you change the expiration date on your master key or any of its subkeys
  - When you revoke or generate a revocation certificate for the complete key.
(Because all these operations need signatures from master key.)

> Each link of the Web of Trust is an endorsement of the binding between a public key and a user ID.

### Generating Master Key
Our master as well as other subkes

We want to generate our keys with custom configurations, so we can use
`--full-generate-key` with `--expert` to configure everything right away,
instead of modifying it later.

```ini
gpg --full-generate-key --expert
Please select what kind of key you want:
   (1) RSA and RSA (default)
   (2) DSA and Elgamal
   (3) DSA (sign only)
   (4) RSA (sign only)
   (7) DSA (set your own capabilities)
   (8) RSA (set your own capabilities) # Using this.
   (9) ECC and ECC
  (10) ECC (sign only)
  (11) ECC (set your own capabilities)
  (13) Existing key
Your selection? 8

# Then we remove Sign and Encrypt capabilities from master key
Possible actions for a RSA key: Sign Certify Encrypt Authenticate
Current allowed actions: Sign Certify Encrypt

   (S) Toggle the sign capability
   (E) Toggle the encrypt capability
   (A) Toggle the authenticate capability
   (Q) Finished

Your selection? s

Possible actions for a RSA key: Sign Certify Encrypt Authenticate
Current allowed actions: Certify Encrypt

   (S) Toggle the sign capability
   (E) Toggle the encrypt capability
   (A) Toggle the authenticate capability
   (Q) Finished

Your selection? e

# Only Certify capability left.
Possible actions for a RSA key: Sign Certify Encrypt Authenticate
Current allowed actions: Certify

   (S) Toggle the sign capability
   (E) Toggle the encrypt capability
   (A) Toggle the authenticate capability
   (Q) Finished

Your selection? Q

RSA keys may be between 1024 and 4096 bits long.
What keysize do you want? (2048) 4096
Requested keysize is 4096 bits
Please specify how long the key should be valid.
         0 = key does not expire
      <n>  = key expires in n days
      <n>w = key expires in n weeks
      <n>m = key expires in n months
      <n>y = key expires in n years
# See expiration date notes
Key is valid for? (0) 3y
Key expires at Sun 02 Jan 2022 02:09:23 PM +04
Is this correct? (y/N) y

# GPG will ask for passphrase to use for storing our key securely on our machine
# That will be necessary whenever we want to unlock private keys (e.g. signing, modifying keys,
# etc)
GnuPG needs to construct a user ID to identify your key.

Real name: Nodari Chkuaselidze
Email address: node@example.com
Comment:
You selected this USER-ID:
    "Nodari Chkuaselidze <node@example.com>"

Change (N)ame, (C)omment, (E)mail or (O)kay/(Q)uit? O
We need to generate a lot of random bytes. It is a good idea to perform
some other action (type on the keyboard, move the mouse, utilize the
disks) during the prime generation; this gives the random number
generator a better chance to gain enough entropy.
gpg: key 0xBD5A840D12D4CAF1 marked as ultimately trusted
gpg: revocation certificate stored as '/openpgp-revocs.d/[fingerprint].rev'
public and secret key created and signed.

pub   rsa4096/0xBD5A840D12D4CAF1 2019-01-03 [C] [expires: 2022-01-02]
      Key fingerprint = 4091 594A 49DE 8610 E6BC  B937 BD5A 840D 12D4 CAF1
uid                              Nodari Chkuaselidze <node@example.com>
```

With this we:
  - Generated 4k RSA primary key.
  - Set expiration date for the key.
  - Added main UID.
  - Generated default revocation certificate. Check
[revocation](#revocation-certificate) for more information.

### Add User IDs and subkeys
Our first user id was created when we were creating master key, and we only gave
Certifying capabilities to our master key, so we need additional subkeys to do anything
useful (e.g. sign, encrypting).
  We can add multiple keys and user ids in one go, what we need is:
  ```sh
  gpg --expert --edit-key [fingerprint]
  ```

We will end up in interactive menu for working with the key.
Order for adding subkeys and uids does not matter.

#### Add user ids
There's no restriction or verification
of user ids, but you should always [verify](#verify-keys) those user
ids yourself when you are importing someone elses keys.

Adding user ids is as simple as:
```sh
gpg> adduid
Real name: Nodari Chkuaselidze
Email address: node@example.org ## Different from previous one
Comment:
You selected this USER-ID:
    "Nodari Chkuaselidze <node@example.org>"

Change (N)ame, (C)omment, (E)mail or (O)kay/(Q)uit? O

sec  rsa4096/0xCC2FBB062E5CC5AF
     created: 2019-01-03  expires: 2022-01-02  usage: C
     trust: ultimate      validity: ultimate
[ultimate] (1)  Nodari Chkuaselidze <node@example.com>
# Unknown will change to ultimate once you save.
[ unknown] (2). Nodari Chkuaselidze <node@example.org>
```

You can add other uids as well, if you want to associate them with this key.

We also want to verify preferences set for user match to our desired preferences,
you can use `showpref` to verify

```sh
[ultimate] (1). Nodari Chkuaselidze <node@example.org>
     Cipher: AES256, AES192, AES, 3DES
     Digest: SHA512, SHA384, SHA256, SHA224, SHA1
     Compression: ZLIB, BZIP2, ZIP, Uncompressed
     Features: MDC, Keyserver no-modify
[ultimate] (2)  Nodari Chkuaselidze <node@example.com>
     Cipher: AES256, AES192, AES, 3DES
     Digest: SHA512, SHA384, SHA256, SHA224, SHA1
     Compression: ZLIB, BZIP2, ZIP, Uncompressed
     Features: MDC, Keyserver no-modify
```

*Note: 3DES will get automatically appended to your set preferences.*
If you did not configure your preferences in gnupg config file, you can use
`setpref` to set them.

#### Add Subkeys
We are going to create separate RSA 4k keys, one for signing and another for encryption,
using same interactive menu, let's generate signing key first:

```sh
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
Your selection? 8

Possible actions for a RSA key: Sign Encrypt Authenticate
Current allowed actions: Sign Encrypt

   (S) Toggle the sign capability
   (E) Toggle the encrypt capability
   (A) Toggle the authenticate capability
   (Q) Finished

Your selection? e

Possible actions for a RSA key: Sign Encrypt Authenticate
Current allowed actions: Sign # We make sure we only have sign as allowed action.

   (S) Toggle the sign capability
   (E) Toggle the encrypt capability
   (A) Toggle the authenticate capability
   (Q) Finished

Your selection? q
RSA keys may be between 1024 and 4096 bits long.
What keysize do you want? (2048) 4096
Requested keysize is 4096 bits
Please specify how long the key should be valid.
         0 = key does not expire
      <n>  = key expires in n days
      <n>w = key expires in n weeks
      <n>m = key expires in n months
      <n>y = key expires in n years
Key is valid for? (0) 2y
Key expires at Sat 02 Jan 2021 06:53:54 PM +04
Is this correct? (y/N) y
Really create? (y/N) y
We need to generate a lot of random bytes. It is a good idea to perform
some other action (type on the keyboard, move the mouse, utilize the
disks) during the prime generation; this gives the random number
generator a better chance to gain enough entropy.

sec  rsa4096/0xCC2FBB062E5CC5AF
     created: 2019-01-03  expires: 2022-01-02  usage: C
     trust: ultimate      validity: ultimate
ssb  rsa4096/0xFC39DAABD3F30C32
     created: 2019-01-03  expires: 2021-01-02  usage: S
[ultimate] (1). Nodari Chkuaselidze <node@example.org>
[ultimate] (2)  Nodari Chkuaselidze <node@example.com>
```

Then generate key for encryption.

#### Wrapping up
After you are done generating your keys, you can double check and see if everything
is setup as you wanted:

```sh
gpg> list

sec  rsa4096/0xCC2FBB062E5CC5AF
     created: 2019-01-03  expires: 2022-01-02  usage: C
     trust: ultimate      validity: ultimate
ssb  rsa4096/0xFC39DAABD3F30C32
     created: 2019-01-03  expires: 2021-01-02  usage: S
ssb  rsa4096/0xA8DCCC48541D7A18
     created: 2019-01-03  expires: 2021-01-02  usage: E
[ultimate] (1). Nodari Chkuaselidze <node@example.org>
[ultimate] (2)  Nodari Chkuaselidze <node@example.com>
```

Here, you can see, I have:
  1. Master key - rsa4k - with Certifying capability that expires in 3 years
  2. Sub key - rsa4k - with Signing capability that expires in 2 years
  3. Sub key - rsa4k - with Encryption capability that expires in 2 years
  4. 2 User IDs associated: `node@example.org` and `node@example.com`

After we have done all the changes and we are happy with them, we need to save:
```sh
gpg> save
```
And updates will be written to the disk.

#### Sign keys
We also want to make sure we have `self-signed` our user ids, so we don't get
DOSed<sup>[[3]][always-sign]</sup>, so we can only verify Fingerprint of the keys
and be assured ids have not been substituted.
*Note: This does not make keys more valid as attacker can generate everything
from scratch and self-sign it's keys, You still need to [verify](#verify)
your key correctly.*

You can check signatures on key using `gpg --checks-sigs [fingerprint]`,
e.g.
```
$ gpg --check-sigs BD61499B95571C1610C255D1CC2FBB062E5CC5AF   
pub   rsa4096/0xCC2FBB062E5CC5AF 2019-01-03 [C] [expires: 2022-01-02]
      Key fingerprint = BD61 499B 9557 1C16 10C2  55D1 CC2F BB06 2E5C C5AF
uid                   [ultimate] Nodari Chkuaselidze <node@example.org>
sig!3        0xCC2FBB062E5CC5AF 2019-01-03  Nodari Chkuaselidze <node@example.org>
uid                   [ultimate] Nodari Chkuaselidze <node@example.com>
sig!3        0xCC2FBB062E5CC5AF 2019-01-03  Nodari Chkuaselidze <node@example.org>
sub   rsa4096/0xFC39DAABD3F30C32 2019-01-03 [S] [expires: 2021-01-02]
sig!         0xCC2FBB062E5CC5AF 2019-01-03  Nodari Chkuaselidze <node@example.org>
sub   rsa4096/0xA8DCCC48541D7A18 2019-01-03 [E] [expires: 2021-01-02]
sig!         0xCC2FBB062E5CC5AF 2019-01-03  Nodari Chkuaselidze <node@example.org>

gpg: 4 good signatures
```

Exclamation marks following sig mean that signature check passed. Also
you can see that UIDs have been signed as well.

### Move your master key to safe place
After we have finished all key generation, we can store our master key to someplace safe.
And remove it from our device.

#### Export keys to external device
You can have your master key on airgapped computer or export it
to secure place and only load it using strict environments, e.g. No internet liveusb and etc.
To export keys are many options you can try  
e.g.
  - `gpg --export-secret-keys --armor ID > /secure/device/path`
In case you forget passphrase you should also back up revocation certificate,
so you can revoke them later.

#### Leave subkeys only <sup>[[4]][perfect-keypair-master]</sup>
  - Export secret subkeys only (they will be encrypted by default),
but you can make sure they are only stored in ram using ramfs.
  - `gpg --export-secret-subkeys uid or fingerprint > /tmp/gpg/key.asc` (be it normal or ramfs dir)
  - `gpg --delete-secret-key uid or fingerprint` - get rid of private keys
  - `gpg --import /tmp/gpg/key.asc` - import subkey private keys.


### Revocation Certificate for a key
When we generate master key revocation certificate will be generated in
gnupg directory under revocations directory with Fingerprint as its name.
You can move that file and move it to someplace safe. 
If it was not generated for you, than you *must* generate
revocation certs so you can revoke your primary key if it was lost
or compromised and back it up.

### Backing up
You might want to backup certificate and master key.
You can use encrypted external devices that are not used for anything else,
Or print them on paper and store somewhere safe.
*Note: It will be error-prone to type that key back in.*

## Web of trust
### verifying keys
Your PGP key can be used to prove your identity, but key itself no
matter what you put in meta data (Name, email, comment) does not really provide
much information as anyone can use same information and create another key pair.
If you want to make sure that Key belongs to someone you need to personally
verify that key is owned by some person. Even though `0xlong` will provide 64bit
hex, that is easy to generate and collide. So when verifying keys it **highly** recommended to
check keys using fingerprint. (*Note: KeyID is last part of the Fingerprint, either
64bits or 32bits*)
After you have verified other persons key you can assign trust level, depending
how much you trust this person to take care of keys or how much you have verified it.
Then sign the key using your certifying key and update key servers.
If someone trusts you enough, they can trust your signature and that you have verifyied other
person, creating web of trust.

### Key servers
Key servers provide a way to share and update keys. Even though this does not provide
additional security, it's necessary to have in order to easily get updates on key states.
There are couple of key servers out there and they sync information with each other.

There are several things to keep in mind when using key servers:
  - When fetching keys from public key servers (`gpg --search-keys names`)
you can't trust that key belongs to the person it is saying it belongs to, because
anyone can upload key with any meta data. You need to verify that key personally,
via web of trust, or in worst case using some publicly published places.
  - Always update information on key servers, whenever you change something.
(`gpg --send-keys keyIds`)
    - Note: You can not delete keys from key servers, when you publish/update certain keys
it will get merged with existing information, so you need to always revoke keys that are
either compromised or lost. (not using anymore)
  - Before sending encrypted information to someone else or verifying signatures 
  fetch updates to those keys, so you can get revocation information in time. (`gpg --refresh-keys`)

## References:
*Note: some of these links use `gpg v1` and flags, outputs or key choices might not match.*

  1. [Allow s2k options for gpg --export-secret-key][gnupg-export-issue]
  2. [Subkeys - Debian Wiki][debian]
  3. [Always Sign Your PGP Public Key][always-sign]
  4. [Transforming your master key into your laptop keypair][perfect-keypair]

  - [GnuPG][gnupg]
  - [GnuPG docs][gnupg-docs]
  - [GnuPG configuration options][gnupg-configs]
  - [The GNU Privacy Handbook][gpg-privacy-handbook]
  - [GnuPG FAQ][gpg-faq]
  - [GnuPG Message Format - RFC 4880][gpg-message-format-rfc]
  - [GnuPG Key Management][gpg-key-management]
  - [GPG Tutorial][futureboy]
  - [Generating the perfect gpg keypair][perfect-keypair]

[gnupg]: https://www.gnupg.org/
[gnupg-docs]:  https://www.gnupg.org/documentation/
[gentoo-hack]: https://archives.gentoo.org/gentoo-announce/message/dc23d48d2258e1ed91599a8091167002
[gnupg-configs]: https://www.gnupg.org/documentation/manuals/gnupg/GPG-Configuration-Options.html
[debian]: https://wiki.debian.org/Subkeys?action=show&redirect=subkeys
[gpg-message-format-rfc]: https://tools.ietf.org/html/rfc4880
[gpg-faq]: https://www.gnupg.org/faq/gnupg-faq.html
[futureboy]: https://futureboy.us/pgp.html#GoodEncryptionPractices
[gnupg-export-issue]: https://dev.gnupg.org/T1800
[gpg-key-management]: https://www.gnupg.org/gph/en/manual/c235.html
[always-sign]: http://www.heureka.clara.net/sunrise/pgpsign.htm
[perfect-keypair]: https://alexcabal.com/creating-the-perfect-gpg-keypair
[perfect-keypair-master]: https://alexcabal.com/creating-the-perfect-gpg-keypair#transforming-your-master-keypair-into-your-laptop-keypair
[gpg-privacy-handbook]: https://www.gnupg.org/gph/en/manual.html
