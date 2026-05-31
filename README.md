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
  - Config Key Deriviation Function (KDF) to ensure PINs are hashed when crossing USB bus
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
| **KDF** | Key Deriviation Function. Makes the YubiKey hash the PIN on the host before sending it to the card, so the PIN isn't transmitted in plaintext. |
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

## Phase I - Setup, Generate Keys

### Step 1. Setup Isolated working directory

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

### Step 2. Generate strong Certify passphrase

Once in the temp directory, generate the passphrase directly to a `certify-pass.txt` file in `$GNUPGHOME`:

```bash
LC_ALL=C tr -dc "A-Z2-9" < /dev/urandom | tr -d "IOUS5" | \
    fold -w 4 | paste -sd - - | head -c 29 > "$GNUPGHOME/certify-pass.txt"
```

This passphrase is needed later for generating keys.

### Step 3. Setup Variables

Before running commands, set the passfile location and set a few variables for re-use. **Replace fictional name and email below with your information.**

```bash
PASSFILE="$GNUPGHOME/certify-pass.txt"
IDENTITY="Anya Forger <anya.forger@spyxfamily.com>" # replace with your information
EXPIRATION=2y                                       # 2 year expiry
```

> [!TIP]
> The email address used in signed commits will appear **publicly** on GitHub.com. I recommend using a professional email address over a personal email.

### Step 4. Generate keys on Mac

> [!IMPORTANT]
> Before adopting this setup, check that every system you sign or authenticate to accepts Curve25519 keys — some older services still require RSA.

#### Curve25519 Cryptography

My [old setup (2018)](https://julie.io/blog/setup-git-multiple-gpg-and-yubikeys) used [RSA cryptography](https://en.wikipedia.org/wiki/RSA_cryptosystem), which has wider support but is older. 

For this 2026 setup I chose the faster and more modern [Curve25519-family cryptography](https://en.wikipedia.org/wiki/Curve25519) with EdDSA for certify/sign/auth and ECDH for encryption (EdDSA can't encrypt):

| Key | Function | Algorithm | Curve |
|-----|----------|-----------|-------|
| `[C]` Primary Key | Certification | EdDSA | `ed25519` |
| `[S]` Subkey | Signature | EdDSA | `ed25519` |
| `[A]` Subkey | Authentication | EdDSA | `ed25519` |
| `[E]` Subkey | Encryption | ECDH | `cv25519` |

#### 4A. Generate Certify key 

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

#### 4B. Generate SIGN subkey 

```bash
# Signing subkey
gpg --batch --pinentry-mode=loopback --passphrase-file "$PASSFILE" \
    --quick-add-key "$KEYFP" ed25519 sign "$EXPIRATION"
```

#### 4C. Generate AUTH subkey 

```bash
# Authentication subkey
gpg --batch --pinentry-mode=loopback --passphrase-file "$PASSFILE" \
    --quick-add-key "$KEYFP" ed25519 auth "$EXPIRATION"
```

#### 4D. Generate ENCRYPT subkey 

```bash
# Encryption subkey — NOTE: cv25519, not ed25519
gpg --batch --pinentry-mode=loopback --passphrase-file "$PASSFILE" \
    --quick-add-key "$KEYFP" cv25519 encrypt "$EXPIRATION"
```

#### 4E. Verify keys were generated

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

#### 4F. Export the public key

Export the public key so we can [upload it to GitHub for signature verification](https://docs.github.com/en/authentication/managing-commit-signature-verification/adding-a-gpg-key-to-your-github-account).

```bash
gpg --armor --export "$KEYID" > "$GNUPGHOME/$KEYID-pub.asc"
```

### Step 5. Back up Keyring to USB Stick

To provision a **second** YubiKey, we need to backup the existing `GNUPGHOME`, which has our private keys. After configuring the first YubiKey, the folder will only contain _stubs_ to the subkeys. 

> [!IMPORTANT]
> **USB Stick Name**: I named my USB drive `GPG-Backup` and thus it is mounted at `/Volumes/GPG-Backup/`. Adjust the path to match your setup.

#### 5A. Move Certify Passphrase to Password Manager

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

#### 5B. Copy the non-secret info to USB stick

Copy the public key and info text file that contains our key ID and key fingerprint to the USB stick root:

```bash
mv "$GNUPGHOME/$KEYID-pub.asc"  /Volumes/GPG-Backup/
mv "$GNUPGHOME/keyinfo.txt"  /Volumes/GPG-Backup/
```

#### 5C. Copy the keyring to USB stick

Now copy the keyring, i.e. `$GNUPGHOME` to the USB Stick.

```bash
cp -av "$GNUPGHOME" /Volumes/GPG-Backup/
```

Now, we'll rename the `tmp.xxxxx` folder to a human-friendly `gnupghome-backup` name.

```bash
mv /Volumes/GPG-Backup/"$(basename "$GNUPGHOME")" /Volumes/GPG-Backup/gnupghome-backup
```

#### USB Directory Structure

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

#### 5D. Verify the backup before trusting it

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

#### 5E. Eject the stick

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

## Phase II - Configure Yubikey #1 

This phase configures my daily driver, the [YubiKey 5C Nano](https://www.yubico.com/de/product/yubikey-5-series/yubikey-5c-nano/).

### OpenGPG Requirements

> [!TIP]
> A YubiKey has multiple applets and thus multiple PINs. This section refers to the OpenGPG applet. Do not confuse with PIV.

Before continuing note the following requirements. We're using the default PINs for setup, so we can pass it inline in bash.

- [ ] OpenGPG applet is in factory state
- [ ] OpenGPG User PIN is default `123456` 
- [ ] OpenGPG Admin PIN is default `12345678`

During setup, you'll change the PINs.

### Step 6. Configure YubiKey (nano)

Plug in the nano. Confirm it's visible:

```bash
gpg --card-status
```

#### 6A. Enable KDF First

The Key Derivation Function (KDF) hashes PINs so they never cross the USB bus in plaintext. Enable KDF **FIRST**, before changing PINs or moving keys:

```bash
gpg --command-fd=0 --pinentry-mode=loopback --card-edit <<EOF
admin
kdf-setup
12345678
EOF
```

##### Troubleshooting

If you get `Conditions of use not satisfied`, the card already had PINs/keys changed — KDF must be first on a fresh applet by running `ykman openpgp reset`.

#### 6B. Change PINs from defaults

Now pick secure PINs that meet the requirements:

- User PIN: min. 6 digits
- Admin PIN: min. 8 digits

> [!NOTE]
> The rest of the guide mostly uses manual interactive steps with `gpg --card-edit`, which avoids secrets in the bash history and better for intentionality and learning.

Run the edit command

```bash
gpg --card-edit
```

Then at the `gpg/card>` prompt, type the subcommands, e.g. `admin` and the corresponding options, e.g. `2`

```bash
gpg/card>
admin
passwd
1  # change User PIN  (default 123456)
3  # change Admin PIN (default 12345678)
q
quit
```

Save your chosen PINs in your password manager.

#### 6C. Set card attributes: login and name

Do this **interactively** — these are admin-authorized writes, so the card needs the **Admin PIN**. A heredoc with `--pinentry-mode=loopback` supplies no PIN and fails with `error setting login data: Bad PIN`. Interactive prompts for the Admin PIN properly (via pinentry) and keeps the PIN off the command line. Both attributes are set in the same session:

```bash
gpg --edit-card
```

Then at the `gpg/card>` prompt, type:

```bash
gpg/card>
admin        # to run Admin operations
login        # Type your identity, e.g.
             #   Your Name <foo@bar.com>
name         # Prompts for Surname then Given name SEPARATELY:
             #   Surname:    Doe
             #   Given name: Jane      
lang         # e.g. "en"
salutation   # e.g. "Ms."
quit
```

When prompted, enter your **Admin PIN** to authorize and save the changes.

##### Troubleshooting

If you get `Bad PIN`, double check and do not brute force. You only have **3 attempts**, after which you need to reset the OpenGPG applet. To check how many attempts are remining, run `ykman openpgp info.

#### 6D. Transfer subkeys to the YubiKey

First confirm we're still in our temp directory, e.g. `/var/folders/.../T/tmp.XXXXX`

```bash
echo "$GNUPGHOME"                
```

> [!CAUTION]
> The `keytocard` method **deletes** on-disk keys after moving them to the YubiKey. It is not possible to recover private keys from YubiKeys. Only proceed after creating keyring backup in step 5.

Before we continue, note that each `keytocard` operation, you will need **two (2) secrets**

- **Certify passphrase** - look up in your password manager
- **Admin PIN** - set in step 6B above

GPG might cache the passphrase. But when I moved my keys, I had to type the long `XXXX-XXXX` each time.

Now, we'll make the changes _interactively_.

```bash
gpg -K
gpg --edit-key "$KEYID"
```

First, **confirm the subkey order** by typing `list`

```bash
gpg/card>
list
```

which shows is zero-indexed key order:

```
sec  ...usage: C    <- position 0 (primary)
ssb  ...usage: S    <- position 1 (signing)
ssb  ...usage: A    <- position 2 (authentication)
ssb  ...usage: E    <- position 3 (encryption)
```

**Selecting (and deselecting) Keys**

Now, we'll move the keys one at a time, which requires selection **_and_** deselection:

- Type `key 1` to select key 1 (e.g. signing)
- Run commands
- Type `key 1` again to deselect the key. Note the ❗️ below.

**Finally, move keys with `keytocard` command**

> [!WARNING]
> Selection is shown by an asterisk `*` next to the line (e.g. `ssb*`). For each `keytocard`, run `list` and confirm that exactly **one** subkey shows the asterisk `*`.

```bash
key 1          # Select signing [S]
list           # Confirm ONLY the usage:S line shows ssb*
keytocard      # Choose slot 1 (Signature key)   <- enter Admin PIN when prompted
key 1          # ❗️Deselect [S]

key 2          # Select authentication [A]
list           # Confirm ONLY the usage:A line shows ssb*
keytocard      # Choose slot 3 (Authentication key)
key 2          # ❗️Deselect [A]

key 3          # Select encryption [E]
list           # Confirm ONLY the usage:E line shows ssb*
keytocard      # Choose slot 2 (Encryption key)
key 3          # ❗️Deselect [E]

save
quit
```

After typing `save` and `quit`, we can verify subkeys are now moved by running:

```bash
gpg -K
```

which should now show three `ssb>` with the greater-than `>` marker, indicating the key is not on-disk and instead on the YubiKey

```
ssb>   ed25519... [S] [expires: ...]
ssb>   ed25519... [A] [expires: ...]
ssb>   cv25519... [E] [expires: ...]
```

#### 6E. Require touch per operation (Recommended)

Because the YubiKey nano lives plugged-in to my computer (susceptible to malware with cached PIN), I also want to **require physically touch** on operations.

This way, the PIN may be cached, but each operation still needs a touch.

**Touch Modes**

YubiKey offers 4 different modes:

| Mode | Behavior | 
|------|----------|
| `off` | No touch; cached PIN is enough |
| `on` | Touch required for **every** operation |
| `cached` | Touch required, then a ~15s window where further ops need no re-touch | 
| `fixed` | Like `on`, but **can't be changed** without wiping the applet |

I am choosing `cached` because I don't want to have to touch it 10 times if I am rebasing 10 commits. Instead I'll leverage the 15 second cache window.

```bash
ykman openpgp keys set-touch sig cached   # signing: frequent → cached (one tap per burst)
ykman openpgp keys set-touch aut cached   # SSH auth: occasional → cached
ykman openpgp keys set-touch dec cached   # decryption: frequent (.netrc) → cached
```

When prompted, enter your **Admin PIN**.

#### 6F. Disable Yubico OTP (Optional)

Because I don't use Yubico OTP (outputs long `cccccc…`-prefixed string) and I occassionally accidentally brush the gold contact of the nano that lives plugged in, I'm going to disable it:

```bash
ykman config usb --disable OTP
```

The first YubiKey (nano) is fully provisioned. 

#### 6G. Eject YubiKey

Before we eject the YubiKey, you can save the serial number. Please note that although both YubiKeys will have identical GPG keys, they will have separate serial numbers.

```bash
gpg --card-status | grep -i "serial"
```

Now we need to kill all agents and daemons with open sockets to our temp directory or card interface.

```bash
gpgconf --kill all 2>/dev/null
pkill gpg-agent
pkill scdaemon
sleep 1
```

Now confirm there are no `scdaemon`s left.

```bash
ps aux | grep -iE "scdaemon" | grep -v grep
```

It should show no processes. Now remove YubiKey #1.

---

## Phase III. Restore Backup

Since the `keytocard` operation removed the private keys from local disk, we need to restore the `GNUPGHOME` contents from the USB stick backup.

### Step 7 - Restore Backup

#### 7A. Insert USB Stick

Insert the USB Stick. Enter the encryption password (set outside this guide) and confirm you have the backup folder `gnupghome-backup`

```bash
ls /Volumes/GPG-Backup/         
```

#### 7B. Delete existing temp directory

To ensure a clean restore, we will delete the previous temp directory.

First, confirm that your `$GNUPGHOME` still points to your temp directory. Run

```bash
echo "$GNUPGHOME"
```

which should show something like `/var/folders/tc/..../T/tmp.aB3kPzQ9rN`.

Then change directories, e.g. go to your home directory and remove the existing temp directory

```bash
cd ~
rm -rf "$GNUPGHOME"
```

#### 7C. Re-create temp directory

Now we'll create a fresh `GNUPGHOME`

```bash
export GNUPGHOME=$(mktemp -d)
```

and copy the backup and keyinfo from the USB stick to it

```bash
cp -av /Volumes/GPG-Backup/gnupghome-backup/* "$GNUPGHOME"/
cp /Volumes/GPG-Backup/keyinfo.txt "$GNUPGHOME/keyinfo.txt"
```

Now confirm verify our backup by running

```bash
gpg -K
```

and all 3 subkeys should show `ssb` _without_ any greater-than `>` symbol.

#### 7D. Kill Agents and Eject USB Stick

Run the same steps from [5E](#5e-eject-the-stick) above. See step for full instructions.

```bash
gpgconf --kill all 2>/dev/null; pkill gpg-agent; sleep 1
diskutil eject /Volumes/GPG-Backup
```

---

## Phase IV. Configure YubiKey #2

### Step 8. Configure YubiKey #2 (NFC)

#### Confirm Variables 

Confirm you still have variables from [step 3](#step-3-setup-variables)

- `KEYID`, which you can find in `$GNUPGHOME/keyinfo.txt` 
- Certify Passphrase (`XXXX-XXXX-…`), which is in your password manager

#### Follow Steps above

- [6A](#6a-enable-kdf-first) - Enable KDF first
- [6B](#6b-change-pins-from-defaults) - Change PINs from defaults 
- [6C](#6c-set-card-attributes-login-and-name) - Set login and name attributes
- [6D](#6d-transfer-subkeys-to-the-yubikey) - Transfer subkeys to the YubiKey
- [6E](#6e-require-touch-per-operation-recommended) - Require touch per operation (optional, but recommended)
- [6F](#6f-disable-yubico-otp-optional) - Disable YubiCo OTP
- [6G](#6g-verify-and-eject-yubikey) Verify and Eject YubiKey

Now both keys carry identical subkeys. Setup Finished! 🎉
