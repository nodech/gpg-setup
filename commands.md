# GPG Commands

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->

- [Useful commands](#useful-commands)
  - [Inspect packets](#inspect-packets)
  - [Restart gpg-agent](#restart-gpg-agent)
- [Keys](#keys)
  - [Create Master Keypair](#create-master-keypair)
  - [Listing keys](#listing-keys)
  - [Edit Keys](#edit-keys)
  - [Sign Key](#sign-key)
  - [Check Signatures](#check-signatures)
  - [Default Cert Level](#default-cert-level)
  - [Delete keys](#delete-keys)
- [Data](#data)
  - [Sign](#sign)
    - [ClearSign vs sign](#clearsign-vs-sign)
  - [Verify signature](#verify-signature)
  - [Encrypt/Decrypt data](#encryptdecrypt-data)
    - [Symmetric Encryption](#symmetric-encryption)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

Example commands for gpg. For more information you can check GPG `man` page.

## Useful commands

### Inspect packets
  `pgpdump` can be used to inspect information from encrypted/signed packets.
  - e.g. `pgpdump encrypted.pgp`

### Restart gpg-agent
GPG-Agent can store passphrases in memory.
  - `gpg-connect-agent reloadagent /bye`
  - `gpgconf --kill gpg-agent`

## Keys
### Create Master Keypair 
Generate Master keypair
```sh
gpg --full-generate-key --expert
```

- `--full-generate-key` - full featured key pair generation
- `--expert` - will let us choose custom masterkey with custom capabilities.

### Listing keys

- `gpg --list-keys` or `gpg -k` - list keys
- `gpg --list-secret-keys` or `gpg -K` - list keys with secret key

Both accept k

### Edit Keys
Edit any keys that you have locally imported.
Will get us in interactive command shell.

```sh
gpg --edit-key --expert keyID
```

If you have multiple private keys, first one will get loaded. If you want to
do any modifications that needs your master key (e.g. signing) you will need
to specify `-u` or `--local-user`


- `--edit-key` - sign or edit a key
- `--expert` - will let us choose custom capabilities (useful when modifying our keypair)
- `-u, --local-user USER-ID` use USER-ID to sign or decrypt

Docs will use `gpg> ` prefix for edit-mode commands.

### Sign Key
  - `gpg --sign-key` - Sign public key
  - `gpg --lsign-key` - Sign public key locally

For more details signing you need to get into edit-mode
  - `gpg> sign` - same as --sign-key.
  - `gpg> lsign` - same as --lsign-key.
  - `gpg> nrsign` - sign with nonRevocable signature.


### Check Signatures

```sh
gpg --check-signatures
gpg --check-sigs
gpg --check-sigs node@example.com
gpg --check-sigs fingerprint
```

Output:
```
pub   rsa4096/0xCC2FBB062E5CC5AF 2019-01-03 [C] [expires: 2022-01-02]
      Key fingerprint = BD61 499B 9557 1C16 10C2  55D1 CC2F BB06 2E5C C5AF
uid                   [ultimate] Nodari Chkuaselidze <node@example.org>
sig!3        0xCC2FBB062E5CC5AF 2019-01-03  Nodari Chkuaselidze <node@example.org>
uid                   [ultimate] Nodari Chkuaselidze <node@example.com>
sig!3        0xCC2FBB062E5CC5AF 2019-01-03  Nodari Chkuaselidze <node@example.org>
sig!2  R     0xECB492A551CCB7F2 2019-01-11  Someone Else <node2@example.com>
sub   rsa4096/0xFC39DAABD3F30C32 2019-01-03 [S] [expires: 2021-01-02]
sig!         0xCC2FBB062E5CC5AF 2019-01-03  Nodari Chkuaselidze <node@example.org>
sub   rsa4096/0xA8DCCC48541D7A18 2019-01-03 [E] [expires: 2021-01-02]
sig!         0xCC2FBB062E5CC5AF 2019-01-03  Nodari Chkuaselidze <node@example.org>
```

This command has the same effect as using `gpg --list-keys --with-sig-check`.
The status of the verification is indicated by a flag directly following the "sig" tag.

 - `sig!` - signature has been successfully verified.
 - `sig-` - bad signature
 - `sig%` - error when checking signature (e.g. a non supported algorithm)

Numbers after `sig`:
 - `sig!` number that follows from 0-3 means certification level you have assigned.
 - `sig!3` - is the signature you fully trust (See [Default Cert Level](#default-cert-level))
 - `sig!2` - See [Default Cert Level](#default-cert-level)

Flags after `sig`:
  - `L` - local signature (non-exportable) (signed using `-lsign-key`)
  - `R` - nonRevocable signature (signed using `--nrsign`)
  - `X` - expired signature
  - `P` - URL of a document that describes the policy under which the signature was issued.
  - `N` - contains a notation (certificate notation)


### Default Cert Level

```man
 --default-cert-level n
      The default to use for the check level when signing a key.

      0 means you make no particular claim as to how carefully you verified the key.

      1 means you believe the key is owned by the person who claims to  own  it  but  you
      could not, or did not verify the key at all. This is useful for a "persona" verifi‐
      cation, where you sign the key of a pseudonymous user.

      2 means you did casual verification of the key. For example, this could  mean  that
      you verified the key fingerprint and checked the user ID on the key against a photo
      ID.

      3 means you did extensive verification of the key. For  example,  this  could  mean
      that you verified the key fingerprint with the owner of the key in person, and that
      you checked, by means of a hard to forge document with a photo ID (such as a  pass‐
      port)  that  the  name of the key owner matches the name in the user ID on the key,
      and finally that you verified (by exchange of email) that the email address on  the
      key belongs to the key owner.

      Note  that  the examples given above for levels 2 and 3 are just that: examples. In
      the end, it is up to you to decide just what "casual" and "extensive" mean to you.

      This option defaults to 0 (no particular claim).
```

### Delete keys
  - `gpg --delete-keys` will remove public keys (will fail if there's private key in keyring)
  - `gpg --delete-secret-keys` - remove secret keys only
  - `gpg --delete-secret-and-public-keys` - remove secret and public keys

## Data
  - `--armor, -a` - will return ASCII armored output, useful if you want to send it using
email or some text data.

### Sign
  - `output | gpg --sign` -- Sign message from stdin and print to stdout. (binary, so probably pipe
to file)
  - `output | gpg --armor --sign` -- Sign message from stdin and print ASCII armored signed message
to stdout.
  - `gpg --sign filename` - Sign file and create filename.gpg
  - `gpg --armor --sign filename` - Sign file and create ASCII armored signed message `filename.asc`
  - `output | gpg --clearsign` - Will leave input text as clear text (useful if your input is ascii
or utf8)
  - `gpg --clear-sign filename` - creates filename.asc, will input as clear text
  - `gpg --detach-sign` - creates separate signature file `tt.sig`
  - `gpg --detach-sign --armor` - creates detached ascii armored signature

#### ClearSign vs sign
You can use `sign` if you are signing non-printable(binary) data, you can use `clearsign` for
text data.

### Verify signature
Note: GPG Logs appear in stderr.

Verify attached signature:
  - `gpg --verify signed.gpg`
  - `gpg --verify signed.asc`

Verify Detached signature:
  - `gpg --verify filename.sig` - if filename is in the same directory
  - `gpg --verify filename1.sig originalfile` - verify them separately

Decode attached signature:
  - `gpg --decrypt signed.sig > decoded`
  - `gpg --decrypt signed.asc > decoded`

### Encrypt/Decrypt data
Encrypt:
  - `gpg --encrypt -r recipient@example.com data` - encrypt to one recipient.
  - `gpg --encrypt -r recipient@example.com -r recipient2@example.com data` - encrypt to multiple
recipient.
  - `gpg --armor --encrypt -r recipient@example.com data` - encrypt to one recipient and armor it.
  - `gpg --encrypt --sign -r recipient@example.com data` - Sign and encrypt (You NEED to sign if
you want to prove that its coming from you)
  - `gpg --encrypt --sign --hidden-recipient recipient@example.com data` - Don't reveal recipient in
the message. Receiver will have to iterate over it's private encryption keys to decode.

Decrypt:
  - `gpg --decrypt data.gpg > decrypted`

#### Symmetric Encryption
By default AES128 + Sha1 will be used, or configs defined in gpg.conf.
  - Encrypt: `gpg --symmetric --cipher-algo AES256 file`
  - Encrypt (default or gpg.conf): `gpg --symmetric file`
  - Encrypt and sign: `gpg --symetric --sign --cipher-algo AES256 filename``
  - Decrypt: `gpg --decrypt file.asc`
