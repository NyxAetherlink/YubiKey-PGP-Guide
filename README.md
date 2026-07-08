# YubiKey PGP Setup Guide

A comprehensive, step-by-step guide to generating PGP keys, storing them on a
YubiKey, using them with Kleopatra for daily encrypted/signed communication,
and unlocking a KeePass database with your YubiKey.

---

## Table of Contents

- [Overview](#overview)
- [Prerequisites](#prerequisites)
- [Part 1: Install the Software](#part-1-install-the-software)
- [Part 2: Generate a PGP Keypair](#part-2-generate-a-pgp-keypair)
- [Part 3: Prepare the YubiKey](#part-3-prepare-the-yubikey)
- [Part 4: Move Subkeys to the YubiKey](#part-4-move-subkeys-to-the-yubikey)
- [Part 5: Configure Kleopatra](#part-5-configure-kleopatra)
- [Part 6: Daily Usage with Kleopatra](#part-6-daily-usage-with-kleopatra)
- [Part 7: YubiKey + KeePass Unlock](#part-7-yubikey--keepass-unlock)
- [Part 8: Backup & Recovery](#part-8-backup--recovery)
- [ykman Command Reference](#ykman-command-reference)
- [Troubleshooting](#troubleshooting)
- [Security Notes](#security-notes)

---

## Overview

A YubiKey is a hardware security key that stores cryptographic keys in a
tamper-resistant chip. By storing your PGP private keys on the YubiKey:

- **Keys never leave the device** — signing and decryption happen *on* the
  YubiKey, not in your computer's memory.
- **Physical possession is required** — if your laptop is stolen, the attacker
  still doesn't have your private keys.
- **Portable identity** — plug the YubiKey into any machine, and your PGP
  identity comes with you.
- **Touch policy** — you can require a physical tap on the YubiKey before it
  performs any crypto operation.

This guide covers the **offline/master-key pattern**: your primary (Certify)
key stays safely offline in a backup, while three subkeys (Sign, Encrypt,
Authenticate) live on the YubiKey.

### How PGP + YubiKey Work Together

```
  +---------------------------+
  |   Master Key (Certify)    |  ← Stored offline, backed up to USB/paper
  |   - Only used to certify  |
  |     subkeys, never daily  |
  +---------------------------+
          |
          | signs
          v
  +---------------------------+     +-----------+
  |  Signing Subkey (S)       |---->| YubiKey   |
  |  Encryption Subkey (E)    |---->| OpenPGP   |
  |  Authentication Subkey (A)|---->| Applet    |
  +---------------------------+     +-----------+
                                      PIN + Touch
```

The subkeys are *moved* (not copied) to the YubiKey. After the move, your
computer only holds **stubs** — pointers that say "this key lives on the
smartcard." The actual key material is inside the YubiKey's secure element
and cannot be extracted.

---

## Prerequisites

- **YubiKey**: A YubiKey 4 or 5 series (5 series recommended). Older NEO
  models work but are limited to RSA 2048.
- **Two blank YubiKey slots**: If your YubiKey already has OTP or other
  credential slots configured, the OpenPGP applet has its own dedicated
  storage — it does not conflict with OTP/WebAuthn/FIDO2 slots.
- **Fedora / RHEL-based Linux** (this guide targets Fedora; package names
  differ on Debian/Arch — see the package-name equivalents below).
- **Internet connection** for installing packages.

---

## Part 1: Install the Software

### 1.1 Install GPG, the Smartcard Daemon, and Kleopatra

```bash
sudo dnf install gpg gpgme pinentry-gtk scdaemon pcsc-lite pcsc-lite-ccid \
  pcsc-tools opensc kleopatra
```

| Package            | Purpose                                        |
|-------------------|------------------------------------------------|
| `gpg`             | GNU Privacy Guard — the PGP engine             |
| `gpgme`           | GPG library Kleopatra talks to                 |
| `pinentry-gtk`    | GUI popup for PIN entry (Kleopatra needs this) |
| `scdaemon`        | GPG's smartcard daemon — talks to the YubiKey  |
| `pcsc-lite`       | PC/SC smartcard service (system-level)         |
| `pcsc-lite-ccid`  | USB CCID driver for smartcard readers          |
| `pcsc-tools`      | Diagnostic tools (`pcsc_scan`)                 |
| `opensc`          | OpenSC smartcard framework (backup driver)     |
| `kleopatra`       | GUI certificate manager for GPG                |

**Debian / Ubuntu equivalents:**
```bash
sudo apt install gpg gpgme pinentry-gtk2 scdaemon pcscd pcsc-tools opensc kleopatra
```

**Arch Linux equivalents:**
```bash
sudo pacman -S gnupg pinentry pcsc-lite ccid opensc kleopatra
```

### 1.2 Enable and Start the Smartcard Service

```bash
sudo systemctl enable pcscd
sudo systemctl start pcscd
```

### 1.3 Install ykman (YubiKey Manager CLI)

```bash
sudo dnf install yubikey-manager
```

Or install via pip for the latest version:

```bash
pip install yubikey-manager
```

Verify it works:

```bash
ykman info
```

You should see your YubiKey model, firmware version, and enabled interfaces.

### 1.4 Verify GPG sees the YubiKey

```bash
gpg --card-status
```

You should see output like:

```
Reader ...........: Yubico YubiKey OTP FIDO CCID 00 00
Application ID ...: D2760001240102010006XXXXXX0000
Version ..........: 3.5
Manufacturer .....: Yubico
Serial number ....: XXXXXXXX
...
```

If you get `No card found` or `error: No such device`, jump to the
[Troubleshooting](#troubleshooting) section at the end of this guide.

> **What's happening under the hood?** `pcscd` (PC/SC daemon) is the system
> service that talks to smartcard readers over USB. When you plug in a
> YubiKey, the kernel sees it as a CCID (Chip/Smart Card Interface Device)
> and hands it to pcscd. GPG's `scdaemon` then connects to pcscd to read
> and write the OpenPGP applet data. `gpg --card-status` is the handshake
> that confirms the whole chain is working.

---

## Part 2: Generate a PGP Keypair

This is the most critical step. We'll create a **primary key** (Certify only,
RSA 4096) and three **subkeys** (Sign, Encrypt, Authenticate, RSA 4096 each).

> **Why "Certify only"?** The primary key's only job is to sign other keys
> (certify). It never touches day-to-day encryption or signing. This lets you
> keep the master key offline in secure storage. If a subkey gets compromised,
> you revoke it and issue a new one with your offline master key. If the master
> key were on the YubiKey and compromised, your entire identity is toast.

### 2.1 Generate the Primary Key

Run:

```bash
gpg --full-generate-key
```

You'll be prompted:

1. **Kind of key**: Select `(8) RSA (sign only)` for the primary key.
   This key will *only* certify — it won't encrypt or sign data itself.
   *(In older GPG versions, select `(1) RSA and RSA` and switch the subkey
   type later; the modern approach is to use `--expert` mode.)*

   Actually, let's use expert mode from the start to have full control:

   ```bash
   gpg --expert --full-generate-key
   ```

2. **Key type**: Choose `(8) RSA (set your own capabilities)`.
   - Toggle `Sign` OFF (we don't want the primary to sign directly).
   - Toggle `Encrypt` OFF.
   - Leave `Certify` ON (it should be the only capability).
   - Press `Q` then Enter to confirm.

3. **Key size**: `4096` bits.

4. **Expiration**: Pick something reasonable — `2y` (2 years) is a good
   balance. You can extend expiration later (even past the original expiry
   date) as long as you still have the master key.

   > **Why not no expiry?** An expiration date is a safety net. If you lose
   > access to your key, expired keys stop being trusted automatically. And
   > expiration can always be pushed forward with the master key — it's not a
   > hard deadline.

5. **Real name**: Your name or pseudonym (e.g., `Johnny`).

6. **Email address**: The email you'll associate with this key.

7. **Comment**: Optional — e.g., `[YubiKey PGP]`.

8. **Passphrase**: Choose a **strong** passphrase (12+ characters, mixed
   case, symbols). This protects your master key backup. Write it down and
   store it somewhere safe separate from the key backup.

Once complete, GPG displays your new key fingerprint. **Write this down** —
you'll use it to reference the key.

```
pub   rsa4096/0xABCDEF1234567890 2026-06-17 [SC]
      Key fingerprint = A1B2 C3D4 E5F6 7890 1234  5678 9ABC DEF1 2345 6789
uid                   [ultimate] Johnny <johnny@example.com>
```

The `Key ID` is the last 16 hex digits of the fingerprint:
`0x9ABCDEF1234567890` (the part after the slash in the `pub` line).

### 2.2 Add Subkeys

Subkeys are separate RSA keypairs signed by your master key. You need three:

- **Sign (S)** — for signing emails, git commits, files.
- **Encrypt (E)** — for decrypting messages sent to you.
- **Authenticate (A)** — for SSH authentication (handy bonus).

Launch the key editor:

```bash
gpg --expert --edit-key 0x9ABCDEF1234567890
```

Replace the key ID with yours.

At the `gpg>` prompt, run the following steps:

#### Add Signing Subkey

```gpg
gpg> addkey
```

Select `(8) RSA (set your own capabilities)`.

- Toggle `Sign` ON (leave Encrypt OFF, Authenticate OFF).
- Press `Q` then Enter.
- Keysize: `4096`.
- Expiration: same as master key (e.g., `2y`).
- Enter your master key passphrase when prompted.

You'll see output like:

```
sub  rsa4096/0xSUBSIGNKEY12345  created 2026-06-17 [S]
```

#### Add Encryption Subkey

```gpg
gpg> addkey
```

Select `(6) RSA (encrypt only)`.

- Keysize: `4096`.
- Expiration: `2y`.

```
sub  rsa4096/0xSUBENCRYPTKEY678  created 2026-06-17 [E]
```

#### Add Authentication Subkey

```gpg
gpg> addkey
```

Select `(8) RSA (set your own capabilities)`.

- Toggle `Sign` OFF.
- Toggle `Encrypt` OFF.
- Toggle `Authenticate` ON.
- Press `Q` then Enter.
- Keysize: `4096`.
- Expiration: `2y`.

```
sub  rsa4096/0xSUBAUTHKEY90123  created 2026-06-17 [A]
```

#### Save and exit

```gpg
gpg> save
```

Your key now looks like this (check with `gpg --list-key 0x...`):

```
pub   rsa4096/0xABCDEF1234567890 2026-06-17 [SC] [expires: 2028-06-17]
uid                   [ultimate] Johnny <johnny@example.com>
sub   rsa4096/0xSUBSIGNKEY12345 2026-06-17 [S] [expires: 2028-06-17]
sub   rsa4096/0xSUBENCRYPTKEY678 2026-06-17 [E] [expires: 2028-06-17]
sub   rsa4096/0xSUBAUTHKEY90123 2026-06-17 [A] [expires: 2028-06-17]
```

### 2.3 Create a Revocation Certificate

A revocation certificate lets you invalidate your key if the master key is
lost or compromised. Create it *now*, before anything goes wrong.

```bash
gpg --output ~/revoke-0xABCDEF1234567890.asc --gen-revoke 0xABCDEF1234567890
```

- Answer the prompts. Reason 1 = key compromised (most common).
- Save this file to **encrypted backup** (USB, printed QR code, etc.).
- Store it in a *different* location from your master key backup.

> **Do not skip this.** Without a revocation certificate, if you lose your
> master key, there is no way to tell the world "this key is no longer mine."
> Anyone who has your old public key will keep trusting it.

### 2.4 Export Your Public Key

```bash
gpg --export --armor 0xABCDEF1234567890 > nyx-pgp-public-key.asc
```

Share this file or upload it to keyservers:

```bash
gpg --send-key 0xABCDEF1234567890
```

Or upload to a specific keyserver:

```bash
gpg --keyserver keys.openpgp.org --send-key 0xABCDEF1234567890
```

---

## Part 3: Prepare the YubiKey

### 3.1 Set the YubiKey PINs

The YubiKey OpenPGP applet has three secrets:

| Secret        | Default Value  | Purpose                                          |
|---------------|---------------|--------------------------------------------------|
| **PIN**       | `123456`      | Unlocks the card for normal use (sign/decrypt)   |
| **PUK**       | `12345678`    | Unblocks a PIN that's been entered wrong 3 times |
| **Admin PIN** | `12345678`    | Required for administrative operations (key import, reset) |

> **⚠️ Warn: change these immediately.** The defaults are public knowledge.

Enter the card editor:

```bash
gpg --edit-card
```

Enter admin mode:

```gpg
gpg/card> admin
Admin commands are allowed
```

Now change the PINs:

```gpg
gpg/card> passwd
```

A menu appears:

```
1 - change PIN
2 - unblock PIN
3 - change Admin PIN
4 - set the Reset Code
Q - quit
```

**Step-by-step:**

1. Type `3` and Enter to change the **Admin PIN** first.
   - Enter the current Admin PIN: `12345678`
   - Enter your new Admin PIN (8–127 characters, numbers only or mixed).
     > Best practice: use a random 16+ character string stored in your
     > password manager.
   - Confirm it.

2. Type `1` and Enter to change the user **PIN**.
   - Enter current PIN: `123456`
   - Enter your new PIN (6–127 characters, numbers only).
   - Confirm it.

3. Type `4` and Enter to set a **Reset Code** (optional but recommended).
   - This lets you reset the PIN without the Admin PIN if you remember
     the Reset Code. Same format as the PIN.

4. Type `Q` to exit the PIN menu.

Write down your PIN and Admin PIN on paper and store them securely. If you
lose both the PIN and Admin PIN, the only way to recover is to factory reset
the YubiKey's OpenPGP applet — which destroys all keys on it.

### 3.2 Set Card Attributes (Optional)

Still in `gpg --edit-card` admin mode, you can set identifying info:

```gpg
gpg/card> login
Your login: johnny@example.com    # URL or email
```

```gpg
gpg/card> name
Cardholder's surname: Johnny
Cardholder's given name:
```

```gpg
gpg/card> lang
Language prefs: en
```

```gpg
gpg/card> salutation
Salutation (M = Mr., F = Ms., or space):
```

These are cosmetic but help identify which card is yours if you have multiple.

### 3.3 Set Key Attributes (Key Size)

By default, the YubiKey OOpenPGP applet uses RSA 2048-bit slots. To use 4096:

```gpg
gpg/card> key-attr
```

You'll be asked three times — once for each slot (SIG, ENC, AUT). For each:

1. Select `(1) RSA`.
2. Select `(1) 4096`.

> **Note:** YubiKey 5 series supports RSA 4096. NEO models max out at 2048.

### 3.4 Set Touch Policy (Highly Recommended)

The touch policy determines whether the YubiKey requires a physical tap before
performing an operation. Options:

| Policy  | Effect                                                 |
|---------|--------------------------------------------------------|
| `Off`   | No touch required — any process with PIN can use key   |
| `On`    | Touch required for *every* operation                   |
| `Fixed` | Touch required and cannot be changed without resetting |
| `Cached`| Touch required once, then same app can reuse for ~15s  |
| `Cached-Fixed` | Touch once, cached for ~15s, and the policy is permanent |

Set this with `ykman`:

```bash
# Require touch for every signing operation
ykman openpgp keys set-touch SIG on

# Require touch for every decryption operation
ykman openpgp keys set-touch ENC on

# Require touch for authentication operations
ykman openpgp keys set-touch AUT on
```

> **Recommendation:** Use `Cached` or `On` for SIG and ENC. The tap gives you
> a physical confirmation that the YubiKey is being used. For AUT (SSH), `Off`
> is reasonable since you'll authenticate many times per session and tapping
> gets old fast.
>
> If you set `Fixed`, you cannot turn touch off without factory-resetting the
> entire OpenPGP applet and re-importing keys.

### 3.5 (Optional) Factory Reset

If the YubiKey has been used before and you want a clean slate:

```bash
ykman openpgp reset
```

**This permanently destroys all keys and data on the OpenPGP applet.**

After reset, unplug and replug the YubiKey, then verify:

```bash
gpg --card-status
```

You should see the factory-default OpenPGP applet.

### 3.6 Save Card Changes and Exit

```gpg
gpg/card> quit
```

---

## Part 4: Move Subkeys to the YubiKey

This is the moment of truth. We transfer the subkeys from your computer into
the YubiKey's secure element. **After this step, the subkeys on your computer
are replaced with stubs** — pointers that reference the YubiKey.

### 4.1 Check Your Key Structure

```bash
gpg --list-secret-keys --keyid-format=long 0xABCDEF1234567890
```

Make sure you have exactly one `sec#` (master key) and three `sub` entries
(S, E, A). The `#` after `sec` means the master key is not present on this
machine — but it should show as `sec` (not `sec#`) since you generated it
here.

### 4.2 Open the Key Editor

```bash
gpg --edit-key 0xABCDEF1234567890
```

You'll see:

```
Secret key is available.

sec  rsa4096/0xABCDEF1234567890
     created: 2026-06-17  expires: 2028-06-17  usage: SC
     trust: ultimate      validity: ultimate
ssb  rsa4096/0xSUBSIGNKEY12345
     created: 2026-06-17  expires: 2028-06-17  usage: S
ssb  rsa4096/0xSUBENCRYPTKEY678
     created: 2026-06-17  expires: 2028-06-17  usage: E
ssb  rsa4096/0xSUBAUTHKEY90123
     created: 2026-06-17  expires: 2028-06-17  usage: A
```

Note: `ssb` means "secret subkey" (the private part is here on disk).

### 4.3 Move the Signing Subkey

```gpg
gpg> key 1
```

This selects the first subkey (the signing key). You should see an asterisk `*`
next to it:

```
ssb* rsa4096/0xSUBSIGNKEY12345
```

Now move it to the YubiKey:

```gpg
gpg> keytocard
```

It asks where to store it:

```
Please select where to store the key:
   (1) Signature key
   (2) Encryption key
   (3) Authentication key
Your selection? 1
```

Choose `1` (Signature key). Enter your master key passphrase when prompted,
then enter your YubiKey **Admin PIN** (not the user PIN).

After success, deselect the subkey:

```gpg
gpg> key 1
```

The asterisk disappears. The signing key is now on the YubiKey.

### 4.4 Move the Encryption Subkey

```gpg
gpg> key 2
gpg> keytocard
```

This time choose `(2) Encryption key`. Enter passphrase + Admin PIN.

```gpg
gpg> key 2
```

### 4.5 Move the Authentication Subkey

```gpg
gpg> key 3
gpg> keytocard
```

Choose `(3) Authentication key`. Enter passphrase + Admin PIN.

```gpg
gpg> key 3
```

### 4.6 Verify and Save

Type `save` to write changes:

```gpg
gpg> save
```

Now check your secret keys:

```bash
gpg --list-secret-keys --keyid-format=long 0xABCDEF1234567890
```

You should see:

```
sec  rsa4096/0xABCDEF1234567890
     created: 2026-06-17  expires: 2028-06-17  usage: SC
     trust: ultimate      validity: ultimate
ssb> rsa4096/0xSUBSIGNKEY12345
     created: 2026-06-17  expires: 2028-06-17  usage: S
     card-no: 0006XXXXXX
ssb> rsa4096/0xSUBENCRYPTKEY678
     created: 2026-06-17  expires: 2028-06-17  usage: E
     card-no: 0006XXXXXX
ssb> rsa4096/0xSUBAUTHKEY90123
     created: 2026-06-17  expires: 2028-06-17  usage: A
     card-no: 0006XXXXXX
```

The `>` after each `ssb` means the private key material is not on disk — it's
on the smartcard (your YubiKey) identified by `card-no`.

### 4.7 Test the YubiKey

Test signing:

```bash
echo "test message" | gpg --clearsign --default-key 0xABCDEF1234567890
```

If touch is enabled, the YubiKey LED blinks. Tap it. You should see a signed
message output.

Test encryption:

```bash
echo "test message" | gpg --encrypt --recipient johnny@example.com | gpg --decrypt
```

You should see your original message after entering your PIN.

---

## Part 5: Configure Kleopatra

Kleopatra is the GUI certificate manager for GPG on KDE and compatible
desktops. When used with a YubiKey, it gives you a visual interface for
encryption, decryption, signing, and certificate management — all backed by
the same GPG infrastructure.

### 5.1 Launch Kleopatra

```bash
kleopatra
```

Or find it in your application menu.

### 5.2 Initial Setup

The first time you open Kleopatra, it may prompt you to set up:

- **Personal OpenPGP key**: If it doesn't detect your key, go to
  **File → New OpenPGP Key Pair**. You can point it to your existing key
  by importing your public key: **File → Import** → select your `.asc` file
  or let it search your GPG keyring (your key should already appear).

  > **Note:** Kleopatra reads from the same `~/.gnupg/` keyring as `gpg` CLI.
  > If `gpg --list-keys` shows your key, Kleopatra will see it too.

### 5.3 Your Key Certificate

Kleopatra lists all keys in its main window. Your key shows up as:

```
┌─────────────────────────────────────────────────────┐
│  Johnny <johnny@example.com>                              │
│  ✅ OpenPGP Key — [SC] [E] [A]                      │
│  Key ID: 0xABCDEF1234567890                         │
│  YubiKey present (card-no: 0006XXXXXX)              │
└─────────────────────────────────────────────────────┘
```

Double-click your key to view details:

- **User IDs**: Your name + email.
- **Subkeys**: The three keys (S, E, A) each showing `card-no: 0006XXXXXX`.
- **Trust level**: Should show "Ultimate" for your own key.

### 5.4 Configure Certificate Expiry Monitoring

Kleopatra can warn you when keys are about to expire.

1. Go to **Settings → Configure Kleopatra...**
2. Under **Certificate Cache**, set:
   - "Days before expiry to show a warning": `30` (sensible default).
   - "Refresh certificates every": `1 day`.

### 5.5 Configure Kleopatra as Default Email Handler (Optional)

This lets you right-click files in Dolphin or other file managers to
encrypt/sign/decrypt.

1. In Kleopatra: **Settings → Configure Kleopatra...**
2. Under **Miscellaneous** → **Components**:
   - Enable "Register Kleopatra as default PGP/MIME handler".
   - Enable "Register file type associations" — this adds right-click
     context menu entries for `.gpg`, `.pgp`, `.asc` files.

> **What this gives you:** In Dolphin, right-click a file and you'll see
> "Sign with OpenPGP", "Encrypt with OpenPGP", "Decrypt with OpenPGP" —
> all backed by your YubiKey.

### 5.6 Import Other People's Public Keys

To encrypt a message for someone, you need their public key.

1. **From a file**: **File → Import** → select the `.asc` file.
2. **From a keyserver**: **File → Lookup on Server...** → enter email or
   key ID → select the key → **Import**.
3. From a text blob: copy the armored key block → **File → Import Clipboard**.
4. Drag-and-drop: drop an `.asc` file directly onto the Kleopatra window.

---

## Part 6: Daily Usage with Kleopatra

Once the YubiKey + Kleopatra are set up, here's how to use them day-to-day.

### 6.1 Encrypting a File

1. Right-click the file in Dolphin → **Encrypt with OpenPGP**.

   *Or in Kleopatra:* **File → Encrypt/Decrypt → Encrypt → Browse** → select file.

2. Select the recipient(s):
   - Double-click the recipient's key from the list.
   - You need their public key imported (see §5.6).

3. Check **"Encrypt for me too"** if you want to be able to decrypt it later.

4. Click **Encrypt**.

Result: a `<filename>.gpg` file is created.

> **Under the hood:** Kleopatra calls `gpg --encrypt` which uses a session
> key (random symmetric key) to encrypt the file, then encrypts that session
> key with the recipient's RSA public key. The result is a binary OpenPGP
> message.

### 6.2 Decrypting a File

1. Right-click a `.gpg` or `.pgp` file → **Decrypt with OpenPGP**.

   *Or in Kleopatra:* **File → Encrypt/Decrypt → Decrypt → Browse** → select file.

2. Enter your **YubiKey PIN** when the pinentry window pops up.

3. If touch is enabled, the YubiKey LED blinks — **tap it**.

4. The decrypted file appears as `<filename>` (without the `.gpg` extension).

> **Under the hood:** Kleopatra asks GPG to decrypt. GPG sees the encryption
> was done to your subkey, looks up the corresponding secret key, and finds
> a stub pointing to the YubiKey. It asks the YubiKey to perform RSA
> decryption. The YubiKey requires your PIN and optionally a touch, then
> returns the decrypted session key to GPG, which decrypts the file. **The
> private key never leaves the YubiKey.**

### 6.3 Signing a File

1. Right-click a file → **Sign with OpenPGP**.

   *Or in Kleopatra:* **File → Sign/Verify → Sign → Browse** → select file.

2. Choose signature type:
   - **Detached signature** (default): creates a separate `.sig` or `.asc`
     file. The original file is unchanged.
   - **Clearsign** (for text): the original text stays readable, with the
     signature wrapped around it.
   - **Combined**: the signature is embedded in the file (binary).

3. Enter your YubiKey PIN, tap the key.

### 6.4 Verifying a Signature

1. Right-click a `.sig` or `.asc` file → **Verify with OpenPGP**.

   *Or in Kleopatra:* **File → Sign/Verify → Verify → Browse** → select the
   signature file (or the signed file).

2. Kleopatra shows:
   - ✅ **Good signature** from `<signer>` — the file is authentic and untampered.
   - ❌ **Bad signature** — the file or signature has been modified.
   - ⚠️ **No known public key** — you don't have the signer's public key.

### 6.5 Sign & Encrypt (Email/File)

This is the canonical way to send authenticated, confidential data.

1. **File → Sign/Encrypt** (there's also a combined option in the wizard).

2. Select the file.

3. **Your signing key** is automatically used (first key that can sign).

4. **Select recipients** for encryption (you need their public key).

5. Enter PIN + touch when prompted.

### 6.6 Encrypted Email with Thunderbird (Bonus)

With the Enigmail successor (Thunderbird 78+ has built-in OpenPGP):

1. In Thunderbird, go to **Account Settings → End-To-End Encryption → OpenPGP**.

2. Click **"Use built-in OpenPGP"** → **"Get My Keys"** → select your key.

3. For composing:
   - Click the **"Lock"** icon in the email composition window to encrypt.
   - Click the **"Pencil"** icon to sign.
   - When you send, Thunderbird prompts pinentry → enter YubiKey PIN.

> **Kleopatra integration:** Thunderbird calls GPG directly from its built-in
> OpenPGP engine. It does not go through Kleopatra itself, but both use the
> same underlying GPG keyring. The YubiKey works with both identically.

---

## Part 7: YubiKey + KeePass Unlock

This section covers using the YubiKey as a second-factor unlock method for
KeePass databases. The YubiKey does **HMAC-SHA1 challenge-response** — the
KeePass application sends a random challenge, and the YubiKey computes an
HMAC-SHA1 response using a secret stored in one of its OTP slots. Since the
secret never leaves the YubiKey, the database cannot be unlocked without it.

> **Important limitation:** This is not the same as storing a PGP key on the
> YubiKey. OTP challenge-response and OpenPGP are separate applets. You can
> use the PGP key *and* the HMAC-SHA1 feature on the same YubiKey
> simultaneously — they don't conflict. The only thing you lose is that OTP
> slot 2 (traditionally used for static password or a second OTP config).

### 7.1 Choose Your KeePass Client

| Client       | YubiKey Support                              | Recommendation         |
|-------------|----------------------------------------------|------------------------|
| KeePassXC   | Native YubiKey challenge-response (built-in) | ✅ **Recommended**     |
| KeePass2    | Requires KeeChallenge plugin                 | Works but more setup   |
| KeePassDX   | Android — supports YubiKey via NFC/USB       | Good for mobile        |

This guide covers **KeePassXC** (Linux desktop, also on Windows/macOS).

### 7.2 Install KeePassXC

```bash
sudo dnf install keepassxc
```

### 7.3 Configure YubiKey Slot 2 for Challenge-Response

The YubiKey has two OTP slots:

| Slot | Default Behavior                    | Our Plan                  |
|------|-------------------------------------|---------------------------|
| 1    | YubiKey OTP (press once → outputs credential) | Leave as-is (or disable if unused) |
| 2    | Secondary OTP (long-press)          | Reprogram for KeePass     |

> **If slot 2 already has an OTP configuration you use:** You cannot use both
> simultaneously. Either pick slot 1 (if you don't use OTP), or use a second
> YubiKey. For this guide we'll use slot 2, which most people don't use.

#### Step 1: Program slot 2

```bash
# Configure slot 2 for HMAC-SHA1 challenge-response with a new random secret
ykman otp chalresp --generate 2
```

You should see:

```
Program a challenge-response credential in slot 2?
  Touch your YubiKey to continue...
```

Touch the YubiKey. After a moment:

```
Configuring YubiKey OTP slot 2... done
```

> **What this does:** It generates a 20-byte random secret and stores it in
> slot 2's HMAC-SHA1 configuration. The secret never leaves the YubiKey —
> `ykman` generated it *on* the device, not in software. This is more secure
> than generating the secret in software and uploading it.

#### Step 2: Verify the configuration

```bash
ykman otp chalresp --generate 2
```

Note: if you run this a second time, it **overwrites** the previous secret.
Only do this if you want to create a new secret (which will invalidate any
database credentials-dependent databases).

To check without regenerating:

```bash
ykman otp info
```

#### Step 3: Alternative — use slot 1 instead of slot 2

If you need slot 2 for something else:

```bash
# Use YubiKey slot 1 for KeePass instead
ykman otp chalresp --generate 1
```

> **Warning:** If you use slot 1, you lose the default YubiKey OTP press-to-type
> functionality on that slot. Most people only use YubiKey OTP for web login,
> and with WebAuthn/FIDO2 being the modern standard, OTP is increasingly
> obsolete. It's safe to repurpose slot 1 if you don't use YubiKey OTP.

### 7.4 Create a New Database with YubiKey Protection

1. **Launch KeePassXC** → **Database → New Database**.

2. **Set a strong master password** (this is your primary credential).
   - Minimum 12 characters, ideally a passphrase.
   - KeePassXC has a built-in password generator — use it.

3. **Add YubiKey as additional protection:**
   - In the **"Master Key"** configuration step, check both:
     - ☑ Master password (your passphrase)
     - ☑ YubiKey Challenge-Response
   - Click the **"YubiKey"** button to detect your key.
   - If it doesn't auto-detect, make sure the YubiKey is plugged in and tap
     the button to scan.

4. **Finish creation** and save the database file (e.g., `my-vault.kdbx`).

> **What's happening behind the scenes:** KeePassXC sends a random challenge
> to the YubiKey, receives the HMAC-SHA1 response, and incorporates it into
> the database key. Both the password AND the YubiKey response are required
> to derive the database encryption key. Without the physical YubiKey (with
> the correct secret in slot 2), the "Password + YubiKey" combination is
> incomplete and the database cannot be decrypted.

### 7.5 Add YubiKey to an Existing Database

If you already have a KeePassXC database without YubiKey protection:

1. **Open your database** in KeePassXC (with your current password).

2. **Database → Database Settings → Security → Master Key**.

3. Currently you should see only "Master password" as a component.

4. Click **"Add additional protection"** → **"YubiKey Challenge-Response"**.

5. Insert your YubiKey (make sure slot 2 is programmed), click **"Detect"**.

6. **Save the database** (Ctrl+S).

7. **Test unlock**: close the database, reopen it. You should be prompted for:
   - Master password
   - YubiKey (plug it in, tap if prompted)

### 7.6 Unlocking Workflow (Daily Use)

1. Plug in your YubiKey.
2. Launch KeePassXC → open your database.
3. Enter your master password.
4. KeePassXC auto-detects the YubiKey — **tap the YubiKey** if it blinks.
5. The database unlocks.

> **No need to configure anything for each unlock.** The program prompts the
> challenge-response credential in slot 2 automatically. As long as the
> YubiKey is plugged in and the secret matches, it works.

### 7.7 Backup Strategy for Database Access

Since the YubiKey is now required to unlock the database, you **must** have
a fallback plan:

1. **Backup the database file** (`.kdbx`) — this is your encrypted vault.
   Keep copies on cloud storage, a USB drive, etc.

2. **Keep a backup YubiKey** (strongly recommended):
   a. Get a second YubiKey (same model or another).
   b. Program slot 2 with the *same* secret:

   ```bash
   ykman otp chalresp --new-secret <base64-or-hex-of-secret> 2
   ```

   Wait — there's a catch. When you use `--generate`, the secret is generated
   *on the device* and never revealed. To use the same secret on two YubiKeys,
   you need to generate it yourself **before** programming either device:

   a. Generate a 40-character hex key:
      ```bash
      openssl rand -hex 20
      ```

   b. Program both YubiKeys with the same hex key:
      ```bash
      ykman otp chalresp 2 <your-hex-key>
      ykman otp chalresp 1 <your-hex-key>   # backup slot on same device
      ```

   > **⚠️ Security:** If you use `--generate`, the secret is created on the
   > YubiKey's hardware and can never be read back. You would need to factory
   > reset and start over with a known hex key if you want a backup. Always
   > use a known hex key from `openssl rand -hex 20` if you plan to back up.
   > Store the hex string in an encrypted password manager, not in a text file.

3. **Emergency paper backup:** Generate a hex key with `openssl rand -hex 20`,
   print it, and store it in a safe deposit box. You can recreate the YubiKey
   slot on any device with `ykman otp chalresp 2 <hex>`.

4. **Password-only fallback:** KeePassXC lets you create an *emergency
   sheet* — a one-time recovery code or a separate key file. Enable this in
   **Database → Database Settings → Security → Master Key → Add emergency
   sheet**. This gives you a printable PDF with a one-time recovery key that
   bypasses the YubiKey requirement.

> **The bottom line:** If you only have one YubiKey and it's lost or broken,
> your database is permanently inaccessible unless you have a backup strategy
> in place. Plan ahead.

---

## Part 8: Backup & Recovery

### 8.1 What to Back Up

| What                    | Why                                    | Storage                    |
|------------------------|----------------------------------------|---------------------------|
| Master key (secret)    | Re-issue subkeys, extend expiry        | Encrypted USB + paper     |
| Subkeys backup (secret)| Restore keys *without* YubiKey         | Encrypted USB (emergency) |
| Public key (export)    | Share with others, re-import anywhere  | Cloud, keyserver, any     |
| Revocation cert        | Invalidate key if compromised          | Encrypted USB + paper     |
| GPG configuration      | Trust settings, preferences            | Config backup             |
| KeePass database       | Your vault contents                    | Cloud + local             |
| YubiKey slot 2 secret  | Reprogram backup YubiKey               | Password manager + paper  |

### 8.2 Back Up the Master Key

```bash
# Export secret master key (encrypted with your passphrase)
gpg --export-secret-keys --armor 0xABCDEF1234567890 > master-key-backup.asc

# Export all subkeys (for emergency restoration)
gpg --export-secret-subkeys --armor 0xABCDEF1234567890 > subkeys-backup.asc

# Export public key (shareable)
gpg --export --armor 0xABCDEF1234567890 > public-key-backup.asc

# Export the revocation certificate
# (You should already have this from Step 2.3)
cp ~/revoke-0xABCDEF1234567890.asc ./revoke-backup.asc
```

**Storage recommendations:**

1. **Encrypted USB drive** — use LUKS:
   ```bash
   sudo cryptsetup luksFormat /dev/sdX
   sudo cryptsetup open /dev/sdX pgp-backup
   sudo mkfs.ext4 /dev/mapper/pgp-backup
   sudo mount /dev/mapper/pgp-backup /mnt
   cp *.asc /mnt/
   sudo umount /mnt
   sudo cryptsetup close pgp-backup
   ```

2. **Paper backup (offline cold storage):**
   Print the armored (text) key, or use `paperkey` to create a more
   compact representation:

   ```bash
   gpg --export-secret-key 0xABCDEF1234567890 | paperkey --output master-key-paper.txt
   ```

   This outputs just the secret data (much smaller than the full armored key).
   Print this and lock it in a safe.

3. **QR code backup**: Encode the armored key as a series of QR codes.

### 8.3 Recovery Scenarios

#### Scenario A: Lost YubiKey (but have backup keys)

1. Buy a new YubiKey.
2. Reset its OpenPGP applet: `ykman openpgp reset`.
3. Import your subkeys from backup:
   ```bash
   gpg --import subkeys-backup.asc
   ```
4. Follow [Part 4](#part-4-move-subkeys-to-the-yubikey) to move the subkeys
   to the new YubiKey.
5. Optionally, set new PINs on the new YubiKey.

#### Scenario B: Lost YubiKey and lost subkeys backup (but have master key backup)

1. Buy a new YubiKey.
2. Reset its OpenPGP applet.
3. Import the master key:
   ```bash
   gpg --import master-key-backup.asc
   ```
4. Generate new subkeys (follow [Part 2.2](#22-add-subkeys)).
5. Move subkeys to the new YubiKey.
6. Export and redistribute your new public key, since the subkey fingerprints
   have changed.

#### Scenario C: Master key compromised

1. **Publish the revocation certificate**:
   ```bash
   gpg --import revoke-backup.asc
   gpg --send-key 0xABCDEF1234567890
   ```
2. Generate a brand-new keypair.
3. Notify your contacts of the new key.

#### Scenario D: All keys and YubiKey lost

1. You still have your KeePass database, right? (And a backup of it?)
2. If you set up the emergency sheet (Part 7.7), use the one-time recovery
   code to open it.
3. Retrieve your backed-up PGP keys from the KeePass vault if you stored
   them there.
4. Otherwise, you've lost your PGP identity. Generate a new key, tell your
   contacts, and update your keyserver entries.

---

## ykman Command Reference

All `ykman` commands for quick reference.

### General

```bash
ykman info                          # Display YubiKey model, firmware, interfaces
ykman list                          # List connected YubiKeys
ykman config usb --enable <interface>  # Enable/disable USB interfaces (FIDO2, OTP, CCID, etc.)
```

### OpenPGP Applet

```bash
ykman openpgp info                  # Show OpenPGP applet state and key info
ykman openpgp reset                 # Factory reset the entire OpenPGP applet
ykman openpgp keys set-touch SIG on # Set touch policy for signing key
ykman openpgp keys set-touch ENC on # Set touch policy for encryption key
ykman openpgp keys set-touch AUT on # Set touch policy for auth key
ykman openpgp keys set-touch SIG cached  # Cached touch (reuse for ~15s)
ykman openpgp keys set-touch SIG off     # Disable touch requirement
ykman openpgp certificates export <key> <cert>  # Export OpenPGP certificate
```

### OTP / Challenge-Response (for KeePass)

```bash
ykman otp info                      # Show configured OTP slots
ykman otp chalresp --generate 1     # Program slot 1 for challenge-response (new random secret)
ykman otp chalresp --generate 2     # Program slot 2 for challenge-response (new random secret)
ykman otp chalresp 2 <hex>              # Program slot 2 with a known secret (backup YubiKey)
ykman otp chalresp --generate 2         # Generate a new random secret on-device (no backup possible)
ykman otp delete 1                  # Clear slot 1 (restore to factory default)
ykman otp delete 2                  # Clear slot 2
```

### Other Useful Commands

```bash
ykman oath info                     # Show OATH applet status
ykman oath accounts list            # List TOTP accounts (if using OATH)
ykman piv info                      # Show PIV applet status
ykman fido info                     # Show FIDO2/WebAuthn info
ykman fido credentials list         # List FIDO2 credentials stored on device
```

---

## Troubleshooting

### "No card detected" / `gpg --card-status` returns nothing

```
gpg: error getting version: No such device
```

1. **Check the PC/SC service:**
   ```bash
   sudo systemctl status pcscd
   # Should show "active (running)"
   ```
   If not running:
   ```bash
   sudo systemctl restart pcscd
   ```

2. **Check if the kernel sees the device:**
   ```bash
   lsusb | grep -i yubico
   # Should show: Yubico.com Yubikey 5/...
   ```

3. **Check PC/SC is talking to it:**
   ```bash
   pcsc_scan
   ```
   Plug/unplug the YubiKey. You should see "Reader: Yubico YubiKey..." appear.

4. **Kill and restart scdaemon:**
   ```bash
   gpgconf --kill scdaemon
   gpg --card-status
   ```

5. **Permissions issue (multiple users):** Ensure you're not running `pcsc_scan`
   as root while trying to use `gpg` as your user. They share the same pcscd
   daemon.

6. **YubiKey not in CCID mode:** The YubiKey needs to have the CCID interface
   enabled. Check with `ykman info`. If CCID is disabled:
   ```bash
   ykman config usb --enable CCID
   ```

### "Inappropriate ioctl for device" on `gpg --card-status`

This is a common issue on Fedora. The smartcard driver is missing or wrong:

```bash
# Install required packages
sudo dnf install pcsc-lite pcsc-lite-ccid pcsc-tools opensc

# Remove the troublesome gnupg-pkcs11-scd if installed (conflicts with pcscd)
sudo dnf remove gnupg-pkcs11-scd

# Restart daemons
sudo systemctl restart pcscd
gpgconf --kill scdaemon
gpg --card-status
```

### "Key not found" when using YubiKey

```
gpg: decryption failed: No secret key
```

1. Your GPG stubs are missing. Re-insert the YubiKey and run:
   ```bash
   gpg --card-status
   ```
   This re-learns the stub references from the card.

2. If still not working, re-import your public key and let GPG detect the
   YubiKey subkeys:
   ```bash
   gpg --import public-key-backup.asc
   gpg --card-status
   ```

### PIN Entered Wrong 3 Times — Card Blocked

The **user PIN** has 3 attempts before it locks. You'll see:

```
gpg: PIN for CHV2 is locked.
```

To unblock:

```bash
gpg --edit-card
gpg/card> admin
gpg/card> passwd
```

Choose option `2` (unblock PIN). You'll need the **PUK** (PIN Unblocking Key).
Enter PUK, then set a new PIN.

If you also lost the PUK (or Admin PIN), the only option is:

```bash
ykman openpgp reset
```

**This destroys all keys on the OpenPGP applet.** You'll need to re-import
from backup.

### PIN Entry Window Doesn't Appear (Kleopatra/KDE)

```bash
# Check which pinentry is being used
update-alternatives --config pinentry
```

Select `pinentry-gtk-2` or `pinentry-qt` on the list. On Fedora:

```bash
sudo alternatives --set pinentry /usr/bin/pinentry-gtk-2
```

Or set it in `~/.gnupg/gpg-agent.conf`:

```
pinentry-program /usr/bin/pinentry-gtk-2
```

Restart the agent:

```bash
gpgconf --kill gpg-agent
```

### KeePassXC: "Failed to find YubiKey"

1. Make sure slot 2 is configured with challenge-response:
   ```bash
   ykman otp info
   ```
   Look for "Challenge-response" on slot 2.

2. Check that KeePassXC has permission to access the YubiKey:
   ```bash
   # On Fedora, you may need to add your user to the 'dialout' group
   sudo usermod -aG dialout $USER
   # Log out and back in for group changes to take effect
   ```

3. Try a different USB port (preferably directly on the motherboard, not a hub).

4. Run KeePassXC from the terminal to see debug output:
   ```bash
   keepassxc 2>&1 | grep -i yubi
   ```

### "Secret subkey not available" After Reboot

This happens when the YubiKey was removed while GPG had cached the key
references. Fix:

```bash
gpg --card-status
```

This refreshes all key stubs from the card. Always run this after plugging
in the YubiKey if GPG can't find the keys.

---

## Security Notes

### Threat Model

This setup protects against:

- ✅ **Laptop theft** — keys are on a separate hardware device.
- ✅ **Malware reading key files** — private key material never enters computer
     memory in usable form.
- ✅ **Remote key extraction** — an attacker must have physical possession of
     the YubiKey AND know your PIN.
- ✅ **Brute force of PIN** — YubiKey locks after 3 wrong PIN attempts and 3
     wrong Admin PIN attempts (separate counters).

This setup does NOT protect against:

- ❌ **Rubber-hose cryptanalysis** (you can be compelled to reveal your PIN).
- ❌ **Malware during signing/decryption** — if your computer is compromised,
     the content you sign or the decrypted data can be read by the attacker.
- ❌ **YubiKey physical theft** — an attacker who steals your YubiKey AND
     obtains your PIN has full access.

### Recommended Practices

1. **Store the master key offline permanently** — ideally on a LUKS-encrypted
   USB drive that stays in a safe unless you need to certify a new subkey.
   The machine you use for master-key operations should be a dedicated,
   air-gapped machine or an ephemeral Tails session.

2. **Never generate keys on a machine with internet access** if you're
   paranoid. Tails OS is excellent for this.

3. **Use different PINs** for user PIN and Admin PIN. They protect different
   operations. The Admin PIN is more sensitive since it can change card
   settings, import keys, and modify the touch policy.

4. **Set touch policies** for the operations you use less frequently. It's
   a minor inconvenience that adds significant protection against unattended
   use.

5. **Multiple YubiKeys** — buy two. Program one as a daily driver, keep the
   other in a safe as backup. If you lose your daily driver, your backup
   YubiKey with the same subkeys keeps you operational while you order a
   replacement.

6. **Revocation cert + expiry** — a key without a revocation certificate and
   an expiration date is a liability. Set both.

7. **Keepass + YubiKey** — the YubiKey is a *second factor*, not a
   replacement for a strong master password. Always use a strong password +
   YubiKey, never YubiKey alone.

### Additional Resources

- [Yubico OpenPGP Guide](https://docs.yubico.com/yesdk/users-manual/application-openpgp/openpgp.html)
- [GPG Smartcard Manual](https://gnupg.org/documentation/manuals/gnupkgs/OpenPGP-Card.html)
- [KeePassXC YubiKey Documentation](https://keepassxc.org/docs/KeePassXC_UserGuide#_using_a_yubikey)
- [Riseup OpenPGP Best Practices](https://riseup.net/en/security/message-security/openpgp/gpg-best-practices)

---

*Generated for personal reference. Verify commands against your specific
YubiKey model and firmware version before executing. YubiKey 5 series
firmware 5.4+ recommended for best compatibility with all features described.*
