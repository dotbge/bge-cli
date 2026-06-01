# bge

**Encrypt files and folders with someone's key or a password.**
An open `.bge` format — RSA-4096 + AES-256-GCM, no servers, no lock-in.

`bge` is the command-line companion to the [dotbge](https://dotbge.com) apps for macOS and
iOS. Same files, same keys, same open format — now in your terminal and your scripts.

## Install (macOS)

**Homebrew** (recommended):

```bash
brew install --cask dotbge/tap/bge
```

Or download a signed, notarized build from [**Releases**](../../releases/latest):

- **`bge-<version>.pkg`** — double-click to install `bge` (+ man page) into `/usr/local/bin`.
- **`bge-<version>-macos-universal.zip`** — the standalone universal binary.

Either way it's a universal binary (Apple Silicon **and** Intel), signed with an Apple
Developer ID and notarized, so it runs without Gatekeeper warnings.

```bash
bge --version
man bge
```

## Quick start

```bash
# 1. Make an identity key pair
bge keygen -o alice                        # → alice.pem (private) + alice.pub.pem

# 2. Encrypt for someone — only their private key can open it
bge encrypt report.pdf -r alice.pub.pem    # → report.pdf.bge

# 3. Decrypt with your private key
bge decrypt report.pdf.bge -k alice.pem    # → report.pdf

# Prefer a password? Skip the keys entirely
bge encrypt report.pdf -p                  # prompts for a passphrase

# Peek at a .bge without decrypting it (add -k/-p to reveal the original filename & type)
bge inspect report.pdf.bge

# Hand out your public key as a contact card the dotbge apps import
bge card alice.pub.pem -n "Alice"          # → Alice.bgekey
```

Run `bge -h`, or `bge <command> -h`, for every option. `enc` / `dec` are aliases for
`encrypt` / `decrypt`.

## What it does

- **Two ways to lock a file** — to a person (RSA-4096 identity) or a passphrase
  (PBKDF2-SHA512). Either way, content is sealed with AES-256-GCM.
- **Files _and_ folders** — a single file, a mirrored tree of `.bge` files, or one zipped
  archive (`-a`).
- **Interops with the apps** — `.bge` files and `.bgekey` identity cards round-trip with the
  dotbge apps for macOS & iOS.
- **Built for scripts** — stdin/stdout piping (`-`), `--password-stdin`, and distinct
  [`sysexits`](https://man.openbsd.org/sysexits.3) exit codes.

## The format is open

`bge` writes the open **BGE v3** format — documented, with test vectors, at
[**dotbge/bge-format**](https://github.com/dotbge/bge-format). No proprietary container, no
server, nothing to lock you in.

## Verify a download

```bash
shasum -a 256 -c SHA256SUMS
```

---

© dotbge · [dotbge.com](https://dotbge.com)
