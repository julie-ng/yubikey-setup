# Personal YubiKey GPG Setup 

After ~7 years, it was time to upgrade to new physical keys and newer encryption algorithms. My existing keys are vulnerable to the [EUCLEAK](https://www.yubico.com/support/security-advisories/ysa-2024-03/). And since I no longer work for Microsoft, I no longer need to [manage multiple identities](https://julie.io/blog/setup-git-multiple-gpg-and-yubikeys). 

## Setup Features

I mostly use YuibiKeys to sign my git commits and for authentication as a passkey as well as gpg encryption/decryption of `.netrc` so long lived tokens are not stored in plain text on my computer.

### Security Features

_Based on [drduh/YubiKey-Guide](https://github.com/drduh/YubiKey-Guide), but adapted for my personal requirements._

- **Cryptography**: prefer modern Curve25519 algorithms over RSA
- **Accounts not Hackable without Physical Access**: 
  - All private keys stored offline (encrypted USB or YubiKey)
  - YubiKey configured to require PIN and physical touch on  the gold contact to authorize operations
  - Config Key Derived Function (KDF) to ensure PINs are hashed when crossing USB bus
- Certify passphrase is stored in password manager, separate from USB stick (two factor)

### Two YubiKeys

- **Primary/Certify Key**  
  Stored separately offline on an APFS-encrypted USB drive. 

- **Subkeys**  
  Setup two physical YubiKeys that have _identical_ subkeys for redudancy and to be used interchangeably:
  - [YubiKey 5C Nano](https://www.yubico.com/de/product/yubikey-5-series/yubikey-5c-nano/) - daily driver and remains plugged into laptop. Removed when traveling.
  - [YubiKey 5C NFC](https://www.yubico.com/de/product/yubikey-5-series/yubikey-5c-nfc/) - backup and used when traveling.

### Required for Setup

- **Mac**  
  FileVault enabled and used for generating key and encrypting USB drive.

- **Password Manager**  
  For PINs and most importantly storing **certify passkey** separately from USB stick. Both are needed to use the offline primary key.

## Security Terminology & Abbreviations

<details>
  <summary>
    Expand to read useful terms and brief definitions, useful for understanding guide. 
  </summary>

### Abbreviations 

| Term | Description |
|------|-------------|
| **GPG / GnuPG** | GNU Privacy Guard, the signing/encryption software. |
| **Certify key** | the primary `[C]` key. Issues and revokes subkeys. |
| **Subkey** | `[S]` sign, `[E]` encrypt, `[A]` authenticate. |
| **ed25519** | elliptic-curve algorithm for SIGNING and SSH auth. Cannot encrypt. |
| **cv25519** | (Curve25519 / X25519) Algorithm used for ENCRYPTION. |
| **KDF** | Key Derived Function. Makes the YubiKey hash the PIN on the host before sending it to the card, so the PIN isn't transmitted in plaintext. |
| **stub** | after `keytocard`, the on-disk key becomes a pointer ("stub") to a specific YubiKey serial number. |
| **User PIN / Admin PIN** | OpenPGP applet PINs. Defaults `123456` / `12345678`. |

### Applets

**Applets** - refers to sandboxed Java program running on YubiKey itself. Each key has following applets:

| Applet | Description |
|--------|-------------|
| **OpenPGP** | GPG keys, signing/decryption, User PIN + Admin PIN.  |
| **PIV** | X.509 smartcard certs, with PIN + PUK + Management Key |
| **FIDO2 / U2F** | passkeys/WebAuthn, own PIN |
| **OATH** | TOTP codes |
| **Yubico OTP** | touch-to-type one-time codes |

</details>

---

# Setup Instructions

Please note some personal preferences in this guide:

- Prefer interactive `--edit-key` over scriptable `--pinentry-mode=loopback` to avoid secrets landing in `.zsh_history`, etc. 
- I used drdruh's config during setup, but not saved for daily use. In a setup once and forget scenario, I prefer default formats, easier for debugging later.

### Pre-requisites checklist

- [ ] Mac: FileVault confirmed ON (System Settings → Privacy & Security → FileVault).
- [ ] USB Stick: format as APFS-encrypted 
- [ ] YubiKeys: confirm not susceptible to EUCLEAK by checking firmware is **version 5.7.0 or higher** via `ykman info`
- [ ] Install tools
  ```bash
  brew install gnupg ykman pinentry-mac
  ```

## Step 1. Setup Isolated working directory

Use `mktemp` to generate in a throwaway `GNUPGHOME`, which will hold our private keys until transferred to YubiKey. This temp directory evaporates on reboot. 

> [!NOTE]
> `GNUPGHOME` requires `export` in order for `gpg` to read it from the shell environment.

```bash
export GNUPGHOME=$(mktemp -d)
echo "$GNUPGHOME"
/var/folders/tc/k2p_qwn37mvldzjps_8x49a6t0000gn/T/tmp.aB3kPzQ9rN # example output
```

Now switch to the temp directory

```bash
cd "$GNUPGHOME"
```

## Step 2. Generate strong Certify passphrase

Once in the temp directory, generate the passphrase directly to a `certify-pass.txt` file in `$GNUPGHOME`:

```bash
LC_ALL=C tr -dc "A-Z2-9" < /dev/urandom | tr -d "IOUS5" | \
    fold -w 4 | paste -sd - - | head -c 29 > "$GNUPGHOME/certify-pass.txt"
```

This passphrase is needed later for generating keys.

## Step 3. Setup Variables

Before running commands, set the passfile location and set a few variables for re-use. **Replace fictional name and email below with your information.**

```bash
PASSFILE="$GNUPGHOME/certify-pass.txt"
IDENTITY="Anya Forger <anya.forger@spyxfamily.com>" # replace with your information
EXPIRATION=2y                                       # 2 year expiry
```

> [!TIP]
> The email address used in signed commits will appear **publicly** on GitHub.com. I recommend using a professional email address over a personal email.

## Step 4. Generate keys on Mac

> [!IMPORTANT]
> Before adopting this setup, check that every system you sign or authenticate to accepts Curve25519 keys — some older services still require RSA.

### Curve25519 Cryptography

My [old setup (2018)](https://julie.io/blog/setup-git-multiple-gpg-and-yubikeys) used [RSA cryptography](https://en.wikipedia.org/wiki/RSA_cryptosystem), which has wider support but is older. 

For this 2026 setup I chose the faster and more modern [Curve25519-family cryptography](https://en.wikipedia.org/wiki/Curve25519) with EdDSA for certify/sign/auth and ECDH for encryption (EdDSA can't encrypt):

| Key | Function | Algorithm | Curve |
|-----|----------|-----------|-------|
| `[C]` Primary Key | Certification | EdDSA | `ed25519` |
| `[S]` Subkey | Signature | EdDSA | `ed25519` |
| `[A]` Subkey | Authentication | EdDSA | `ed25519` |
| `[E]` Subkey | Encryption | ECDH | `cv25519` |

### 4A. Generate Certify key 

The certify key has no expiry date and is used to re-generate subkeys after they expire in 2 years.

> [!NOTE]
> To set an expiry date, replace `never` with your expiry time period, e.g. `5y` for five years.

Generate the certify `[C]` key:

```bash
gpg --batch --passphrase-file "$PASSFILE" \
    --quick-generate-key "$IDENTITY" ed25519 cert never
```

Capture the key ID and fingerprint into a `keyinfo.txt` for configuration later.

```bash
KEYID=$(gpg -k --with-colons "$IDENTITY" | awk -F: '/^pub:/ { print $5; exit }')
KEYFP=$(gpg -k --with-colons "$IDENTITY" | awk -F: '/^fpr:/ { print $10; exit }')
{ echo "KEYID=$KEYID"; echo "KEYFP=$KEYFP"; } > "$GNUPGHOME/keyinfo.txt"
```

### 4B. Generate SIGN subkey 

```bash
# Signing subkey
gpg --batch --pinentry-mode=loopback --passphrase-file "$PASSFILE" \
    --quick-add-key "$KEYFP" ed25519 sign "$EXPIRATION"
```

### 4C. Generate AUTH subkey 

```bash
# Authentication subkey
gpg --batch --pinentry-mode=loopback --passphrase-file "$PASSFILE" \
    --quick-add-key "$KEYFP" ed25519 auth "$EXPIRATION"
```

### 4D. Generate ENCRYPT subkey 

```bash
# Encryption subkey — NOTE: cv25519, not ed25519
gpg --batch --pinentry-mode=loopback --passphrase-file "$PASSFILE" \
    --quick-add-key "$KEYFP" cv25519 encrypt "$EXPIRATION"
```

### 4E. Verify keys were generated

Verify `[C]`, `[S]`, `[A]`, `[E]` keys were generated. Run

```bash
gpg -K
```

and expect something like:

```
sec   ed25519/0x... [C]
ssb   ed25519/0x... [S] [expires: ...]
ssb   ed25519/0x... [A] [expires: ...]
ssb   cv25519/0x... [E] [expires: ...]
```

### 4F. Export the public key

Export the public key so we can [upload it to GitHub for signature verification](https://docs.github.com/en/authentication/managing-commit-signature-verification/adding-a-gpg-key-to-your-github-account).

```bash
gpg --armor --export "$KEYID" > "$GNUPGHOME/$KEYID-pub.asc"
```

## Step 5. Back up Keyring to USB Stick

To provision a **second** YubiKey, we need to backup the existing `GNUPGHOME`, which has our private keys. After configuring the first YubiKey, the folder will only contain _stubs_ to the subkeys. 

> [!IMPORTANT]
> **USB Stick Name**: I named my USB drive `GPG-Backup` and thus it is mounted at `/Volumes/GPG-Backup/`. Adjust the path to match your setup.

### 5A. Move Certify Passphrase to Password Manager

Read the passfile to get the passphrase (formatted like `XXXX-XXXX-XXXX-XXXX-XXXX-XXXX`)

```
less $PASSFILE
```

and **save the contents to a password manager**, e.g. 1Password. 

> [!CAUTION]
> Double check your saved the passphrase **before** deleting the file. If you lose it, you can not renew your subkeys.

Delete the passfile so we don't save the certify keys and required passphrase together.

```bash
rm $PASSFILE
```

### 5B. Copy the non-secret info to USB stick

Copy the public key and info text file that contains our key ID and key fingerprint to the USB stick root:

```bash
mv "$GNUPGHOME/$KEYID-pub.asc"  /Volumes/GPG-Backup/
mv "$GNUPGHOME/keyinfo.txt"  /Volumes/GPG-Backup/
```

### 5C. Copy the keyring to USB stick

Now copy the keyring, i.e. `$GNUPGHOME` to the USB Stick.

```bash
cp -av "$GNUPGHOME" /Volumes/GPG-Backup/
```

Now, we'll rename the `tmp.xxxxx` folder to a human-friendly `gnupghome-backup` name.

```bash
mv /Volumes/GPG-Backup/"$(basename "$GNUPGHOME")" /Volumes/GPG-Backup/gnupghome-backup
```

### USB Directory Structure

If done correctly, you should have something like this

```
/Volumes/GPG-Backup/
├── gnupghome-backup/        
│   ├── pubring.kbx
│   ├── private-keys-v1.d/   # contains certify key + subkeys
│   ├── trustdb.gpg
│   └── ...
├── keyinfo.txt              # KEYID / KEYFP (not secret)
└── <KEYID>-pub.asc          # exported public key
```

If you see a file named `pubring.kbx~` that ends with a tilde `~`, that is a stale copy and safe to delete from the USB stick.

### 5D. Verify the backup before trusting it

This is a paranoid step to confirm the SECRET keys are readable from the renamed stick copy:

```bash
GNUPGHOME=/Volumes/GPG-Backup/gnupghome-backup gpg -K
```

which should output something like this

```
/Volumes/GPG-Backup/gnupghome-backup/pubring.kbx
------------------------------------------------
sec   ed25519... [C]
uid   ...
ssb   ed25519... [S] [expires: ...]
ssb   ed25519... [A] [expires: ...]
ssb   cv25519... [E] [expires: ...]
```

> [!IMPORTANT]
> Check that `sec` and `ssb` lines have **no** greater-than `>` or hash `#` markers, i.e. no `sec#` or `ssb>`. If you follow the guide exactly and working with freshly generated keys, you should be fine.

Again, this is sanity check confirmation that the copy worked properly before continuing.

### 5E. Eject the stick

Before we can eject the USB stick, we need to kill a `gpg-agent` that was spawned in previous step (5D) when we verified the backup:

```bash
gpgconf --kill all 2>/dev/null; pkill gpg-agent; sleep 1
lsof | grep -i gpg-backup 
```

If `lsof` returns no results, we can eject the stick via Mac OS Finder **or** by running

```bash
diskutil eject /Volumes/GPG-Backup
```

Pull the stick out of the computer for good measure. We'll plug it back in later when configuring the second key.

> [!NOTE]
> **Defense in depth** – past mistakes and errors in AI-generated drafts of this guide put destructive `keytocard` steps *before* ejecting the stick — one wrong `GNUPGHOME` and the backup goes with it. A physically unmounted stick is protected from destructive commands.

---

## 5. Configure + load YubiKey #1 (nano)

Plug in the nano. Confirm it's seen:

```bash
gpg --card-status
```

### 5a. Enable KDF FIRST (before PINs, before keys)

```bash
gpg --command-fd=0 --pinentry-mode=loopback --card-edit <<EOF
admin
kdf-setup
12345678
EOF
```

> If you get `Conditions of use not satisfied`, the card already had PINs/keys changed — KDF must be first on a fresh applet. On a factory-state applet this works.
>
> Note the `12345678` in the heredoc is the *default* Admin PIN — this only works because KDF runs BEFORE 5b changes the PIN. Order matters: KDF (5a) → change PINs (5b) → login + keytocard (5c/5d). If you ever reorder, this hardcoded default breaks.

### 5b. Change PINs off defaults

Pick values and save them in 1Password. User PIN ≥ 6 digits, Admin PIN ≥ 8.

```bash
# interactive is fine and clearer here:
gpg --card-edit
# then:
#   admin
#   passwd
#   1  -> change User PIN  (default 123456)
#   3  -> change Admin PIN (default 12345678)
#   q
#   quit
```

> Save in 1Password: "YubiKey nano — OpenPGP User PIN / Admin PIN".

### 5c. Set card attributes: login (required) + holder name (optional)

Do this **interactively** — these are admin-authorized writes, so the card needs the **Admin PIN**. A heredoc with `--pinentry-mode=loopback` supplies no PIN and fails with `error setting login data: Bad PIN`. Interactive prompts for the Admin PIN properly (via pinentry) and keeps the PIN off the command line. Both attributes are set in the same session:

```bash
gpg --edit-card
```

Then at the `gpg/card>` prompt:

```
admin
login        # REQUIRED. Prompts for login data — type your identity, e.g.
             #   Your Name <foo@bar.com>
name         # OPTIONAL. Prompts for Surname then Given name SEPARATELY:
             #   Surname:    Doe
             #   Given name: Jane
             # (shows up as "Holder:" in the pinentry dialog — handy for telling
             #  the two cards apart at a glance)
quit
```

You'll be prompted for the **Admin PIN** to authorize the writes (cached after the first, so likely once for both).

> Both `login` and `name` are **card attributes** (like PINs and touch policy), not key material — set them per-card. So you'll set the name again on the 5C NFC in step 6. The OpenPGP `name` field is length-limited (surname + given name, ~39 chars total); long names get truncated.

> If you see `Bad PIN`, you're either entering the wrong Admin PIN or (if scripted) supplying none. Check `ykman openpgp info` → `Admin PIN tries remaining` — it starts at 3, and hitting 0 locks the Admin PIN (recoverable only via `ykman openpgp reset`, which wipes the applet). Don't brute-force it; stop and verify the PIN first.

### 5d. Transfer subkeys to the card (keytocard)

The stick is already ejected (step 4d), so `keytocard` physically cannot harm the backup — an unmounted volume can't be modified. `keytocard` is **destructive** to the on-disk keys (each subkey becomes a stub pointing at the card), which is fine because the backup is safe on the ejected stick.

**Confirm you're on the temp dir**, then transfer. Use plain interactive `--edit-key` — do NOT use `--pinentry-mode=loopback` here. `keytocard` needs TWO secrets (the Certify passphrase to unlock the on-disk key, AND the YubiKey Admin PIN to authorize the card write), and loopback can only feed the passphrase via `--passphrase-file` — it has no way to supply the Admin PIN, so it fails with `KEYTOCARD failed: Bad PIN`. Plain interactive mode routes both through the pinentry-mac dialog:

```bash
echo "$GNUPGHOME"                # sanity: should be /var/folders/.../T/tmp.XXXXX
gpg -K                           # sanity: full ssb keys, NO ">" yet
gpg --edit-key "$KEYID"          # NO --pinentry-mode, NO --passphrase-file
```

> **The two secrets and how to tell them apart** — during `keytocard` you'll be prompted for both, and reading the wording is the only reliable way to know which is which:
> - "Please enter your passphrase, so that the secret key can be **unlocked** for this session" → the **Certify passphrase** (the long `XXXX-XXXX-…` string from `"$GNUPGHOME/certify-pass.txt"` / 1Password). NOT the Admin PIN.
> - A prompt referencing the **Admin PIN** or the card/serial → the YubiKey **Admin PIN** (the one set in 5b). This authorizes the card write.
>
> GPG usually **caches** the Certify passphrase after the first unlock, so subkeys 2 and 3 may only prompt for the Admin PIN (or nothing, if that's cached too). Fewer prompts later = normal caching, not a skipped step.
>
> Pasting the wrong secret into a prompt gives a Bad-PIN/bad-passphrase error and decrements a counter — read each prompt before typing.

Then at the `gpg>` prompt. **First type `list`** to see the subkey order — positions count from the top with the primary as 0. For this keyset the order is:

```
sec  ...usage: C    <- position 0 (primary)
ssb  ...usage: S    <- position 1 (signing)
ssb  ...usage: A    <- position 2 (authentication)
ssb  ...usage: E    <- position 3 (encryption)
```

> Selection is shown by a `*` next to the line (e.g. `ssb*`), NOT an arrow UI. `key N` takes the position NUMBER (not the usage letter — `key s` does nothing useful). Typing `key N` again toggles it OFF. Before each `keytocard`, run `list` and confirm exactly ONE subkey shows `*`.

Go one subkey at a time, verifying selection each time:

```
key 1          # select signing [S]
list           # confirm ONLY the usage:S line shows ssb*
keytocard      # choose slot 1 (Signature key)   <- enter Admin PIN when prompted
key 1          # deselect

key 2          # select authentication [A]
list           # confirm ONLY the usage:A line shows ssb*
keytocard      # choose slot 3 (Authentication key)
key 2          # deselect

key 3          # select encryption [E]
list           # confirm ONLY the usage:E line shows ssb*
keytocard      # choose slot 2 (Encryption key)
key 3          # deselect

save
```

> Gotchas:
> - **Slot numbers ≠ list positions.** By card slot/function: Signature→**1**, Encryption→**2**, Authentication→**3**. gpg's prompt names each slot — read the prompt and match by NAME, not by my numbers.
> - **One `*` at a time.** Toggle each subkey off before selecting the next.
> - **Admin PIN** (not User PIN) is requested during `keytocard` — it authorizes the card write.
> - If your `list` shows a different order than above, follow YOUR order — match by the `usage:` letter, not the position number.

Verify the subkeys now show `ssb>` (the `>` = on smartcard):

```bash
gpg -K
```

### 5e. (Recommended) require touch per operation

By default the OpenPGP applet signs/decrypts/authenticates using just the cached PIN — **no touch needed**. (This is separate from FIDO2's always-on passkey touch; different applet.) That means with the nano permanently plugged in and the PIN cached, any process running as you could ask the card to sign/decrypt silently. A touch policy breaks that: the card won't perform the operation until a human physically taps it — malware can't fake a finger. Worth enabling *because* the nano lives plugged in.

**Modes** (set per operation, and per YubiKey — these live on the card, not the keys, so they don't affect redundancy):

| Mode | Behavior | Use when |
|------|----------|----------|
| `off` | No touch; cached PIN is enough | default; not recommended for an always-plugged-in key |
| `on` | Touch required for **every** operation | rare, high-value ops where per-op confirmation is cheap |
| `cached` | Touch required, then a ~15s window where further ops need no re-touch | frequent ops (signing during a rebase = one tap for the burst) |
| `fixed` | Like `on`, but **can't be changed** without wiping the applet | avoid while tuning — removes your ability to adjust later |

**Recommended for this setup:**

```bash
ykman openpgp keys set-touch sig cached   # signing: frequent → cached (one tap per burst)
ykman openpgp keys set-touch aut cached   # SSH auth: occasional → cached
ykman openpgp keys set-touch dec cached   # decryption: frequent (.netrc) → cached
```

Each prompts for the **Admin PIN**.

Reasoning:
- **`sig` → `cached`:** commit signing is your highest-frequency op. `cached` still blocks silent background signing (malware can't get the first touch), but a rebase or rapid commits only cost ONE tap thanks to the ~15s window. Avoids the per-commit nag.
- **`aut` → `cached`:** SSH auth is less frequent; same logic, one tap covers a working session.
- **`dec` → `cached`:** if you use `.netrc.gpg` for git credentials, **every** git push/pull over HTTPS decrypts it — with `dec on` that's a tap on every git operation, which gets old fast. `cached` keeps the protection (malware still can't get the first tap to decrypt your credentials silently) while letting a burst of git commands cost one tap. Use `dec on` instead only if you decrypt rarely and want max protection.
- **Avoid `fixed`** until you know your tolerance — it's `on` you can't undo without `ykman openpgp reset`.

> To change a policy later (e.g. you started with `dec on` and it taps too often): `ykman openpgp keys set-touch dec cached` — same command, prompts for Admin PIN, takes effect immediately. As long as you never used `fixed`, you can always relax or tighten. Per-card, so change it on each YubiKey you want affected.

> Per-key, not per-subkey: you set this again on the 5C NFC in step 6, and can choose differently (e.g. stricter `on` for `sig` on the travel key since it leaves the house). The subkeys are identical either way — only the touch gate differs.

### 5f. (Optional) disable Yubico OTP

The YubiKey's **Yubico OTP** applet types a one-time password (a long `cccccc…`-prefixed string) whenever its contact is touched while a text field has focus. On an always-plugged-in nano this fires by accident now and then — bumping the key dumps an OTP into whatever field has focus (a chat box, a terminal, a commit message). It's harmless (a single OTP is useless out of context and single-use) but annoying.

If you don't use Yubico OTP for anything, disable it:

```bash
ykman config usb --disable OTP
```

> - Reversible: re-enable later with `ykman config usb --enable OTP`.
> - This disables OTP over USB. Add `--disable OTP` under `ykman config nfc …` too if you want it off over NFC as well (relevant for the 5C NFC, not the nano).
> - Does NOT affect OpenPGP, FIDO2, PIV, or OATH — only the OTP applet.
> - Accidental touches will then do nothing instead of emitting an OTP.

### 5g. Finish with the nano (verify, eject, clean up agents)

The nano is fully provisioned. Before moving to the 5C NFC, confirm the result and clear agent state so step 6 starts clean.

```bash
# 1. Final confirmation the nano has all three subkeys
gpg -K                    # three ssb> lines, each with "Card serial no." = nano's

# 2. (optional) note the nano's OpenPGP card serial for your records
gpg --card-status | grep -i "serial"

# 3. Physically remove the nano — no "safe eject" needed for a YubiKey
#    (that ceremony is only for storage volumes like the backup stick)

# 4. Kill all gpg-agents AND scdaemons so nothing stays pinned to this temp dir,
#    the nano's cached state, or the card's USB interface. `gpgconf --kill all` only
#    reaps daemons for the CURRENT $GNUPGHOME — a scdaemon spawned against a temp dir
#    can survive bound to that (now-dead) homedir and squat on the card, causing
#    "selecting card failed: Operation not supported by device" later. `pkill` clears
#    them regardless of homedir.
gpgconf --kill all 2>/dev/null; pkill gpg-agent; pkill scdaemon; sleep 1
ps aux | grep -iE "scdaemon" | grep -v grep   # expect empty — no squatters left
```

> At this point your temp-dir keyring holds **stubs** pointing at the nano (the full subkeys were consumed by `keytocard`). That's expected. Step 6 wipes this temp dir and restores **full** keys from the stick so the 5C NFC can receive the same subkeys.

---

## 6. Load YubiKey #2 (5C NFC)

Picking up from 5g: agents are killed, the nano is out, and the temp dir holds stubs. Now restore full keys from the backup so the 5C NFC can receive the *same* subkeys.

**Re-insert the backup stick** (you ejected it in 4d) and unlock it:

```bash
ls /Volumes/GPG-Backup/          # confirm you see: gnupghome-backup
```

Then restore into a FRESH temp dir:

```bash
# Step out of the dir first — you can't cleanly delete a directory you're standing in
# (back in step 1 you cd'd into $GNUPGHOME to fetch gpg.conf):
cd ~

# Confirm $GNUPGHOME points at the INTERNAL temp dir before deleting it:
echo "$GNUPGHOME"
```

> ⚠️ **Look at that output before running the next line.** It should be an internal path like `/var/folders/tc/..../T/tmp.aB3kPzQ9rN`. If it shows anything under `/Volumes/`, STOP — `rm -rf` would wipe the backup on the stick. Visually confirming the target of a destructive `rm -rf` is the habit worth building; never run it on a variable you haven't just eyeballed.

```bash
rm -rf "$GNUPGHOME"                # only after confirming the echo above
export GNUPGHOME=$(mktemp -d)
cp -av /Volumes/GPG-Backup/gnupghome-backup/* "$GNUPGHOME"/
gpg -K   # CHECKPOINT: must show full ssb keys (NO ">"). If you see ssb>, STOP —
         # the restore pulled stubs, not full keys.
```

> ⚠️ This restores into a NEW temp dir, so `$GNUPGHOME` now has a different basename than in step 5. That's expected.

The helpers (`certify-pass.txt`, `keyinfo.txt`, `<KEYID>-pub.asc`) live at the **stick root** as siblings of `gnupghome-backup/` (set up that way in step 4), so the `cp` above restores a clean keyring without them — which is correct. You don't need the passphrase file restored: keytocard prompts for it interactively via pinentry (paste from 1Password). The public-key sibling is there if you need to re-import it.

Re-set the variables if the shell was lost. KEYID comes from `keyinfo.txt` (at the stick root). The passphrase doesn't need a shell variable — you'll type it at the pinentry prompt during keytocard:

```bash
source <(grep '^KEYID=' /Volumes/GPG-Backup/keyinfo.txt)
IDENTITY="Your Name <foo@bar.com>"          # if the shell was lost
```

After the restore + KEYID source, you're done reading from the stick. **Eject it now using the same 4d procedure** (kill agents → `lsof` empty → `diskutil eject` → confirm `echo "$GNUPGHOME"` is the internal temp dir). Then everything below is identical to step 5 — no eject baked into the middle:

Plug in the 5C NFC (remove the nano first), then run **5a–5g** against it exactly as for the nano:

- 5a — KDF setup
- 5b — Change User/Admin PINs (can be same values as the nano or different — your call; save in 1Password either way, labeled "YubiKey 5C NFC")
- 5c — Set card attributes: login (required) + holder name (optional), via interactive `gpg --edit-card`. Set the name on this card too if you set it on the nano.
- 5d — `keytocard` the three subkeys (plain `gpg --edit-key "$KEYID"`, NO loopback; Certify passphrase + Admin PIN at the pinentry prompts). Stick is already ejected.
- verify `gpg -K` shows `ssb>`
- 5e — Touch policy. Consider stricter settings here since the 5C NFC travels.
- 5f — Disable OTP if you want — for the 5C NFC, add the NFC variant too: `ykman config nfc --disable OTP`
- Finish (5g): verify, remove the 5C NFC, kill agents

Now both keys carry identical subkeys.

> Re-insert the stick afterward only if you still need it (e.g. to keep the backup updated). For daily use in step 7 you don't need the stick at all.

---

## 7. Daily-use setup (your real ~/.gnupg)

Switch back to your normal environment (you can `unset GNUPGHOME` or open a new shell).

Import the public key and trust it ultimately:

```bash
gpg --import /path/to/$KEYID-pub.asc   # or: gpg --recv-keys $KEYID if published
gpg --edit-key "$KEYID"
# trust
# 5
# y
# quit
```

With a YubiKey plugged in, create the stub:

```bash
gpg --card-status
```

**Agent config — skipped on purpose.** This setup keeps GnuPG's defaults: no `gpg.conf` and no `gpg-agent.conf`. The drduh configs offer crypto-preference pinning (largely redundant on modern GnuPG) and cosmetic listing formatting — not worth it for light signing/encryption use. Pinentry already works via the build default (`/opt/homebrew/bin/pinentry`, which auto-selects the terminal frontend when invoked from a TTY). No agent restart is needed because no config changed.

> If you ever sign commits from a **pure-GUI tool** (not a terminal) and hit a `No pinentry` / `Inappropriate ioctl for device` error, that's the terminal-only frontend having no TTY to attach to. Fix is one line — pin the GUI pinentry:
> ```bash
> echo "pinentry-program /opt/homebrew/bin/pinentry-mac" >> ~/.gnupg/gpg-agent.conf
> gpgconf --kill gpg-agent     # reload after the change
> ```
> (Intel Mac: `/usr/local/bin/pinentry-mac`.) Until that happens, nothing to do.

Add to your shell rc (`~/.zshrc`):

```bash
export GPG_TTY=$(tty)    # helps the terminal pinentry attach to the right TTY
alias yk-switch='gpg-connect-agent "scd serialno" "learn --force" /bye'  # see step 9
```

---

## 8. Git config + test

```bash
git config --global user.signingkey "$KEYID"
git config --global commit.gpgsign true
git config --global user.email "foo@bar.com"           # must match the key's UID + GitHub
```

> **Why `user.signingkey` is needed now (it may not have been before).** Without it, GPG picks a signing key implicitly — historically by matching `user.email` to a key's UID. That worked in the past when each key had a **distinct email/identity**: the email uniquely disambiguated which key to use, so `signingkey` was redundant. It's needed now because (a) there are **multiple secret keys** in the keyring (old + new), and (b) this setup uses a **single identity** across both new YubiKeys, so email no longer uniquely selects a key. With ambiguity present, implicit selection can silently sign with the *wrong* key (a wrong-key signature still succeeds — no error). Setting `signingkey` explicitly guarantees the new key is used. Passing the primary key ID is enough; GPG auto-uses its `[S]` subkey.

Add the public key to GitHub: Settings → SSH and GPG keys → New GPG key → paste the contents of `$KEYID-pub.asc`.

Test a real signed commit:

```bash
cd ~/some-repo
git commit --allow-empty -S -m "test: yubikey signed commit"
git verify-commit HEAD
git push
```

Confirm the commit shows **Verified** (green) on GitHub. (Tap the YubiKey when it signs — the touch policy from 5e is waiting for your finger; the commit will appear to hang until you tap.)

> **If it shows "Unverified" on GitHub** despite a valid local signature, the cause is almost always that `user.email` doesn't match both a UID on the key AND a verified email on your GitHub account. Check all three line up.

---

## 9. Switching between the two YubiKeys

GPG records a serial number per stub. When you physically swap to the other key, it may say `Please insert the card with serial number ...` because the stub still points at the previous key. Re-reading the inserted card rewrites the stub to its serial:

```bash
gpg-connect-agent "scd serialno" "learn --force" /bye
```

This is only needed **on each swap**, and only matters because the stub is serial-bound — the subkeys themselves are identical on both keys, so signatures from either verify against the same public key. With the nano living plugged in permanently and the 5C NFC only for travel/backup, swaps should be rare (a few times a year, not daily).

### Set up the `yk-switch` alias (one-time)

Add this to `~/.zshrc` so a swap is a single word instead of the full command:

```bash
alias yk-switch='gpg-connect-agent "scd serialno" "learn --force" /bye'
```

Reload the shell (`source ~/.zshrc` or open a new terminal). Then swapping is:

1. Remove the current YubiKey
2. Insert the other one
3. Run `yk-switch`

If `yk-switch` doesn't clear a stuck state, the heavier reset (restarts the agent — from drduh note #4) usually does:

```bash
pkill "gpg-agent|ssh-agent|pinentry"; gpg-connect-agent updatestartuptty /bye
```

> If you ever end up swapping *frequently* and the alias gets tiresome, the `remove-keygrips.sh` approach in the drduh guide deletes the serial-bound stub entirely so GPG accepts any card holding the right keys without prompting. More setup; only worth it for genuinely frequent swapping. Not needed for the nano-always-in pattern.

---

## 10. Update the .netrc.gpg (if still using it)

Re-encrypt the credentials file to the new key, and rotate the GitHub PAT to a fine-grained token while you're in there.

```bash
rm ~/.netrc.gpg
vi ~/.netrc                  # edit: put the NEW fine-grained PAT in here
gpg --encrypt --recipient "$KEYID" -o ~/.netrc.gpg ~/.netrc
rm ~/.netrc                  # remove the plaintext — it holds the token in clear
```

> ⚠️ **Use a keyid that uniquely identifies the NEW key, not the email, during the parallel period.** If you have TWO keys with the same `foo@bar.com` UID — an old key and the new one — `--recipient foo@bar.com` is **ambiguous**: GPG may encrypt to either, and "it worked" doesn't tell you which. If it picks the **old** key, you won't be able to decrypt `.netrc.gpg` after you retire that key. Use the new key's primary keyid (`$KEYID`) — GPG auto-routes to its `[E]` encryption subkey, and the keyid uniquely picks the new key. Encryption uses the *public* key, so no YubiKey needs to be inserted for the encrypt step; the card is only needed later to *decrypt*.

**Verify it encrypted to the NEW key:**

```bash
gpg --list-packets ~/.netrc.gpg | grep keyid
```

- keyid matches the new key's `[E]` subkey → correct. ✓
- keyid matches the OLD key's `[E]` subkey → re-encrypt with the explicit new keyid above before retiring the old key.

> Note: you encrypt *to* the primary keyid (`$KEYID`), but the packet records the `[E]` **subkey** id — because GPG routes encryption to the subkey. Both refer to the same (new) key; that's expected, not a mismatch.

> **Once the old key is retired** (section below) the duplicate-UID ambiguity is gone, and `--recipient foo@bar.com` resolves uniquely to the new key again — so you can go back to the simpler email form for anything you re-encrypt after that.

---

## Retiring an old YubiKey (do later, NOT today)

Run this only after the new keys have been working in parallel for a week or two and you're confident. Commands provided now for reference. **Two-part cleanup** — remove the key material AND the ownertrust entry, or you'll create a dangling "ultimately trusted key not found" ghost (like the `F7F32…` one cleaned during setup).

**1. Deregister the old key from services first (manual, before deleting anything):**
   - GitHub → Settings → SSH and GPG keys → remove the old GPG key. Past commits stay Verified via GitHub's persistent record.
   - Remove the old key as a registered security key / passkey anywhere it's used (GitHub, Google, Microsoft, etc.).
   - Confirm `.netrc.gpg` and any other encrypted files are encrypted to the NEW key (see step 10 verify) — anything still encrypted to the old key must be re-encrypted first, or you lose access to it.

**2. Investigate any unexpected subkeys before deleting:** List everything attached to the old key and confirm nothing is unaccounted for:
   ```bash
   gpg --list-keys --with-subkey-fingerprints <OLD_KEYID>
   gpg --list-secret-keys --with-subkey-fingerprints <OLD_KEYID>
   ```

**3. Remove the old key material from the keyring:**
   ```bash
   # secret first (the stubs / any on-disk secret), then public
   gpg --delete-secret-keys <OLD_KEYID>
   gpg --delete-keys <OLD_KEYID>
   ```

**4. Remove the old key's ownertrust entry (the part people forget):**
   ```bash
   gpg --export-ownertrust > /tmp/ot.txt
   # ownertrust uses the full fingerprint:
   grep -v '<OLD_FULL_FINGERPRINT>' /tmp/ot.txt > /tmp/ot-clean.txt
   rm ~/.gnupg/trustdb.gpg
   gpg --import-ownertrust < /tmp/ot-clean.txt
   gpg --check-trustdb        # should run clean, no "not found" note
   rm /tmp/ot*.txt
   ```
   (Remember: `--import-ownertrust` MERGES, it doesn't replace — deleting `trustdb.gpg` and rebuilding from the cleaned file is what actually removes the entry.)

**5. Physically retire the old YubiKey:** `ykman openpgp reset` wipes its OpenPGP applet. (Note: EUCLEAK affects this pre-5.7 key, so a reset doesn't fix that — but for disposal/repurposing it's clean.) Also reset/remove FIDO2 and PIV if you used them. Then store or dispose of the key.

---

## 11. Cleanup + storage

- [ ] Confirm the Certify passphrase, both keys' PINs, and KEYID are saved in 1Password.
- [ ] Delete the loose `keyinfo.txt` helper from the stick root (KEYID is in 1Password now):
      ```bash
      rm /Volumes/GPG-Backup/keyinfo.txt
      ```
- [ ] **Remove the `certify-pass.txt` plaintext from the stick root** — keep the passphrase only in 1Password, so stick + passphrase stay two separate factors (the backed-up Certify key then still needs the passphrase to use, even if the stick is unlocked):
      ```bash
      rm /Volumes/GPG-Backup/certify-pass.txt
      ```
      (Alternative if you prefer convenience over the two-factor split: leave it — the stick is APFS-encrypted, so an unlocked stick could then do everything.)
- [ ] The temp-dir working copy (`$GNUPGHOME` on internal disk, incl. its `certify-pass.txt`) is cleared on reboot. To zap it now: `rm -rf "$GNUPGHOME"`.
- [ ] Reboot (clears any remaining `/var/folders/.../T/` working dir + live Certify key).
- [ ] Eject the backup stick (`diskutil eject /Volumes/GPG-Backup`).
- [ ] Store the stick SEPARATELY from the YubiKeys (different bag while traveling; offline cold storage when home — NOT the Synology, it's networked).
- [ ] Ideally make a SECOND backup stick and store it in a different location.
- [ ] 1Password sanity check — saved: Certify passphrase, both keys' User/Admin PINs, stick password, KEYID. Note "KDF enabled" so future-me knows the PINs are KDF-derived.

> The backup directory (`gnupghome-backup/` minus the passphrase file) STAYS on the stick — that's your Certify-key backup for subkey renewal and provisioning more YubiKeys.

---

## Later (independent of all the above)

- **FIDO2 passkeys**: GitHub → Settings → Passkeys → add each YubiKey + Mac Touch ID as separate credentials. ~30s each. 5C NFC already has a FIDO2 PIN set; nano may prompt to set one.
- **Retire any old YubiKey**: see the full "Retiring an old YubiKey" section above (after step 10) — deregister everywhere, investigate any unexpected subkeys, remove key material AND ownertrust entry, then `ykman openpgp reset` / dispose. (Pre-5.7 firmware is EUCLEAK-affected.)
- **Subkey renewal** in ~2 years: mount the backup stick, restore from `/Volumes/GPG-Backup/gnupghome-backup` into a temp `$GNUPGHOME` (same as step 6), `--quick-set-expire` on the subkeys, re-export the public key, re-import it where you use it. Kill agents + eject the stick after (the agent-hygiene rule still applies). SSH keys don't need updating on remote hosts. See drduh "Updating keys".

---

## Troubleshooting

### `gpg --card-status` → "selecting card failed: Operation not supported by device"

The card is plugged in but GPG can't talk to it. Almost always **daemon contention**, not a hardware or key problem (your keys on the card are unaffected). Most common cause in this workflow: a **stale `scdaemon` bound to a dead temp `$GNUPGHOME`** is squatting on the card's USB interface.

Diagnose:

```bash
ps aux | grep -iE "scdaemon|ykman|yubico|pcsc" | grep -v grep
```

Look for a `scdaemon ... --homedir /var/folders/.../T/tmp.XXXX` line — that's a leftover from a provisioning session whose temp dir is gone. (Also quit Yubico Authenticator / any `ykman` process — they hold the card's CCID interface and block scdaemon.)

Fix — kill ALL scdaemons (not just the current homedir's), then retry:

```bash
gpgconf --kill all 2>/dev/null; pkill gpg-agent; pkill scdaemon; sleep 1
ps aux | grep -iE "scdaemon" | grep -v grep   # expect empty
gpg --card-status                              # should now return card details
```

> Why `gpgconf --kill all` alone isn't enough: it only reaps daemons for the *current* `$GNUPGHOME`. A scdaemon spawned earlier against a temp dir survives bound to that dead homedir and keeps the card. `pkill scdaemon` clears them regardless of homedir.

If that still fails after reseating the key (unplug, wait, replug, kill daemons, retry), it may be a driver-layer issue rather than contention. Then add the macOS workaround `disable-ccid` (makes scdaemon use PCSC instead of its internal CCID driver):

```bash
echo "disable-ccid" >> ~/.gnupg/scdaemon.conf
gpgconf --kill all; sleep 1; gpg --card-status
```

This is one of the few config additions worth making even on a defaults-only setup — it's a functional fix, not cosmetic. Only add it if the daemon-kill alone doesn't work.

### Swapping keys: "Please insert the card with serial number …"

Expected when switching between the two YubiKeys — the stub is serial-bound. Run `yk-switch` (the alias from step 7) to re-read the inserted card. See step 9.

### Commit signs locally but shows "Unverified" on GitHub

`user.email` must match BOTH a UID on the key AND a verified email on your GitHub account. Check all three line up (see step 8).

### `sec#` in `gpg -K` output

Expected and correct — the `#` means the secret primary (Certify) key is NOT in this keyring (it's offline on your backup stick). `sec#` + `ssb>` = "Certify offline, subkeys on card", which is the intended end state.
