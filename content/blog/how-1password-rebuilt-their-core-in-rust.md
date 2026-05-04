+++
title = "How 1Password rebuilt their core in Rust"
date = 2025-09-27
description = "A deep look at how 1Password replaced platform-specific codebases with a shared Rust core - the architecture, the crypto, the FFI, and the hard parts."

[taxonomies]
tags = ["rust", "ffi", "architecture", "security"]
+++

I wrote briefly about 1Password in my earlier post on [Rust in production](/blog/rust-in-production-what-companies-actually-use-it-for/). The short version: they built a shared Rust core, went from 0% to 63% code sharing across platforms, and open-sourced [Typeshare](https://github.com/1Password/typeshare). That was the 30-second summary.

This post is the full story. How the architecture actually works, how native apps talk to the Rust core, why memory safety isn't just a buzzword when you're handling crypto, and what went wrong along the way.

<!-- more -->

## The problem: N platforms, N implementations

Before Rust, 1Password had separate codebases for macOS (Swift), Windows (C#), Android (Kotlin), iOS (Swift), Linux (various), and browser extensions (JavaScript/Go). Each platform had its own implementation of everything - crypto, sync, vault logic, server communication.

Think about what that means for a password manager. Cryptographic code - the part that encrypts and decrypts your vault - was implemented independently on every platform. A bug fix in the iOS crypto layer had to be separately implemented and tested on Android, Windows, macOS, and the browser extension. Not just ported - reimplemented, because each platform used different languages with different crypto primitives.

This is the kind of situation where "it works" and "it's correct" are two different statements. Crypto bugs are silent. A subtle difference in how two platforms handle padding, or key derivation, or nonce generation doesn't crash your app. It weakens your security without anyone noticing until a security audit (or worse, an attacker) finds it.

The answer was to write the crypto exactly once, in a language that compiles to every target platform. That language turned out to be Rust.

## The architecture: a headless password manager

1Password's Rust core isn't a library you call into occasionally. It's a complete headless application - a password manager with no UI. It maintains its own runtime loop and handles:

- All cryptographic operations (encryption, decryption, key derivation, TOTP generation)
- Database access (local SQLite vault storage)
- Server communication and authentication
- Data synchronization between cloud and local storage
- Business logic (vault organization, item management, sharing)
- View model generation (the data structures the UI layer needs to render)

The native apps on each platform are thin UI shells. SwiftUI on iOS/macOS, React + Electron (with [Neon](https://neon-rs.dev/) bindings) on desktop, Kotlin on Android. They send requests to the Rust core and render whatever comes back.

This isn't a traditional "call a function, get a result" library pattern. The core runs its own event loop. Native apps communicate with it through what 1Password calls **invocations** - structured requests sent over FFI with C calling conventions. The core processes these asynchronously through an internal channel-based system, then fires callbacks with response data when operations complete.

```
Native App (Swift/Kotlin/TypeScript)
        │
        ▼
   FFI Layer (C calling convention)
        │
        ▼
   Channel-based async dispatcher
        │
        ▼
   Rust Core runtime loop
   ├── Crypto engine (ring + zeroizing)
   ├── Sync engine (pull-first strategy)
   ├── SQLite database layer
   ├── Server communication (reqwest)
   └── View model generation
        │
        ▼
   Callback with response data
        │
        ▼
   Native App renders UI
```

This design has a crucial property: the UI layer never touches sensitive data directly. Secrets pass through the FFI boundary only when necessary, and the Rust core controls when and how that happens.

## Crypto: why Rust actually matters here

"Memory safety" gets thrown around a lot in Rust marketing. For most web services, a memory bug means a crash and a restart. For a password manager, a memory bug can mean your master password or decrypted vault contents sitting in memory longer than they should - accessible to any process that can read your address space, or recoverable from a memory dump.

1Password's crypto stack is built on [ring](https://github.com/briansmith/ring), a Rust cryptography library that follows a deliberately restrictive API design. Ring is intentionally hard to misuse. You can't accidentally use ECB mode, you can't skip authentication on ciphertext, you can't reuse nonces. The type system makes the wrong thing hard to express.

But ring alone doesn't solve the memory problem. When you decrypt a password, the plaintext exists somewhere in memory. In a garbage-collected language, you can't control when that memory gets reclaimed or whether it gets zeroed. The GC might move the data during compaction, leaving copies in the old location. You literally cannot guarantee that sensitive data is wiped.

In Rust, you have direct control. 1Password built and open-sourced [zeroizing-alloc](https://crates.io/crates/zeroizing-alloc), a custom allocator that zeros memory on free. Combined with the [secrecy](https://crates.io/crates/secrecy) crate (which prevents accidental mutation or cloning of secret values), they can enforce a pattern where:

1. Sensitive data is allocated through the zeroizing allocator
2. The `secrecy` crate wraps it in a type that prevents accidental exposure (no `Display` impl, no `Clone`)
3. When the value is dropped, the allocator flips every byte to zero before releasing the memory

This pattern is hard to get right in C (you have to remember to call the zeroing function every time, and compilers can optimize away `memset` on memory that's about to be freed). It's impossible to guarantee in GC languages. In Rust, the `Drop` trait makes it automatic - when the value goes out of scope, zeroing happens. No developer discipline required.

Here's what that looks like conceptually:

```rust
use secrecy::{ExposeSecret, SecretString};

// Secret data is wrapped - can't accidentally log or serialize it
let master_password: SecretString = get_password_from_user().into();

// Explicit exposure required to use the value
let derived_key = derive_key(master_password.expose_secret());

// When master_password drops, memory is zeroed automatically
// No developer has to remember to call a cleanup function
```

The `secrecy` crate's `SecretString` doesn't implement `Display`, `Debug`, `Serialize`, or `Clone`. If a developer accidentally writes `println!("{}", master_password)`, it won't compile. Compare that to a plain `String` where logging sensitive data is one `dbg!()` away.

1Password also disabled dynamic logging by default across their Rust core. Adding any new loggable data field requires a security review. This isn't a Rust feature per se, but the type system makes it enforceable - you can't accidentally format a `SecretString` into a log message.

## The Brain: from Go to Rust to WebAssembly

One of the more interesting migrations was the "1Password Brain" - the engine that analyzes web pages and fills in credentials. This component lives in the browser extension and needs to parse DOM structures, identify form fields, match them against vault entries, and fill them correctly.

The Brain was originally written in Go. In late 2019, 1Password [rewrote it in Rust and compiled it to WebAssembly](https://blog.1password.com/1passwordx-december-2019-release/) for the browser extension. The motivation was performance - Go compiled to WASM produced large binaries with a runtime overhead that showed up in page interaction latency.

The results were dramatic. From [1Password's own measurements](https://serokell.io/blog/rust-in-production-1password):

- Page filling and analysis ran **at least 2x faster** across the board
- Pages with many form fields saw **up to 13x improvement in Chrome**
- Firefox showed **up to 39x faster** performance on complex pages

Those aren't micro-benchmark numbers. That's real user-facing latency on real web pages. The 39x number on Firefox specifically came from pages with a large number of fields - think enterprise admin panels or long registration forms where the old Go-to-WASM implementation would visibly lag.

The Rust WASM binary also let them share logic between the browser extension and the native apps. TOTP generation, Markdown parsing for secure notes, and form-filling heuristics are all written once in Rust and compiled to both native targets and WASM.

But WASM brought its own problems.

## WASM: the hard parts

1Password discovered that using Rust compiled to WASM as a function library works well. Using it as a full runtime does not.

Their initial ambition was to run the entire Rust core in WASM for the web app - the same core that runs natively on desktop and mobile. They hit several walls:

**Binary size is critical in WASM.** Every kilobyte matters when you're loading code in a browser. The tokio async runtime, which works great on native targets, has a non-trivial footprint in WASM. The team had serious internal debates about whether hand-rolling their own futures was worth saving 14KB. In native code, nobody would think twice about 14KB. In WASM, it's a real trade-off.

**Threading model mismatch.** Tokio's work-stealing scheduler (which I covered in detail in [the tokio post](/blog/understanding-tokio-the-rust-async-runtime-under-the-hood/)) maps to OS threads on native platforms. In WASM, you're working with Web Workers, which have different semantics - no shared memory by default, message-passing instead of mutexes. The 1:1 mapping between tokio's runtime model and WASM workers simply doesn't exist.

**State synchronization.** For the web app, 1Password uses a Redux store on the JavaScript side that needs to stay in sync with state inside the WASM core. They built a bridge layer to handle this, but it adds complexity that doesn't exist on native platforms where the Rust core owns all state directly.

The current web app runs with a partial Rust core integration - shared libraries compiled to WASM handle specific tasks (crypto, filling, parsing), but the full headless runtime approach that works on native platforms hasn't been fully ported. Andrew Burkhart, a senior Rust developer at 1Password, described the situation on the [Syntax podcast](https://syntax.fm/show/776/how-1password-uses-wasm-and-rust-for-local-first-dev-with-andrew-burkhart/transcript): standing up an entire runtime in WASM remains a challenge, and the limitations are mostly in the WASM platform itself rather than in Rust.

## The sync engine: clients do the hard work

1Password's sync architecture follows a pull-first, push-second strategy, and the design reflects a fundamental constraint of end-to-end encryption: the server is cryptographically blind.

The server stores encrypted blobs, version numbers, and access metadata. It cannot read, merge, or resolve conflicts in your data because it literally cannot decrypt it. This means all conflict resolution has to happen on the client side, in the Rust core.

The flow works like this:

1. A "notifier" microservice pings connected clients via WebSocket when changes occur on the server
2. The client debounces these notifications into a sync job (to avoid hammering the server on rapid changes)
3. The client pulls the complete account state from the server
4. Conflict resolution runs locally - the Rust core compares local state against the pulled state
5. Resolved changes are pushed back to the server

Developers at 1Password define merge strategies via Rust traits for each data type. This is a deliberate design choice - there's no automatic "last write wins" default that could silently drop data. Each entity type declares how conflicts should be resolved, and the compiler enforces that every entity has a strategy.

This is a good example of Rust's type system doing real architectural work. In a dynamically typed language, it's easy to add a new data type and forget to define its merge behavior - you'd only discover the bug when two clients conflict on that type in production. In Rust, the trait bound means the code won't compile if the merge strategy is missing.

## Typeshare: keeping the FFI boundary honest

When your Rust core produces data structures that Swift, Kotlin, and TypeScript all need to consume, you have a type synchronization problem. Change a field name in Rust, and suddenly the iOS app is trying to deserialize a field that no longer exists.

1Password's solution was [Typeshare](https://github.com/1Password/typeshare), which they open-sourced in 2022. You annotate Rust types with `#[typeshare]`, and a CLI tool generates equivalent type definitions in Swift, Kotlin, TypeScript, Go, Python, and Scala.

```rust
#[typeshare]
#[serde(rename_all = "camelCase")]
pub struct VaultItem {
    pub id: String,
    pub title: String,
    pub category: ItemCategory,
    pub created_at: String,
    pub updated_at: String,
    pub favorite: bool,
}

#[typeshare]
#[serde(tag = "type", content = "value")]
pub enum ItemCategory {
    Login,
    SecureNote,
    CreditCard,
    Identity,
    Password,
}
```

Running `typeshare --lang typescript ./src` generates:

```typescript
export interface VaultItem {
    id: string;
    title: string;
    category: ItemCategory;
    createdAt: string;
    updatedAt: string;
    favorite: boolean;
}

export type ItemCategory =
    | { type: "Login" }
    | { type: "SecureNote" }
    | { type: "CreditCard" }
    | { type: "Identity" }
    | { type: "Password" };
```

Under the hood, Typeshare uses the [syn](https://crates.io/crates/syn) crate to parse Rust source files and extract annotated types. It respects serde annotations (`rename_all`, `tag`, `content`, `skip`), so the generated types match how the data actually serializes - not just how the Rust struct looks.

One limitation the 1Password team acknowledged: Typeshare can't see types generated inside procedural macros. If a macro creates a struct, Typeshare won't pick it up because it operates on the pre-expansion source. This matters for codebases that lean heavily on code generation.

Typeshare runs in 1Password's CI pipeline. Every PR that changes a Rust type triggers regeneration of client-side types, so the iOS and Android teams see the type changes in the same PR.

## Crate structure: compile what you need

As the Rust core grew, 1Password hit a practical problem. Compiling the core pulled in the entire dependency tree, even for platforms that only needed a subset of functionality. The browser extension doesn't need the SQLite layer. The desktop app doesn't need the WASM-specific shims.

The team restructured their Rust code into a service-oriented crate architecture. Instead of one monolithic crate, the core is split into focused crates:

- **op-app** and **op-ui** - platform integration crates that compose the others
- **foundation** - platform-specific services (kernel keyrings, biometrics, system clipboard via [arboard](https://github.com/1Password/arboard))
- Crypto crates - ring-based encryption, key derivation, TOTP
- Sync crates - the pull/push synchronization engine
- Database crates - SQLite access layer

Each platform's build only links the crates it needs. The browser extension compiles a much smaller subset than the desktop app, which matters both for binary size (especially in WASM) and for compile times.

## The team: learning Rust at scale

1Password has over 1,000 employees. Not all of them write Rust - there are dedicated platform teams for iOS, Android, and desktop that work primarily in Swift, Kotlin, and TypeScript. But a significant number of developers needed to learn Rust for the core.

The company has been open about the learning curve. From their Serokell interview: "Many of the folks on our team were new to Rust, and they experienced the typical learning curve that comes with its memory management and ownership model."

Compile times were a recurring pain point. The Rust core, with its many crates and heavy use of generics and proc macros, takes a while to build. Developers reported their "CPUs and fans getting a workout." The crate restructuring helped incrementally - changing one crate doesn't require recompiling everything - but full clean builds remain slow.

1Password invested in tooling to manage this:

- **[cargo-deny](https://github.com/EmbarkStudios/cargo-deny)** for dependency auditing and license compliance
- **Clippy** with a strict configuration to catch common mistakes
- **Nix flakes** for reproducible build environments across the team
- **[tracing](https://crates.io/crates/tracing)** (the Rust crate, not custom tooling) for structured diagnostics

They also built custom internal tooling for logging and localization that the Rust ecosystem didn't adequately provide. Security requirements around what gets logged, combined with localization needs across many languages, meant off-the-shelf solutions weren't sufficient.

## What they shipped

The end result, visible in 1Password 8 and later:

- A single Rust core that compiles to macOS, Windows, Linux, iOS, Android, and WASM
- Around **63% of the codebase** is shared Rust code
- Cryptographic operations written once, tested once, audited once
- Sync engine that handles conflict resolution consistently across every client
- Browser extension with 2-39x faster page filling via Rust-to-WASM
- Several open-source contributions: [Typeshare](https://github.com/1Password/typeshare), [passkey-rs](https://github.com/AlfioEmanueleFresta/passkey-rs) (WebAuthn/passkey implementation in Rust), [arboard](https://github.com/1Password/arboard) (cross-platform clipboard), [electron-hardener](https://github.com/1Password/electron-hardener) (security hardening for Electron apps)

The company described Rust as having "fulfilled 90% of what we were hoping for." The remaining 10% is mostly WASM limitations and compile time costs.

## Lessons worth stealing

You don't have to be building a password manager to learn from this.

**The "headless core" pattern works.** If you're building a cross-platform app and your business logic lives inside your UI framework, you're going to duplicate it. Extract the logic into a shared core with a defined API boundary, and let each platform be a thin rendering layer. Rust is great for this core, but even doing it in a shared Kotlin Multiplatform or C++ layer beats having five independent implementations.

**Type-safe FFI boundaries prevent an entire class of bugs.** Typeshare's approach - generating client types from the source of truth - is something any team with a Rust backend and mobile/web clients can adopt today. It's open source and actively maintained.

**Memory safety for crypto isn't theoretical.** If you're handling secrets, encryption keys, or authentication tokens, the zeroing-on-drop pattern with `secrecy` and a zeroizing allocator is worth adopting. It's not much code, and it eliminates the entire class of "sensitive data lingering in memory" vulnerabilities.

**WASM is good, but it's not native.** If your plan is "write it in Rust, compile to WASM, ship everywhere" - it works for libraries and focused computation. It does not work well (yet) for full application runtimes. Plan for native deployment as the primary target and WASM as a constrained subset.

**Modular crate design matters from day one.** 1Password had to restructure their crate graph after hitting compile time and binary size walls. If you're starting a cross-platform Rust core today, design your crate boundaries around deployment targets early.

1Password's rewrite wasn't a weekend project or a hack week experiment. It was a multi-year, company-wide architectural shift that required hiring Rust developers, building custom tooling, and rethinking how their apps are structured. The payoff - consistent crypto, shared business logic, fewer platform-specific bugs - was real, but so was the cost. That's an honest trade-off, and it's the kind of detail that matters more than any benchmark.
