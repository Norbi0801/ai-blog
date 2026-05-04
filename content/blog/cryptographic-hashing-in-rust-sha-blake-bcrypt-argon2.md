+++
title = "Cryptographic hashing in Rust - SHA, BLAKE, bcrypt, argon2"
date = 2025-03-26
description = "A practical tour of hash functions in Rust: when to use SHA-256, when BLAKE3 wins, why bcrypt and argon2 exist, and how to not shoot yourself in the foot."

[taxonomies]
tags = ["rust", "security", "cryptography", "auth"]
+++

Hashing looks like one thing from the outside. Text in, fixed-size blob out. In practice there are at least three distinct families of hash functions, each designed for a different threat model, and picking the wrong one is how you end up in breach write-ups. Use SHA-256 to hash passwords and your user database becomes a rainbow table buffet. Use bcrypt to fingerprint a 10GB file and your CI will still be running next Tuesday.

This post is a walk through the four hash families you will actually reach for in Rust - SHA-2, BLAKE3, bcrypt, and Argon2 - plus HMAC for message authentication. Each with the right crate, the right API, and the specific failure modes that keep showing up in postmortems.

<!-- more -->

## What a cryptographic hash actually guarantees

Any hash function maps arbitrary-length input to fixed-length output. What makes a hash function *cryptographic* is three properties that non-crypto hashes (like [FxHash](https://crates.io/crates/fxhash) or [ahash](https://crates.io/crates/ahash)) do not provide:

1. **Preimage resistance.** Given a hash `h`, it should be computationally infeasible to find any input `x` such that `hash(x) == h`. This is what stops you reversing a password hash into the password.
2. **Second preimage resistance.** Given an input `x1`, it should be infeasible to find a different input `x2` such that `hash(x1) == hash(x2)`. This matters when the attacker knows one input and wants to forge a collision against it.
3. **Collision resistance.** It should be infeasible to find *any* pair `(x1, x2)` such that `hash(x1) == hash(x2)`. This is strictly harder to break than second preimage resistance because the attacker has full freedom over both sides.

MD5 and SHA-1 fail collision resistance today. MD5 collisions can be generated in seconds on a laptop. SHA-1 collisions have been demonstrated by [SHAttered](https://shattered.io/) in 2017 for around 6,500 CPU-years of compute, which is well within reach of a motivated attacker. Both are still preimage-resistant in practice, so using them as a cache key is fine, but any security-sensitive use (signatures, content addressing where an attacker picks inputs, HMAC with MD5) is broken. Git famously still uses SHA-1, with [plans to migrate to SHA-256](https://git-scm.com/docs/hash-function-transition/).

Non-cryptographic hashes fail all three properties by design - they are optimized for speed and good distribution, not resistance. Do not confuse `std::collections::HashMap`'s hasher with what this post is about.

## SHA-256 - the boring default for non-password hashing

SHA-256 is part of the SHA-2 family designed by the NSA and published in 2001. It produces a 256-bit (32-byte) output, is not broken, and is implemented in hardware on every x86-64 CPU from roughly 2015 onward via the SHA extensions (SHA-NI). On AArch64 the equivalent instructions landed with ARMv8.

Use it when you need:

- A content-addressed identifier (files, blobs, cache keys where correctness matters)
- A deterministic fingerprint that will be checked by code written in other languages or decades from now
- Input to HMAC for message authentication
- Part of a signature scheme

Do NOT use it for:

- Password hashing (see Argon2 section - SHA-256 is too fast)
- Short-input deduplication where the input space is small (attacker can brute force)

The Rust crate is [`sha2`](https://crates.io/crates/sha2), part of the RustCrypto project. Version 0.10 has been stable for years and is used by `cargo`, `rustup`, and basically every Rust binary that touches a checksum.

```rust
use sha2::{Sha256, Digest};

fn main() {
    let mut hasher = Sha256::new();
    hasher.update(b"the quick brown fox");
    hasher.update(b" jumps over the lazy dog");
    let result = hasher.finalize();

    // result is GenericArray<u8, U32>, deref as &[u8]
    println!("{:x}", result);
    // 05c6e08f1d9fdafa03147fcb8f82f124c76d2f70e3d989dc8aadb5e7d7450bec
}
```

The `Digest` trait is the key abstraction. Every RustCrypto hash (`Sha256`, `Sha512`, `Sha3_256`, `Blake2b512`) implements it, so you can swap algorithms by changing one line. `update` can be called repeatedly - internally the hasher processes input in 64-byte blocks and buffers the rest, so feeding it chunks from a file is free.

For a whole file:

```rust
use std::io::{self, Read};
use std::fs::File;
use sha2::{Sha256, Digest};

fn hash_file(path: &str) -> io::Result<[u8; 32]> {
    let mut file = File::open(path)?;
    let mut hasher = Sha256::new();
    let mut buf = [0u8; 8192];
    loop {
        let n = file.read(&mut buf)?;
        if n == 0 { break; }
        hasher.update(&buf[..n]);
    }
    Ok(hasher.finalize().into())
}
```

On my laptop (Ryzen 7, SHA-NI enabled) this hashes around 1.8 GB/s. Without SHA-NI the software fallback drops to around 400 MB/s. You can check which you have with `RUSTFLAGS="-C target-cpu=native" cargo build --release`.

## BLAKE3 - the Rust-native answer

[BLAKE3](https://github.com/BLAKE3-team/BLAKE3) is a newer hash, published in 2020 by the same team behind BLAKE2, Zooko Wilcox, and Jack O'Connor. It is designed to be *fast* on modern CPUs by exploiting SIMD and intrinsic parallelism - the internal tree structure means the same input can be hashed across multiple cores with no synchronization.

On the same Ryzen 7 laptop, single-threaded BLAKE3 hits around 6 GB/s. With 8 threads, over 30 GB/s. That is faster than most SSDs can deliver data, so your file hasher is now I/O-bound instead of CPU-bound.

The [`blake3`](https://crates.io/crates/blake3) crate is written in Rust with SIMD paths for AVX2, AVX-512, NEON, and a portable fallback. Version 1.5.x at the time of writing.

```rust
use blake3::Hasher;

fn main() {
    let mut hasher = Hasher::new();
    hasher.update(b"hello world");
    let hash = hasher.finalize();
    println!("{}", hash.to_hex());
    // d74981efa70a0c880b8d8c1985d075dbcbf679b99a5f9914e5aaf96b831a9e24
}
```

BLAKE3 also supports keyed hashing (replaces HMAC for most use cases) and key derivation:

```rust
// Keyed mode - MAC without HMAC construction
let key: [u8; 32] = *b"01234567890123456789012345678901";
let mut h = Hasher::new_keyed(&key);
h.update(b"message");
let mac = h.finalize();

// Key derivation
let master = b"my root secret";
let derived = blake3::derive_key("myapp 2026-04-22 session-key", master);
// derived is [u8; 32]
```

When to pick BLAKE3 over SHA-256:

- You control both sides of the protocol (internal fingerprinting, custom sync tools, content-addressed storage in your own system)
- Performance matters and you are hashing a lot of data
- You need keyed hashing or extendable output (XOF)

When to stick with SHA-256:

- Interoperability with other systems, standards, or signature schemes
- Anything FIPS-regulated (BLAKE3 is not NIST-blessed)
- Existing protocols that specify SHA-2

Both are fine cryptographically. This is a portability vs speed trade-off, not a security one.

## Why SHA and BLAKE are the WRONG choice for passwords

A 4-letter lowercase password has 26^4 = 456,976 possibilities. On a modern GPU, SHA-256 runs at roughly 10 billion hashes per second. A RTX 4090 cracks every 4-letter password in your database in about 45 microseconds, all of them, in parallel. At 6 letters it is still under a second. At 8 letters you are up to minutes.

This is exactly why fast hashes fail for passwords. The whole point of fast hashing is that we cannot slow down an attacker selectively. Hashing a million API request bodies per second is great for throughput and terrible when that million-per-second is someone running `hashcat` against your leaked user table.

The fix is to use a *slow* hash - a function intentionally designed to be expensive to compute, with a tunable work factor so you can re-tune as hardware improves. Two families dominate: bcrypt (1999) and Argon2 (2015, winner of the Password Hashing Competition).

## bcrypt - the old reliable

[bcrypt](https://en.wikipedia.org/wiki/Bcrypt) was designed by Niels Provos and David Mazieres, based on the Blowfish cipher's expensive key setup. It has two properties that matter:

1. A *cost* parameter that doubles the time every time you increment it. Cost 10 = 2^10 = 1024 rounds of the expensive setup. Cost 12 = 4096 rounds.
2. A built-in 128-bit salt, included in the output string, so you do not need to store salts separately.

The output is a 60-character string like:

```
$2b$12$R9h/cIPz0gi.URNNX3kh2OPST9/PgBkqquzi.Ss7KIUgO2t0jWMUW
```

That encodes the algorithm (`$2b$`), the cost (`$12$`), and the salt+hash concatenated. When you verify, you pass the full string back in and bcrypt extracts the parameters - no separate columns needed.

The Rust crate is [`bcrypt`](https://crates.io/crates/bcrypt), currently 0.17.x:

```rust
use bcrypt::{hash, verify, DEFAULT_COST};

fn main() -> Result<(), Box<dyn std::error::Error>> {
    let password = "correct horse battery staple";
    let hashed = hash(password, DEFAULT_COST)?;  // cost 12

    assert!(verify(password, &hashed)?);
    assert!(!verify("wrong", &hashed)?);
    Ok(())
}
```

`DEFAULT_COST` is 12, which takes roughly 250 ms on a 2024-era laptop. That is the right order of magnitude for a login: slow enough to make offline cracking painful (billions of attempts per dollar of GPU time vs trillions for SHA-256), fast enough that a real user does not notice.

bcrypt's main weakness is its 72-byte input truncation. Any password longer than 72 bytes is silently cut off. If you accept passphrases, pre-hash the password through SHA-256 first and feed the hex or base64 into bcrypt. Or use Argon2.

## Argon2 - the modern default

Argon2 won the Password Hashing Competition in 2015 and has been the recommended default ever since, including by [OWASP](https://cheatsheetseries.owasp.org/cheatsheets/Password_Storage_Cheat_Sheet.html). It has three parameters instead of one:

- **Time cost** (`t`) - iterations, linear cost
- **Memory cost** (`m`) - KB of RAM the function must allocate
- **Parallelism** (`p`) - lanes that can be computed in parallel

The memory cost is the key innovation. GPUs have lots of compute but relatively little memory per core. Forcing a password hash to allocate 64 MB of RAM per attempt means a GPU that can do 10,000 parallel SHA-256 hashes can only do a few dozen parallel Argon2 hashes. The asymmetric advantage the attacker had is erased.

There are three variants:

- **Argon2d** - faster, vulnerable to side-channel attacks. Use for cryptocurrencies, not passwords.
- **Argon2i** - side-channel resistant but slower. Deprecated in favor of id.
- **Argon2id** - hybrid, what you should use. OWASP recommends it.

The Rust crate is [`argon2`](https://crates.io/crates/argon2), 0.5.x:

```rust
use argon2::{
    password_hash::{rand_core::OsRng, PasswordHash, PasswordHasher, PasswordVerifier, SaltString},
    Argon2,
};

fn main() -> Result<(), Box<dyn std::error::Error>> {
    let password = b"correct horse battery staple";

    // Generate 16-byte salt from OS RNG
    let salt = SaltString::generate(&mut OsRng);

    // Default: Argon2id, m=19456 KiB, t=2, p=1 (OWASP 2024 baseline)
    let argon2 = Argon2::default();
    let hash = argon2.hash_password(password, &salt)?.to_string();

    // Output looks like:
    // $argon2id$v=19$m=19456,t=2,p=1$c29tZXNhbHQ$hash...

    // Verification parses the parameters out of the string
    let parsed = PasswordHash::new(&hash)?;
    assert!(Argon2::default().verify_password(password, &parsed).is_ok());
    Ok(())
}
```

One thing to notice: unlike bcrypt, you generate the salt explicitly from `OsRng`. That is the contract of the `password_hash` trait crate, which Argon2 implements. The salt goes into the output string in PHC format, so again you only store one column.

For tuning, the [OWASP cheatsheet](https://cheatsheetseries.owasp.org/cheatsheets/Password_Storage_Cheat_Sheet.html#argon2id) currently recommends `m=19456 KiB, t=2, p=1` as the minimum. If your server has more memory, bump `m` up. Aim for roughly 500 ms per hash on your actual production hardware and re-tune every year or two.

## Timing attacks and constant-time comparison

Every password verification ends with comparing a computed hash against a stored hash. If you do this with `==` on a `String` or `&[u8]`, you have introduced a timing side channel: the comparison returns as soon as it finds the first differing byte. An attacker who can measure response time down to microseconds learns how many leading bytes of their guess match.

In practice, network jitter usually drowns this out for remote attacks, but the fix is so cheap you should just always do it:

```rust
use subtle::ConstantTimeEq;

let a: &[u8] = b"expected";
let b: &[u8] = b"provided";
if a.ct_eq(b).into() {
    // equal
}
```

The [`subtle`](https://crates.io/crates/subtle) crate provides `ConstantTimeEq` and friends. Internally `ct_eq` XORs all bytes and ORs the results, always touching every byte regardless of content. Compiler optimizations that would short-circuit are blocked by `black_box`-style tricks.

The good news: `bcrypt::verify` and `argon2::verify_password` both use constant-time comparison internally. You only need to worry about `subtle` if you are writing your own verification code, comparing API tokens, or checking HMAC tags.

## HMAC - message authentication, not password hashing

HMAC (Hash-based Message Authentication Code) solves a different problem: given a message and a shared secret key, produce a tag that proves the message was not tampered with and was produced by someone who knows the key. It is defined in [RFC 2104](https://www.rfc-editor.org/rfc/rfc2104) and parameterized by any cryptographic hash - `HMAC-SHA256` is by far the most common.

You cannot just do `hash(key || message)` and call it an HMAC. Length-extension attacks against Merkle-Damgard hashes (which SHA-256 is) mean an attacker can append data to the message and extend the tag without knowing the key. HMAC uses two hash invocations with carefully derived inner and outer keys specifically to prevent that.

In Rust, use the [`hmac`](https://crates.io/crates/hmac) crate:

```rust
use hmac::{Hmac, Mac};
use sha2::Sha256;

type HmacSha256 = Hmac<Sha256>;

fn main() -> Result<(), Box<dyn std::error::Error>> {
    let mut mac = HmacSha256::new_from_slice(b"shared secret key")?;
    mac.update(b"message to authenticate");

    let tag = mac.finalize().into_bytes();  // GenericArray<u8, U32>

    // Verification (constant-time internally)
    let mut mac2 = HmacSha256::new_from_slice(b"shared secret key")?;
    mac2.update(b"message to authenticate");
    mac2.verify_slice(&tag)?;

    Ok(())
}
```

HMAC is what lives inside `HS256` JWTs (see [JWT deep dive](/blog/jwt-deep-dive-how-tokens-work-and-common-mistakes/)), webhook signatures from Stripe and GitHub, and most symmetric API authentication schemes. Any time you see an X-Signature header on a webhook, it is probably HMAC-SHA256 over the raw request body with a secret you provisioned.

A note on BLAKE3 keyed mode: it provides the same guarantee as HMAC-SHA256 without the HMAC construction wrapper, because BLAKE3 is not vulnerable to length extension (its internal state has hidden bits). If you control both ends, `Hasher::new_keyed` is simpler and faster than HMAC-BLAKE3 would be.

## Do not roll your own

Every crate mentioned here - `sha2`, `blake3`, `bcrypt`, `argon2`, `hmac`, `subtle` - is maintained, audited, and used by production systems you depend on. The implementations handle:

- Constant-time operations where required
- Proper salt generation from OS entropy
- Correct parameter encoding in output strings
- Upgrade paths (bcrypt's `$2a$` vs `$2b$` prefix, Argon2's version field)
- Platform-specific SIMD acceleration with software fallback

None of that is interesting business logic. All of it is extremely hard to get right. The history of "I wrote my own" ends at [Adobe's 2013 breach](https://nakedsecurity.sophos.com/2013/11/04/anatomy-of-a-password-disaster-adobes-giant-sized-cryptographic-blunder/) (3DES-ECB with no salting on 150 million passwords), or any of the dozens of "we hashed with SHA-1 and hoped" incidents on [haveibeenpwned.com](https://haveibeenpwned.com/). There is no credit for being clever here.

## A cheat sheet

- Fingerprinting files, generating content-addressed IDs, CAS cache keys: `sha2::Sha256` for interop, `blake3` for speed
- Message authentication with a shared secret: `hmac::Hmac<Sha256>` or `blake3::Hasher::new_keyed`
- Key derivation from a high-entropy secret: `blake3::derive_key` or HKDF (`hkdf` crate)
- Password storage: `argon2` with OWASP defaults, or `bcrypt` if you need the simplicity and accept the 72-byte limit
- Comparing secrets (tokens, MAC tags, hashes): `subtle::ConstantTimeEq`
- Anything else "crypto-shaped": stop, read the [RustCrypto docs](https://github.com/RustCrypto), and use a vetted crate

Hashing has no single right answer because it has no single problem. SHA-256 and BLAKE3 are the right answer for "I need a fingerprint." Argon2 and bcrypt are the right answer for "I need to store this secret an attacker will eventually see." HMAC is the right answer for "I need to prove this message came from a trusted party." Use them for what they are for, tune the parameters for your hardware, and let the well-maintained crates handle the sharp edges.
