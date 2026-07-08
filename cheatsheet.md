# YubiKey PGP — Quick Reference Card

One-liner commands for when you already know the flow.

## Key Generation

```bash
# Generate master key (certify only, RSA 4096)
gpg --expert --full-generate-key

# Add subkeys (S, E, A) — repeat three times inside gpg> prompt
gpg --expert --edit-key 0xKEYID
gpg> addkey       # Sign (toggle S on, set 4096)
gpg> addkey       # Encrypt (RSA encrypt only, 4096)
gpg> addkey       # Auth (toggle A on, set 4096)
gpg> save
```

## Revocation Certificate

```bash
gpg --output revoke-0xKEYID.asc --gen-revoke 0xKEYID
```

## YubiKey PIN Setup

```bash
gpg --edit-card
gpg/card> admin
gpg/card> passwd     # 3 = Admin PIN, 1 = User PIN, 2 = Unblock
gpg/card> quit
```

## Set Touch Policy

```bash
ykman openpgp keys set-touch SIG on      # or: cached, off, fixed
ykman openpgp keys set-touch ENC on
ykman openpgp keys set-touch AUT off
```

## Move Subkeys to YubiKey

```bash
gpg --edit-key 0xKEYID
gpg> key 1         # select signing subkey
gpg> keytocard     # option 1 (Signature)
gpg> key 1         # deselect
gpg> key 2         # select encryption
gpg> keytocard     # option 2 (Encryption)
gpg> key 2         # deselect
gpg> key 3         # select auth
gpg> keytocard     # option 3 (Authentication)
gpg> key 3         # deselect
gpg> save
```

## Verify Keys Are on Card

```bash
gpg --list-secret-keys --keyid-format=long 0xKEYID
# Look for ssb>  with card-no: 0006XXXXXX
```

## Testing

```bash
# Sign
echo "test" | gpg --clearsign --default-key 0xKEYID

# Encrypt+Decrypt
echo "test" | gpg --encrypt --recipient you@email.com | gpg --decrypt
```

## YubiKey Factory Reset

```bash
ykman openpgp reset
```

## KeePass — Program Slot 2 for HMAC-SHA1

```bash
# Generate new secret on-device (recommended)
ykman otp chalresp --generate 2

# Program second YubiKey with same secret
ykman otp chalresp --secret <hex> 2

# Read existing secret from slot 2 (requires touch)
ykman otp chalresp --secret 2
```

## Backup

```bash
gpg --export-secret-keys --armor 0xKEYID > master-key.asc
gpg --export-secret-subkeys --armor 0xKEYID > subkeys.asc
gpg --export --armor 0xKEYID > public-key.asc
```

## Refresh Stubs (after YubiKey re-insert)

```bash
gpg --card-status
```

## Emergency Recovery

```bash
# YubiKey lost — restore from subkey backup onto new YubiKey
gpg --import subkeys.asc
gpg --card-status      # auto-links to card
gpg --edit-key 0xKEYID
# repeat keytocard for each subkey

# PIN blocked — unblock with PUK
gpg --edit-card
gpg/card> admin
gpg/card> passwd  # option 2

# Admin PIN also lost — full reset and restore
ykman openpgp reset
gpg --import master-key.asc
# generate new subkeys or re-import from subkey backup
```
