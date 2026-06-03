---
name: bge-encryption
description: >-
  Encrypt and decrypt files and folders from the command line with the `bge` tool
  (the DotBGE CLI), using the open BGE v3 `.bge` format (RSA-4096 + AES-256-GCM).
  Use this when the user wants to encrypt or decrypt a file or folder, create or
  open a `.bge` file, password-protect a file, generate an RSA key pair for
  encryption, encrypt something "for" a specific recipient/public key or a saved
  contact by name, manage saved contacts and identities, inspect a
  `.bge` file without decrypting it, or turn a public key into a shareable
  `.bgekey` identity card. Two modes: RSA identity (a recipient's key) and
  password (a passphrase).
---

# bge — file encryption from the command line

`bge` encrypts and decrypts files and folders in the open **BGE v3** `.bge` format
(RSA-4096-OAEP + AES-256-GCM). It is the terminal companion to the DotBGE macOS/iOS
apps: `.bge` files and `.bgekey` identity cards round-trip with them.

## 0. Before anything: is it installed?

```bash
bge --version          # prints e.g. 0.1.0 if installed
```

If `bge` is not found, tell the user how to install it and stop — do not try to
build it yourself:

- **macOS (Homebrew):** `brew install --cask dotbge/tap/bge`
- **macOS / Linux:** download a build from https://github.com/dotbge/bge-cli/releases/latest

## 1. Rules for running bge non-interactively (read this first)

You are an agent without a TTY. Follow these or commands will hang or leak secrets:

- **Never use bare `-p` / `--password`.** It opens an interactive passphrase prompt
  (`getpass`) that will hang forever in a non-interactive shell. For password mode
  **always use `--password-stdin`** and pipe the passphrase in.
- **Never put a passphrase as a literal argument** (e.g. don't do `... <<< "$PW"` onto
  a flag, and never type the password in the command line). It would leak into the
  process list and shell history. Pipe it from a variable or file via stdin instead:
  ```bash
  printf %s "$BGE_PASSPHRASE" | bge encrypt notes.txt --password-stdin
  bge encrypt notes.txt --password-stdin < passphrase.txt
  ```
  (`--password-stdin` strips exactly one trailing newline, so `echo` and `printf` agree.)
- **Prefer RSA key mode for automation** — pass a key file with `-r`/`-k`; no prompt needed.
- **bge never overwrites by default.** If the output exists it fails with exit `64`
  ("Output already exists … use --force"). Pass `-f`/`--force` only when overwriting is intended.
- **Confirm with the user before**: overwriting files (`--force`), generating new keys,
  or writing decrypted (plaintext) secrets to disk. When you only need to feed plaintext
  into another tool, decrypt to stdout (`-o -`) and pipe it instead of leaving it on disk.

## 2. The two modes

| Mode | Encrypt with | Decrypt with | Who can open it |
|------|--------------|--------------|-----------------|
| **RSA identity** | a recipient's **public** key — `-r alice.pub.pem`, `-r alice.bgekey`, or `-r <saved-name>` | matching **private** key (`-k alice.pem`) | only the private-key holder |
| **Password** | a passphrase (`--password-stdin`) | the same passphrase (`--password-stdin`) | anyone with the passphrase |

`decrypt` needs exactly one mode. `encrypt` too — **except** that with no recipient and no
password it defaults to sealing to your own **active identity** (see the address book in §3).
Not sure which mode a `.bge` uses? Run `bge inspect <file>` first (no key needed).

## 3. Commands & recipes

`enc` and `dec` are aliases for `encrypt` and `decrypt`. Every command takes `-h`.

### Generate an RSA key pair — `keygen`
```bash
bge keygen -o alice          # → alice.pem (private, chmod 600) + alice.pub.pem (public)
bge keygen                   # default basename → bge_key.pem + bge_key.pub.pem
```
- Only flag is `-o/--out <basename>`. There is **no `--force`** — keygen refuses to
  overwrite an existing key file; rename/remove it or pick a different `-o`.
- Keep the `.pem` (private) secret. Share the `.pub.pem` (or a `.bgekey` card). Prints
  a `key ID` — the fingerprint that says which private key can open a file.

### Encrypt — `encrypt` / `enc`
```bash
# RSA: only the holder of alice.pem can decrypt
bge encrypt report.pdf -r alice.pub.pem            # → report.pdf.bge
bge encrypt report.pdf -r Alice                    # to a SAVED CONTACT by name (see below)
bge encrypt report.pdf                             # no -r → to your own active identity
bge encrypt report.pdf -r alice.bgekey             # recipient given as an identity card
bge encrypt report.pdf -r alice.pub.pem -o out.bge # choose the output path

# Password (non-interactive — pipe the passphrase)
printf %s "$BGE_PASSPHRASE" | bge encrypt report.pdf --password-stdin   # → report.pdf.bge
```
Flags: `-r/--recipient <pem|.bgekey|saved-name>`, `--self`/`-s`, `--password-stdin`,
`-o/--output <path|->`, `-a/--archive` (folders), `-f/--force`. **Encrypt reads its input
from a file/dir, not stdin** (it records the input size up front), but can write the `.bge`
to stdout with `-o -`.

### The address book — encrypt by name (handles "encrypt … for <name>")

`bge` keeps a **public-key** address book under `~/.bge/` so you can encrypt to people by
name instead of locating key files. **Use this whenever the user says "encrypt X for <name>".**

```bash
bge contact ls --json           # discover saved contacts → [{name, keyId, source}]
bge encrypt report.pdf -r Info  # encrypt to a saved contact (name is case-INsensitive)
```

How `-r <X>` resolves: an existing **file path** wins (`./info`, `info.pub.pem`); otherwise
`X` is matched in the address book by **name or Key ID**, case-insensitively. `bge` does the
lookup — don't read `~/.bge/` yourself.

**When the user says "encrypt X for `<name>`":**
1. `bge contact ls --json` — is `<name>` (or a clear match) there?
2. Yes → `bge encrypt X -r <name>`.
3. No → ask the user for that person's public key or `.bgekey`, then
   `bge contact add <pem|.bgekey> -n <name>` and encrypt. (A `.bgekey` carries its own name.)
   **Never invent or guess a key.** An unknown name fails with exit `64` and points you here.

**Encrypt to yourself:** with no `-r`/`-p`, `bge encrypt X` (or `-s`/`--self`) seals to your
**active identity** — good for "encrypt this for me / for backup".

Manage the book (public keys only — it never holds a private key):
```bash
bge contact add alice.bgekey       # add/import a friend (a PEM needs -n; a .bgekey has a name)
bge contact show <name|keyId>      # one contact;  bge contact rm <name|keyId> to delete
bge identity add me.pub.pem -n Me  # register one of your own keys
bge identity use <name>            # set your active identity (bge identity ls to list)
```
If `bge contact`/`bge identity` reports an **unknown subcommand**, the installed `bge`
predates the address book — fall back to key-file paths (`-r alice.pub.pem`).

### Decrypt — `decrypt` / `dec`
```bash
bge decrypt report.pdf.bge -k alice.pem                        # RSA → report.pdf
printf %s "$BGE_PASSPHRASE" | bge decrypt report.pdf.bge --password-stdin   # password → report.pdf
bge decrypt report.pdf.bge -k alice.pem -o restored.pdf        # choose output path
```
Flags: `-k/--key <priv.pem>`, `--password-stdin`, `-o/--output <path|->`,
`-x/--extract` (archives), `-f/--force`. Default output strips a trailing `.bge`.
**Decrypting to stdout (`-o -`) buffers the plaintext in memory (~200 MB cap)** — for
large files, decrypt to a real file (`-o path`), which streams in constant memory.

The address book is **encrypt-only** — it holds no private keys, so it can't help you
decrypt. You need the recipient's **private key file** (`-k <path>`); ask the user for it
rather than hunting for `.pem` files on disk.

### Folders & bundling several files
When the input is a **directory**, `bge` processes it two ways:
```bash
# Structured (default): mirror the tree, one .bge per file
bge encrypt myfolder -r alice.pub.pem      # → myfolder_bge/  (a.txt → a.txt.bge, recursively)
bge decrypt myfolder_bge -k alice.pem      # → myfolder/      (trailing _bge stripped)

# Archive (-a): zip the whole folder, then encrypt to ONE .bge
bge encrypt myfolder -a -r alice.pub.pem   # → myfolder.zip.bge
bge decrypt myfolder.zip.bge -k alice.pem -x   # decrypt + unzip → myfolder/
```
Use **structured** to browse/sync/decrypt files individually; use **archive** (`-a`) for
one self-contained blob that also hides the file names and count. In folder mode,
existing outputs are **skipped** unless `-f/--force`.

**Package/encrypt a *selection* of files** ("zip and encrypt these photos") — pass the
files directly with `-a`; **don't stage them in a temp folder yourself**, `bge` bundles them:
```bash
bge encrypt a.heic b.heic c.heic -a -r Alice -o photos.bge   # files → one photos.bge
bge decrypt photos.bge -k alice.pem -x                       # → the files back
```
Without `-o`, the output defaults to `archive.zip.bge` in the current directory. Multiple
inputs **require** `-a` (otherwise exit `64`); a directory among them is included whole.

### Inspect a `.bge` (no key needed) — `inspect`
```bash
bge inspect report.pdf.bge               # mode, key ID (RSA) or KDF+iterations (password), size
bge inspect report.pdf.bge -k alice.pem  # + decrypts ONLY the metadata block → original name, type, dims
printf %s "$BGE_PASSPHRASE" | bge inspect report.pdf.bge --password-stdin   # same for password files
```
Keyless inspect reads only the public header (the original filename/type/size are
encrypted). Adding a credential decrypts just the small metadata block, never the content.
If the recipient's key is in your address book, inspect also prints `Encrypted for: <name>`
(add `--json` for `{mode, keyId, encryptedFor, isSelf, …}`) — a keyless way to see **who a
`.bge` is for**, hence which private key opens it. Run it before decrypting.

### Make a `.bgekey` identity card — `card`
```bash
bge card alice.pub.pem -n "Alice"        # → Alice.bgekey  (a contact card the DotBGE apps import)
```
Wraps a **public** key + display name into the `.bgekey` the DotBGE apps import as a
contact. No private key is read. Others can then `bge encrypt -r Alice.bgekey` to you.
Input may be a PEM or an existing `.bgekey` (to rename it). `-o/--output <path|->`, `-f/--force`.

### Pipelines (stdin/stdout with `-`)
```bash
# Decrypt from stdin and pipe straight into tar — no plaintext on disk
bge decrypt - -k alice.pem < archive.tgz.bge | tar xzf -

# Encrypt a stream to stdout
bge encrypt archive.tgz -r alice.pub.pem -o - > archive.tgz.bge
```
Status lines go to **stderr**, so stdout carries only data. You **cannot** combine input
`-` with `--password-stdin` (both read stdin) — use a key, or `--password-stdin` with a
file input.

## 4. Exit codes — branch on these, don't parse messages

| Code | Meaning | Typical cause |
|-----:|---------|---------------|
| `0`  | Success | |
| `64` | Usage error | bad flags, no/2 modes given, **output exists without `--force`** |
| `65` | Data error | not a `.bge` file, corrupt/unsupported file, malformed key PEM |
| `66` | No input | input or key file not found |
| `73` | Can't create output | output path not writable |
| `77` | **Can't decrypt** | wrong key, wrong password, or tampered ciphertext |
| `70` | Internal error | unexpected failure |

```bash
bge decrypt in.bge -k k.pem -o out
case $? in
  0)  echo "ok" ;;
  77) echo "wrong key or password" ;;
  66) echo "input or key missing" ;;
esac
```

## 5. Scope — what bge does NOT do

`bge` is file/folder encryption with file-based keys only. It does **not** do: system
Keychain storage, hardware tokens (YubiKey/PIV), QR codes, iCloud sync, or decrypting
legacy v1/v2 files — those live in the DotBGE macOS/iOS apps, not the CLI. The `.bge`
format is open and documented at https://github.com/dotbge/bge-format.
