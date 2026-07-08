# Behind the Curtain: How PGP, Smartcards, and Challenge-Response Work

A plain-language explanation of the cryptographic machinery you're using.
No equations, just concepts.

---

## 1. Asymmetric Cryptography (Public-Key Cryptography)

Most traditional locks use a *symmetric* system: the same key locks and
unlocks the door. If you want to share a locked box with someone, you have
to somehow get them a copy of the same key — which is the original problem
you were trying to solve.

Public-key cryptography flips this:

- **Public key** — like a mailbox slot. Anyone can stuff a message in.
- **Private key** — like your mailbox key. Only you can open it to read
  the messages.

You can freely give away your public key. People encrypt messages with it,
and only your private key can decrypt them. This solves the "how do I share
the key" problem — you don't.

There's also **signing**, which is encryption in reverse: you encrypt a
fingerprint of the message with your *private* key, and anyone with your
*public* key can verify that you were the one who signed it. Like a wax seal
that's uniquely yours and cryptographically verifiable.

### Why Keys Come in Pairs

```
       Alice's Public Key          Alice's Private Key
       (shared with everyone)      (known only to Alice)
               |                           |
  Message ────>|  Encrypt  ────> ciphertext ────>|  Decrypt  ────> Message
                                               (only Alice can do this)
```

And for signing:

```
  Message + Alice's Private Key ────> Signature
  Message + Signature + Alice's Public Key ────> "Verified: signed by Alice" or "FAIL"
```

---

## 2. Why the Master-Key / Subkey Architecture?

If you use **one** keypair for everything — signing, encrypting,
authenticating — and that key lives on your YubiKey:

- If the YubiKey is lost, your entire identity is compromised.
- You can't selectively revoke "I lost my signing ability" while keeping
  "I can still decrypt old emails."
- If RSA 4096 is ever broken, your entire identity goes with it.

The solution: **a hierarchy.**

```
  Master Key (C)
     │ only used to certify
     │ subkeys and extend expiry
     │
     ├── Signing Subkey (S)     ─── on YubiKey
     ├── Encryption Subkey (E)  ─── on YubiKey
     └── Auth Subkey (A)        ─── on YubiKey
```

- The master key spends almost its entire life offline in a safe.
- Subkeys do the daily work. If a subkey is compromised, you can revoke
  *just that subkey* with your master key and issue a new one. Your identity
  (the master key fingerprint) stays the same.
- When your key expires, you extend it with the master key — without
  changing the subkeys on your YubiKey.

### Key Usage Flags

Every GPG key has capability flags embedded in the certificate:

| Flag | Letter | What It Allows                            |
|------|--------|-------------------------------------------|
| C    | Certify| Sign other keys (create/revoke subkeys)   |
| S    | Sign   | Sign data (emails, files, git commits)   |
| E    | Encrypt| Decrypt data (only this can decrypt)      |
| A    | Auth   | Prove identity (SSH, TLS client certs)   |

A key can have multiple flags. Our master key has only `C` (and `SC` for
backward compatibility). The subkeys each have exactly one: `S`, `E`, or `A`.

---

## 3. How the YubiKey OpenPGP Applet Works

The YubiKey isn't just a USB drive that stores files. It's a **secure
element** — a tiny computer with its own processor, memory, and
cryptographic accelerator.

### The Applet Concept

Think of the YubiKey as a phone that can run multiple apps:

| Applet     | What It Does                     |
|------------|----------------------------------|
| OpenPGP    | PGP signing, decryption          |
| PIV        | Smartcard for Windows/DoD        |
| OTP        | One-time password (press once)   |
| FIDO2      | WebAuthn / passwordless auth     |
| OATH       | TOTP (time-based one-time codes) |

These are independent. Configuring OpenPGP doesn't affect FIDO2 or OTP slots.

### The Key Slots

The OpenPGP applet has exactly **three key slots**:

| Slot          | Key Type              | Default Algorithm |
|---------------|-----------------------|-------------------|
| Signature (SIG) | Signing subkey      | RSA 2048          |
| Decryption (DEC) | Encryption subkey  | RSA 2048          |
| Authentication (AUT) | Auth subkey   | RSA 2048          |

When you do `keytocard` and select "option 1" (Signature key), GPG writes the
private key material into the SIG slot. The subkey on your computer is then
replaced with a **stub** — just the card serial number and the public portion.

### The On-Device Crypto Operation

When you decrypt a file:

1. GPG on your computer says "I need RSA decryption with subkey X."
2. It checks its keyring and finds a stub: "subkey X is on card 0006XXXXXX."
3. It sends the ciphertext to the YubiKey via `scdaemon`.
4. The YubiKey checks if the PIN has been entered (and if touch is required).
5. If authorized, it performs RSA decryption **inside the secure element**.
6. The result (the session key) is sent back to GPG.
7. **The private key never leaves the YubiKey** — not even for the
   operation.

This is fundamentally different from a software key. In software, the private
key is decrypted in RAM and used there. An attacker with memory access can
steal it. With a YubiKey, the key material is in a hardware vault that
explicitly refuses to export it.

### PIN Verification

The YubiKey has two PIN verification states:

| Code | Name    | Credential | Attempts | Purpose                 |
|------|---------|------------|----------|-------------------------|
| CHV1 | User PIN| PIN (6-127 chars)| 3  | Normal use (sign/decrypt)|
| CHV3 | Admin PIN| Admin PIN    | 3    | Card admin (key import) |

(CHV2 doesn't exist on OpenPGP cards — it's reserved.)

- PIN is cached by `gpg-agent` so you don't re-enter it for every operation.
  You can configure the cache timeout in `~/.gnupg/gpg-agent.conf`:
  ```
  max-cache-ttl 300          # 5 minutes
  default-cache-ttl 300
  ```
- After 3 wrong PIN entries, the card locks CHV1. You need the PUK (PIN
  Unblocking Key) or Admin PIN to unlock it via the `passwd` menu.
- After 3 wrong Admin PIN entries, the card is **permanently locked** and
  only a factory reset (`ykman openpgp reset`) can restore it — deleting
  all keys.

### Touch Policy

The YubiKey has a capacitive touch sensor (the gold contact). The touch policy
controls whether the YubiKey requires a physical tap before performing an
operation:

- **Off**: Any process that knows the PIN can use the key without touch.
  This means if malware on your computer has PIN access (e.g., keylogged),
  it can sign/decrypt silently.
- **On**: Every operation requires a tap. Your YubiKey LED blinks, you tap,
  the operation proceeds. This gives you a **physical confirmation** every
  time the key is used.
- **Cached**: After one tap, the same application can reuse it for ~15
  seconds without tapping again. Good for batch operations.
- **Fixed**: Like `On`, but the policy itself cannot be changed without
  factory reset. Maximum security, maximum annoyance if you change your mind.

The touch policy is stored in the YubiKey's flash memory and is independent
of the key material. You can change touch policy even after keys are loaded
(unless you set `Fixed`).

---

## 4. How Challenge-Response Works (KeePass)

This is the other YubiKey feature you're using, and it's a completely
different system from OpenPGP.

### The OTP Slots

Your YubiKey has two "OTP slots" that are actually general-purpose
credential slots:

| Slot | Default              | Can Be Used For              |
|------|----------------------|------------------------------|
| 1    | YubiKey OTP (press once = types a one-time password) | HMAC-SHA1 challenge-response |
| 2    | Secondary OTP (long press) | HMAC-SHA1 challenge-response |

Each slot stores a **20-byte secret key**. By default, the YubiKey was
configured at the factory with a unique AES key for Yubico's OTP protocol.
When you run `ykman otp chalresp --generate`, you overwrite that slot with
your own secret.

### The Protocol

Challenge-response is simple:

```
                       Challenge (random bytes)
Your Computer ────────────────────────────────────────> YubiKey
                                                       │
                                                       │ HMAC-SHA1(
                                                       │   secret stored in slot,
                                                       │   challenge bytes
                                                       │ )
                                                       │
                       Response (first 16 bytes of     v
Your Computer <───────────────────────────────────────── HMAC output)
```

1. **You unlock the database** → KeePassXC sends a 64-byte random challenge
   to the YubiKey.
2. **The YubiKey computes** HMAC-SHA1(secret, challenge) using the secret
   stored in the OTP slot. The result is deterministic — same secret +
   same challenge = same response, always.
3. **The response** (truncated to 16 bytes — 128 bits) is returned to
   KeePassXC.
4. **KeePassXC hashes** the challenge + response together with your master
   password and uses that to derive the AES-256 encryption key for the
   database.
5. **To unlock**, KeePassXC repeats the challenge (which it stores in the
   database file header), asks the YubiKey for the response, and checks
   whether the derived key decrypts the database correctly.

### Why This Is Secure

- The secret is **generated on the device** and never exposed to the computer
  (when using `--generate`).
- The YubiKey **never reveals the secret** — you can't ask it "what's your
  secret?" You can only ask "what's the HMAC of this challenge?"
- The challenge is different every time you access the database, but since
  KeePassXC stores the challenge in the database header, it doesn't need to
  be secret.
- Without the physical YubiKey with the correct secret, no amount of
  computation can derive the correct response to the stored challenge. The
  database is effectively encrypted with a key that only exists inside the
  YubiKey.

### Why You Need a Backup YubiKey

Because the secret **cannot be extracted from a properly programmed YubiKey**,
if you only have one YubiKey and you lose it, the database is permanently
inaccessible.

That's why the `ykman otp chalresp --secret <hex> 2` command exists: it lets
you explicitly read the secret from one YubiKey (breaching the hardware
vault, but under your control) so you can program it into a backup. This is
an intentional tradeoff — you sacrifice some security for a recovery path.

---

## 5. The GPG Keyring and Trust Model

### Web of Trust vs. Centralized PKI

TLS (HTTPS) uses a **Certificate Authority** model: a small set of companies
decide who is who. If Verisign says you're Nyx, then you're Nyx. This is
centralized and requires trusting third parties.

PGP uses a **Web of Trust**: you decide who to trust. When you sign someone
else's public key, you're saying "I've verified this person's identity."
People who trust you can then trust that person through you. It's
decentralized — like a social network of identity vouching.

### GPG Keyring

Your `~/.gnupg/` directory:

```
~/.gnupg/
├── pubring.kbx              # Public key database (SQLite-like)
├── trustdb.gpg              # Trust database (who you trust)
├── private-keys-v1.d/       # Secret key stubs and metadata
├── openpgp-revocs.d/        # Auto-generated revocation certificates
├── gpg-agent.conf           # Agent configuration (PIN caching, etc.)
└── sshcontrol               # Which keys are allowed for SSH
```

When you move subkeys to the YubiKey, the files in `private-keys-v1.d/` are
reduced to stub files that contain only:

- The corresponding public key
- The card serial number
- The key slot on the card

That's it. No private key material.

---

## 6. Why RSA 4096 and Not ECC?

The YubiKey 5 supports both RSA and ECC (Elliptic Curve Cryptography) for its
OpenPGP applet. You could use ECC keys (like Ed25519 or NIST P-384) instead of
RSA. The tradeoffs:

| Feature       | RSA 4096                      | ECC (Ed25519 / P-384)         |
|---------------|-------------------------------|-------------------------------|
| Key size      | Large (public ~500 bytes)     | Tiny (public ~32-64 bytes)    |
| Speed         | Slower (big math)            | Faster (smaller math)         |
| Security      | 128-bit equivalent           | 128-192 bit equivalent        |
| Compatibility | Universal (everyone supports)| Good but not universal        |
| YubiKey support | Yes (all models)           | Yes (YubiKey 5+, not NEO)     |

General wisdom: RSA 4096 is the safest choice for broad compatibility. ECC is
the modern choice for performance. For a PGP key that sits in a safe and
rarely moves, RSA 4096's speed disadvantage doesn't matter.

---

*The YubiKey is a tool that bridges a conceptual gap: it makes the abstract
math of asymmetric cryptography something you can touch, tap, and physically
possess. The PIN, the gold contact, the USB plug — these turn cryptographic
identities into something you hold in your hand.*
