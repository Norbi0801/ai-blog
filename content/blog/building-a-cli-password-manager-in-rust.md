+++
title = "Building a CLI password manager in Rust"
date = 2025-06-18
description = "A from-scratch password manager in around 300 lines of Rust: AES-256-GCM, Argon2 key derivation, encrypted SQLite, clipboard auto-clear, and zeroized memory."

[taxonomies]
tags = ["rust", "security", "cryptography", "cli"]
+++

The boring way to learn applied crypto is to read papers. The fun way is to build something that, if you got it wrong, would leak your actual passwords. A CLI password manager is the smallest project that forces you to make every interesting decision: how to turn a master password into a key, which AEAD to use, how to store a nonce, what to do with the plaintext after you decrypt it, and how the user gets the password into their browser without the whole laptop seeing it.

This post walks through a working `pm` binary in roughly 300 lines of Rust. AES-256-GCM for the symmetric primitive, Argon2id for key derivation, SQLite as the container, `arboard` for clipboard with timed clear, and `zeroize` to scrub keys and plaintexts on drop. If you are not familiar with the underlying hash families, I covered them in [Cryptographic hashing in Rust](/blog/cryptographic-hashing-in-rust-sha-blake-bcrypt-argon2/), and the code below assumes you are comfortable with the Argon2 PHC format.

<!-- more -->

## Threat model first

You cannot build a password manager without writing down what you are defending against. Every crypto choice below maps to a specific attacker.

The vault file is on disk. An attacker who steals the laptop, exfiltrates the home directory through a backup leak, or pulls it off a stolen Time Machine drive should see only ciphertext. They get the file, the salt, the nonce, the ciphertext, and the parameter metadata. They do not get the master password, and they do not get the running process memory.

The attacker is **offline** once they have the file. They can throw a million GPUs at guessing the master password. The only thing standing between them and the plaintext is how slow and memory-hard our key derivation is. SHA-256 of the master password would last about three seconds against a single RTX 4090. Argon2id with `m=64 MiB, t=3, p=4` will take that same GPU longer than the heat death of the project's relevance, assuming the user picked a non-trivial master password.

The attacker is **not** assumed to have a kernel keylogger or live process memory access. If they do, no userspace password manager can save you. KeePassXC, Bitwarden, and 1Password all give up at that boundary too. What we *can* do is reduce the window: do not keep the master key, the derived key, or decrypted entries in memory longer than needed, and zero them on drop. That is what `zeroize` is for. It does not prevent live memory dumps, but it shrinks the time interval in which one would find anything useful, and it stops accidental swap-to-disk leaks from leaving a derived key in `/swapfile` for months.

## The crate list

```toml
[dependencies]
aes-gcm = "0.10"
argon2 = "0.5"
rand = "0.8"
rand_core = { version = "0.6", features = ["std"] }
rusqlite = { version = "0.31", features = ["bundled"] }
arboard = "3.4"
zeroize = { version = "1.7", features = ["derive"] }
rpassword = "7.3"
clap = { version = "4.5", features = ["derive"] }
anyhow = "1"
```

A few notes on the picks. `aes-gcm` from RustCrypto is the same crate `rustls` reaches for when AES-NI is available, so we get hardware acceleration on every modern x86-64 and AArch64 CPU. `rusqlite` with the `bundled` feature ships its own SQLite, so the binary does not depend on the system library and the user gets the same behavior on macOS, Linux, and Windows. `rpassword` reads the master password without echoing it to the terminal (it does the `termios` ICANON dance for you). `arboard` is the only cross-platform clipboard crate that works on Wayland, X11, macOS, and Windows without bringing in GTK.

## Deriving a key from the master password

The master password is high-entropy by user standards (one-line memorable phrase) but extremely low-entropy by AES standards. We need to stretch it into a 32-byte key for AES-256-GCM. Argon2id does that, and salts the input so two users with the same master password produce different keys.

```rust
use argon2::{Argon2, Algorithm, Version, Params};
use zeroize::Zeroizing;

fn derive_key(master: &[u8], salt: &[u8]) -> anyhow::Result<Zeroizing<[u8; 32]>> {
    let params = Params::new(64 * 1024, 3, 4, Some(32))
        .map_err(|e| anyhow::anyhow!("bad argon2 params: {e}"))?;
    let argon2 = Argon2::new(Algorithm::Argon2id, Version::V0x13, params);

    let mut key = Zeroizing::new([0u8; 32]);
    argon2
        .hash_password_into(master, salt, key.as_mut_slice())
        .map_err(|e| anyhow::anyhow!("argon2 derivation failed: {e}"))?;
    Ok(key)
}
```

`Zeroizing<[u8; 32]>` is the important part. When the variable goes out of scope, `Drop` writes zeros over those 32 bytes before the memory is reused. Without it, the derived key sits in whatever heap or stack slot the allocator hands out next, and any later panic that prints a `Vec<u8>` could conceivably surface it. The `zeroize` crate uses [`core::ptr::write_volatile`](https://doc.rust-lang.org/std/ptr/fn.write_volatile.html) so the writes cannot be optimized out by LLVM, which is the failure mode of a naive `key.fill(0)`.

The parameters - 64 MiB memory, 3 iterations, 4 lanes, 32-byte output - are above the OWASP minimum and target around 250 ms on a 2024-era laptop. Tune them for your slowest target machine; the entire reason to use Argon2 is that the cost scales with hardware, so picking a weak preset just to be fast on a Raspberry Pi defeats the point.

The salt is generated once at vault creation and stored in a header alongside the ciphertext. It does not need to be secret, only unique per vault. 16 random bytes is the standard.

## Encrypting an entry

AES-256-GCM is an AEAD: it encrypts and authenticates in one shot. Authentication matters here because without it an attacker who has write access to the file (a malicious sync service, a misconfigured backup) could flip bits in the ciphertext to corrupt entries. With GCM, any tampering causes decryption to fail loudly.

```rust
use aes_gcm::{Aes256Gcm, Key, Nonce};
use aes_gcm::aead::{Aead, KeyInit};
use rand::RngCore;
use zeroize::Zeroizing;

const NONCE_LEN: usize = 12;

fn encrypt(key: &[u8; 32], plaintext: &[u8]) -> anyhow::Result<Vec<u8>> {
    let cipher = Aes256Gcm::new(Key::<Aes256Gcm>::from_slice(key));
    let mut nonce = [0u8; NONCE_LEN];
    rand::thread_rng().fill_bytes(&mut nonce);

    let ciphertext = cipher
        .encrypt(Nonce::from_slice(&nonce), plaintext)
        .map_err(|_| anyhow::anyhow!("encryption failed"))?;

    // Layout: nonce || ciphertext || tag (tag is appended by aes-gcm)
    let mut out = Vec::with_capacity(NONCE_LEN + ciphertext.len());
    out.extend_from_slice(&nonce);
    out.extend_from_slice(&ciphertext);
    Ok(out)
}

fn decrypt(key: &[u8; 32], blob: &[u8]) -> anyhow::Result<Zeroizing<Vec<u8>>> {
    if blob.len() < NONCE_LEN + 16 {
        anyhow::bail!("ciphertext too short");
    }
    let (nonce, ct) = blob.split_at(NONCE_LEN);
    let cipher = Aes256Gcm::new(Key::<Aes256Gcm>::from_slice(key));
    let pt = cipher
        .decrypt(Nonce::from_slice(nonce), ct)
        .map_err(|_| anyhow::anyhow!("decryption failed (wrong password or corrupted vault)"))?;
    Ok(Zeroizing::new(pt))
}
```

The single most important rule of GCM: **never reuse a nonce with the same key**. If you do, GCM's confidentiality and authenticity both collapse. Two ciphertexts under the same (key, nonce) pair leak the XOR of their plaintexts and let an attacker forge tags. We dodge this by generating 12 random bytes per encryption from `OsRng`. The probability of a collision after $2^{32}$ encryptions under one key is around $2^{-33}$, which is fine for a personal vault that will see thousands of entries, not billions. If you ever push past that range, switch to a counter-based scheme or use `XChaCha20-Poly1305` with its 192-bit nonce.

The plaintext returned from `decrypt` is wrapped in `Zeroizing<Vec<u8>>`. Same reasoning as the key: when the caller is done reading the entry, the bytes get scrubbed.

## The vault format

SQLite is overkill for a few hundred entries, but it gives us free atomic writes (the `ROLLBACK JOURNAL` survives crashes), free concurrent reads, and a query language we can extend later. The trick is that we do not want SQLite to see plaintext entry contents, only the ciphertext.

Schema:

```sql
CREATE TABLE meta (
    k TEXT PRIMARY KEY,
    v BLOB NOT NULL
);

CREATE TABLE entries (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    name TEXT NOT NULL UNIQUE,
    blob BLOB NOT NULL,
    updated_at INTEGER NOT NULL
);
```

`meta` holds the salt and KDF parameters as a small JSON blob keyed by `header`. `entries` holds one row per stored credential, where `name` is the lookup key (e.g. `github.com`) and `blob` is `nonce || ciphertext || tag`. The plaintext, before encryption, is JSON like `{"username":"alice","password":"hunter2","notes":"work account"}`. Storing structured data lets us add fields later without a schema migration.

Listing names does not require decryption. That is intentional: the user can `pm ls` to see what they have without typing the master password. Reading the actual password requires unlocking the vault, which prompts for the master and re-derives the key.

## Generating new passwords

A password manager that stores garbage passwords is no better than the user's brain. Generation lives next to the rest:

```rust
use rand::seq::SliceRandom;

const ALPHABET: &[u8] =
    b"ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789!@#$%^&*-_=+";

fn generate_password(len: usize) -> String {
    let mut rng = rand::thread_rng();
    (0..len)
        .map(|_| *ALPHABET.choose(&mut rng).unwrap() as char)
        .collect()
}
```

`rand::thread_rng()` is seeded from `OsRng` (which on Linux means `getrandom(2)`), reseeds periodically, and is cryptographically secure. A 20-character password from this alphabet has $\log_2(74^{20}) \approx 124$ bits of entropy, which is well past anything brute-force-able and well past anything the destination site is going to accept anyway.

A common mistake in homemade generators is using `%` to map a random byte to an alphabet index. If `byte % alphabet.len()` is computed over a 256-value range with a 74-character alphabet, the first 36 characters appear slightly more often than the rest because 256 is not divisible by 74. `SliceRandom::choose` from the `rand` crate uses rejection sampling internally, so the distribution is uniform.

## Clipboard with auto-clear

Pasting passwords into login forms is the actual user-facing job. `arboard` makes that a one-liner, but the interesting part is what happens after.

```rust
use std::time::Duration;
use std::thread;

fn copy_with_clear(value: &str, secs: u64) -> anyhow::Result<()> {
    let mut cb = arboard::Clipboard::new()?;
    cb.set_text(value.to_string())?;
    println!("copied. clearing in {secs}s");

    let original = value.to_string();
    thread::spawn(move || {
        thread::sleep(Duration::from_secs(secs));
        if let Ok(mut cb) = arboard::Clipboard::new() {
            // Only clear if our value is still on the clipboard.
            // If the user copied something else, leave it alone.
            if cb.get_text().ok().as_deref() == Some(original.as_str()) {
                let _ = cb.set_text(String::new());
            }
        }
    });
    Ok(())
}
```

Two design notes. First, the timer thread does not block the main process. The CLI returns control to the user immediately and the clearing happens in the background. On Wayland this matters because the clipboard contents are tied to the source process: if `pm` exits, the clipboard goes empty regardless. We side-step that by `thread::spawn` from main and only `return` from main *after* the sleep, in practice with a top-level `await` or a `join`. The sample above is illustrative; in the real binary the main thread waits, prints a single dot per second, and exits cleanly when the timer fires.

Second, the comparison before clearing is critical. If the user copied something else in the intervening 30 seconds (an address, a code from their authenticator), we do not want to wipe that. A naive `cb.set_text("")` would do exactly that and confuse anyone who used the manager casually.

`arboard` on Linux talks to `wl-copy` semantics on Wayland and X11 selection on X. On macOS it goes through `NSPasteboard`. Each platform has different timeouts on what "the clipboard" even means; X11, for instance, throws away clipboard contents when the source process exits unless a clipboard manager is running. There is no portable solution. The 30-second clear is an upper bound; the actual contents may already be gone.

## CRUD: putting it together

The `clap` derive gives us the surface. Each command takes the master password through `rpassword`, derives the key, and operates on the vault.

```rust
use clap::{Parser, Subcommand};

#[derive(Parser)]
#[command(version, about = "minimal cli password manager")]
struct Cli {
    #[arg(long, default_value = "vault.db")]
    vault: String,
    #[command(subcommand)]
    cmd: Cmd,
}

#[derive(Subcommand)]
enum Cmd {
    Init,
    Add { name: String, #[arg(long)] user: String, #[arg(long, default_value_t = 20)] len: usize },
    Get { name: String, #[arg(long, default_value_t = 30)] clear: u64 },
    Ls,
    Rm { name: String },
}
```

The `add` flow looks like this:

```rust
fn cmd_add(vault: &Vault, key: &[u8; 32], name: &str, user: &str, len: usize) -> anyhow::Result<()> {
    let pw = generate_password(len);
    let plaintext = serde_json::to_vec(&serde_json::json!({
        "username": user,
        "password": pw,
        "created": chrono::Utc::now().to_rfc3339(),
    }))?;
    let blob = encrypt(key, &plaintext)?;
    vault.upsert(name, &blob)?;
    println!("stored entry {name}");
    Ok(())
}
```

`vault.upsert` is a thin wrapper over `INSERT INTO entries(name, blob, updated_at) VALUES (?, ?, ?) ON CONFLICT(name) DO UPDATE SET blob=excluded.blob, updated_at=excluded.updated_at`. SQLite handles the atomicity. If the process is killed mid-write, the journal rolls back and the previous entry is intact.

`get` is symmetrical:

```rust
fn cmd_get(vault: &Vault, key: &[u8; 32], name: &str, clear_secs: u64) -> anyhow::Result<()> {
    let blob = vault.get(name)?.ok_or_else(|| anyhow::anyhow!("no entry: {name}"))?;
    let plaintext = decrypt(key, &blob)?;
    let v: serde_json::Value = serde_json::from_slice(&plaintext)?;
    let pw = v["password"].as_str().ok_or_else(|| anyhow::anyhow!("malformed entry"))?;
    copy_with_clear(pw, clear_secs)?;
    Ok(())
}
```

The decrypted plaintext is wrapped in `Zeroizing<Vec<u8>>` and dropped at the end of the function. The clipboard copy is the only path the password leaves the process by; we do not log it, do not pass it to format strings except the literal value, and do not stash it in a `String` variable that lives longer than necessary.

## What still goes wrong

The master password sits in memory the whole time the process is running. `rpassword::read_password()` returns a `String`, which the standard library does not zeroize. You can fix this in two ways: wrap the result in `Zeroizing<String>` immediately and pass references, or use the [`secrecy`](https://crates.io/crates/secrecy) crate which wraps secrets in a type that prevents accidental `Debug`/`Display` and zeroes on drop. For a real tool, do the latter.

Rust's heap allocator may move data when a `Vec` grows. If you push to a `Vec<u8>` holding a key, the old buffer is freed without zeroizing. `Zeroizing<Vec<u8>>` does not help here because `Drop` only fires on the final owner. Use a `[u8; 32]` for keys (fixed size, never reallocated) or `zeroize::Zeroizing<Box<[u8]>>` for sized buffers and avoid `Vec` for secret material.

Process memory dumps and core files are not addressed at all. On Linux, `prctl(PR_SET_DUMPABLE, 0)` disables core dumps for the process. `mlock(2)` keeps pages off swap. The [`memsec`](https://crates.io/crates/memsec) crate wraps both. If the threat model includes core dumps or attackers with disk access to swap, you need these. For a personal CLI on a full-disk-encrypted laptop, they are nice-to-haves.

The clipboard primary selection on X11 is separate from the regular clipboard. Some applications (notably terminals) auto-paste the primary selection on middle-click. `arboard` only writes to the regular clipboard, so this is mostly fine, but if the user explicitly selects the displayed password with the mouse, that text lands in primary and lives until they select something else. The cleanest fix is to never `println!` the password at all.

GCM authentication tags are 16 bytes. If you truncate them you trade authenticity for size, and there is no good reason to do that here. Keep the full tag.

## The whole binary

All in, the `main.rs` weighs around 280 lines: roughly 60 for the CLI, 50 for the SQLite wrapper, 40 each for crypto and key derivation, 30 for the clipboard helper, 20 for password generation, and the rest for error handling and the `init`/`unlock` flows. Compile time is around 8 seconds incremental, 35 seconds clean, with a release binary of about 4 MB stripped (mostly SQLite).

The point of writing it was never to ship a competitor to KeePassXC. It is that every block above is a real cryptographic decision: which AEAD, what KDF cost, where the nonce comes from, what gets zeroed when, what the user is allowed to see without authenticating. You do not get a real intuition for these decisions from a textbook. You get it from picking each one wrong once, watching the test fail, and going back to the docs.

A working starter version of this code lives at [github.com/RustCrypto/AEADs](https://github.com/RustCrypto/AEADs) for the AEAD side, [github.com/RustCrypto/password-hashes](https://github.com/RustCrypto/password-hashes) for Argon2, and [github.com/iddm/arboard](https://github.com/1Password/arboard) for the clipboard. Read the source. The implementations are short, well-commented, and assume you are exactly the kind of person who builds toy password managers to learn.
