+++
title = "Semantic versioning - what breaking changes actually means in Rust"
date = 2026-02-03
description = "A deep look at what constitutes a breaking change in Rust, what doesn't, the gray areas in between, and the tooling that catches violations before your users do."

[taxonomies]
tags = ["rust", "semver", "api-design", "tools"]
+++

You bump the version from `0.4.2` to `0.4.3`. You publish. Twenty minutes later, six issues roll in. Half the ecosystem that depends on your crate is broken.

What happened? You added a field to a public struct. That's a minor change, right? No. In Rust, that's a major breaking change - and the compiler is ruthless about it. There's no duck typing to save you, no runtime reflection to paper over the difference. If the types don't line up, the build fails. Period.

Semantic versioning in Rust is different from semver in most other ecosystems. The type system, ownership model, and exhaustive pattern matching mean that changes which would be invisible in Python or JavaScript become hard compile errors in Rust. Understanding exactly where the line is - and how to design APIs that give you room to evolve - is one of the most practical skills a Rust library author can develop.

<!-- more -->

## Quick semver refresher

If you've read my post on [conventional commits](/blog/conventional-commits-why-and-how-to-write-meaningful-commit-messages), you already know the version format: `MAJOR.MINOR.PATCH`.

- **MAJOR** (1.x.x to 2.0.0): breaking API changes
- **MINOR** (1.0.x to 1.1.0): new functionality, backwards compatible
- **PATCH** (1.0.0 to 1.0.1): bug fixes, backwards compatible

Cargo relies on this heavily. When someone writes `my-crate = "1.4"` in their `Cargo.toml`, Cargo interprets that as `>=1.4.0, <2.0.0`. It will happily pull `1.4.7` or `1.5.0`, but never `2.0.0`. This means every minor or patch release you make gets automatically adopted by everyone who depends on you. If you sneak a breaking change into a minor release, you break their builds with zero warning.

The official [Cargo SemVer reference](https://doc.rust-lang.org/cargo/reference/semver.html) catalogs which changes are major, minor, and "possibly breaking." It's thorough but dense. Let's go through what actually matters with real code.

## What IS a breaking change

### Removing or renaming public items

This one is obvious but worth stating: if you remove a public function, struct, enum, trait, type alias, constant, or module, that's a breaking change. Same goes for renaming.

```rust
// v1.0.0
pub fn parse_config(path: &str) -> Result<Config, Error> { /* ... */ }

// v1.1.0 - BREAKING: renamed the function
pub fn load_config(path: &str) -> Result<Config, Error> { /* ... */ }
```

Every downstream crate calling `parse_config()` now fails. Doesn't matter that the behavior is identical. The name changed, the build breaks.

If you want to rename something, keep the old name around as a deprecated re-export:

```rust
pub fn load_config(path: &str) -> Result<Config, Error> { /* ... */ }

#[deprecated(since = "1.1.0", note = "renamed to `load_config`")]
pub fn parse_config(path: &str) -> Result<Config, Error> {
    load_config(path)
}
```

This buys your users time to migrate while you prepare for the next major version.

### Changing function signatures

Adding a parameter, removing a parameter, changing a parameter type, changing the return type - all breaking.

```rust
// v1.0.0
pub fn connect(host: &str, port: u16) -> Connection { /* ... */ }

// v1.1.0 - BREAKING: added a parameter
pub fn connect(host: &str, port: u16, tls: bool) -> Connection { /* ... */ }
```

In dynamic languages, you might add a keyword argument with a default value and call it a day. Rust doesn't have default arguments. Every call site must provide every argument explicitly. So this breaks every single caller.

The idiomatic workaround is the builder pattern:

```rust
pub struct ConnectOptions {
    host: String,
    port: u16,
    tls: bool,
}

impl ConnectOptions {
    pub fn new(host: impl Into<String>, port: u16) -> Self {
        Self {
            host: host.into(),
            port,
            tls: false, // sensible default
        }
    }

    pub fn tls(mut self, tls: bool) -> Self {
        self.tls = tls;
        self
    }

    pub fn connect(self) -> Connection {
        // ...
    }
}
```

Now you can add `timeout`, `proxy`, `retry_count` - whatever you need - as new builder methods without breaking anyone. We'll talk more about future-proofing patterns later.

### Adding fields to public structs

This is where Rust's semver rules surprise people coming from other languages. If all your struct fields are public and you add a new one, that's a breaking change.

```rust
// v1.0.0
pub struct DatabaseConfig {
    pub host: String,
    pub port: u16,
}

// v1.1.0 - BREAKING
pub struct DatabaseConfig {
    pub host: String,
    pub port: u16,
    pub max_connections: u32,  // new field
}
```

Why? Because Rust requires exhaustive struct construction. Downstream code doing this:

```rust
let cfg = DatabaseConfig {
    host: "localhost".into(),
    port: 5432,
};
```

...now fails. The compiler demands a value for `max_connections`. And destructuring patterns break too:

```rust
let DatabaseConfig { host, port } = cfg; // error: missing field `max_connections`
```

This is genuinely one of the most common semver mistakes in the Rust ecosystem. The fix is simple: add a private field (even a zero-sized one) or mark the struct `#[non_exhaustive]`. We'll cover `#[non_exhaustive]` in depth shortly.

### Adding enum variants

Same exhaustiveness problem, different shape. If your enum doesn't use `#[non_exhaustive]`, adding a variant is breaking.

```rust
// v1.0.0
pub enum Color {
    Red,
    Green,
    Blue,
}

// v1.1.0 - BREAKING
pub enum Color {
    Red,
    Green,
    Blue,
    Alpha(u8),  // new variant
}
```

Every `match` on `Color` that doesn't have a wildcard arm now fails:

```rust
match color {
    Color::Red => { /* ... */ }
    Color::Green => { /* ... */ }
    Color::Blue => { /* ... */ }
    // error[E0004]: non-exhaustive patterns: `Color::Alpha(_)` not covered
}
```

### Adding a non-defaulted item to a trait

If you add a method (or associated type, or associated constant) to a public trait without providing a default implementation, every implementor breaks.

```rust
// v1.0.0
pub trait Storage {
    fn get(&self, key: &str) -> Option<Vec<u8>>;
    fn set(&mut self, key: &str, value: Vec<u8>);
}

// v1.1.0 - BREAKING
pub trait Storage {
    fn get(&self, key: &str) -> Option<Vec<u8>>;
    fn set(&mut self, key: &str, value: Vec<u8>);
    fn delete(&mut self, key: &str) -> bool;  // no default - breaks all impls
}
```

Every downstream `impl Storage for MyBackend` block is now incomplete. The compiler refuses to compile it until they add the new method.

### Tightening generic bounds

If you add new trait bounds to a generic function or type, code that passed types satisfying the old (looser) bounds no longer compiles.

```rust
// v1.0.0
pub fn process<T: Clone>(item: T) { /* ... */ }

// v1.1.0 - BREAKING: added Send bound
pub fn process<T: Clone + Send>(item: T) { /* ... */ }
```

Any caller passing a `!Send` type (like `Rc<T>`) just got broken. If you've read my post on [trait bounds](/blog/rust-trait-bounds-where-clauses-associated-types-and-the-rest-of-the-iceberg/), you know how much bounds influence what types are valid - tightening them shrinks that set, which is breaking.

### Switching from no_std to requiring std

If your crate previously supported `#![no_std]` and you add code that requires `std`, that's a breaking change for every embedded or WASM user depending on you. It's less visible than a signature change but just as real.

## What is NOT a breaking change

### Adding new public items

Adding a new function, struct, module, or constant is generally safe. Existing code doesn't reference it, so it can't break.

```rust
// v1.0.0
pub fn connect(host: &str) -> Connection { /* ... */ }

// v1.1.0 - safe
pub fn connect(host: &str) -> Connection { /* ... */ }
pub fn connect_with_tls(host: &str, cert: &Path) -> Connection { /* ... */ }
```

There's a caveat involving glob imports, which I'll cover in the gray area section.

### Adding trait methods with default implementations

If the new method has a default body, existing implementors don't need to change anything:

```rust
// v1.0.0
pub trait Storage {
    fn get(&self, key: &str) -> Option<Vec<u8>>;
    fn set(&mut self, key: &str, value: Vec<u8>);
}

// v1.1.0 - safe: default impl provided
pub trait Storage {
    fn get(&self, key: &str) -> Option<Vec<u8>>;
    fn set(&mut self, key: &str, value: Vec<u8>);

    fn delete(&mut self, key: &str) -> bool {
        // default: try to set empty, return whether key existed
        let existed = self.get(key).is_some();
        if existed {
            self.set(key, Vec::new());
        }
        existed
    }
}
```

All existing `impl Storage for X` blocks keep compiling. New implementors can override `delete` if they want. This is the primary way to evolve traits without breaking downstream.

There's a subtle caveat here too - covered in the gray area section.

### Loosening generic bounds

The opposite of tightening. If you remove a bound, more types become valid. Nobody who was passing valid types before gets broken.

```rust
// v1.0.0
pub fn serialize<T: Serialize + Debug>(item: &T) -> String { /* ... */ }

// v1.1.0 - safe: removed Debug bound
pub fn serialize<T: Serialize>(item: &T) -> String { /* ... */ }
```

### Adding fields when private fields already exist

If your struct already has at least one private field, users can't construct it with struct literal syntax anyway. They have to use a constructor function or builder. So adding more fields - public or private - is fine.

```rust
// v1.0.0
pub struct Pool {
    pub max_size: usize,
    connections: Vec<Connection>,  // private
}

// v1.1.0 - safe: struct already had private fields
pub struct Pool {
    pub max_size: usize,
    pub idle_timeout: Duration,  // new public field
    connections: Vec<Connection>,
    pending: VecDeque<Waker>,    // new private field
}
```

Users were already calling `Pool::new()` or something similar. The constructor handles initialization of new fields internally. This is why many library authors include at least one private field in every public struct from day one - even if it's a `_private: ()` phantom field. It gives you room to grow.

### Making an unsafe function safe

If a function was `unsafe fn` and you make it `fn`, that's backwards compatible. Callers who wrapped it in `unsafe { }` blocks will get a warning (unused unsafe) but their code still compiles.

## The gray area: "possibly breaking" changes

[RFC 1105](https://rust-lang.github.io/rfcs/1105-api-evolution.html) established an important principle: **all major changes are breaking, but not all breaking changes are major.** Some changes can technically break downstream code, but the breakage is always "shallow" - fixable with a local, mechanical change. These are allowed in minor releases.

### New items and glob imports

Adding a public function named `connect` to your crate is a minor change. But if someone writes:

```rust
use your_crate::*;
use other_crate::*;
```

...and `other_crate` also exports `connect`, the new item creates an ambiguity error. This is technically breakage, but the fix is trivial - use an explicit import instead of a glob. Glob imports are generally discouraged in production code for exactly this reason, and the Rust project considers this acceptable breakage for a minor release.

### New trait methods shadowing inherent methods

When you add a defaulted method to a trait, it can shadow an inherent method that a downstream type happens to have with the same name:

```rust
// your crate v1.1.0 adds:
pub trait Widget {
    fn name(&self) -> &str { "unnamed" }
}

// downstream code:
struct Button;
impl Button {
    fn name(&self) -> &str { "button" }
}
impl your_crate::Widget for Button {}

// This used to call Button::name, still does (inherent methods win).
// But this:
fn print_name<W: Widget>(w: &W) {
    println!("{}", w.name()); // calls Widget::name, not Button::name
}
```

The behavior in generic contexts might change. This is "possibly breaking" - it's a judgment call for library maintainers.

### MSRV bumps

Changing your Minimum Supported Rust Version (say, from 1.70 to 1.75) technically breaks anyone still on 1.70. The Cargo SemVer reference lists this as "possibly breaking." Most of the ecosystem treats MSRV bumps as minor changes, though opinions vary. Some crates document an explicit MSRV policy in their README.

### Adding new inherent methods

If you add an inherent method to a type, and a user has imported a trait that defines a method with the same name, calling that method becomes ambiguous:

```rust
// your crate, v1.1.0
impl Config {
    pub fn keys(&self) -> Vec<&str> { /* ... */ }
}

// downstream
use some_trait::HasKeys; // also defines .keys()

let cfg = Config::new();
cfg.keys(); // which keys()? Ambiguous.
```

The fix is straightforward (fully qualified syntax: `HasKeys::keys(&cfg)` or `Config::keys(&cfg)`), so this is classified as a minor change despite the potential for breakage.

## The #[non_exhaustive] escape hatch

`#[non_exhaustive]` is the single most important attribute for semver-safe API design in Rust. It tells the compiler: "this type might grow in the future." Let's look at what it does for each kind of item.

### On enums

```rust
#[non_exhaustive]
pub enum Error {
    NotFound,
    PermissionDenied,
    Timeout,
}
```

Within your own crate, nothing changes - you can match exhaustively. But downstream code must include a wildcard arm:

```rust
match err {
    Error::NotFound => { /* ... */ }
    Error::PermissionDenied => { /* ... */ }
    Error::Timeout => { /* ... */ }
    _ => { /* handle future variants */ }
}
```

Now you can add `Error::RateLimited` in a minor release. The wildcard catches it. No breakage.

### On structs

```rust
#[non_exhaustive]
pub struct Config {
    pub host: String,
    pub port: u16,
}
```

Downstream code can no longer construct this with struct literal syntax:

```rust
// This fails outside the defining crate:
let cfg = Config {
    host: "localhost".into(),
    port: 5432,
};
```

They must use whatever constructor you provide (`Config::new()`, `Config::builder()`, etc.). And destructuring requires `..`:

```rust
let Config { host, port, .. } = cfg; // the `..` is mandatory
```

This means you can add `pub max_connections: u32` in a minor release. Users who destructure with `..` are fine. Users who construct via your API are fine. Nobody breaks.

### On enum variants

You can also mark individual variants:

```rust
pub enum Command {
    #[non_exhaustive]
    Get {
        key: String,
        timeout: Option<Duration>,
    },
    Delete {
        key: String,
    },
}
```

Now you can add fields to the `Get` variant (like `cache: bool`) without breaking match arms that use `..`.

### When to use it

The rule is simple: **mark it `#[non_exhaustive]` when you first create the type.** Adding `#[non_exhaustive]` to an existing type that didn't have it is itself a breaking change - it forces all downstream match arms to add wildcards and prevents struct literal construction. You can't retroactively add it without a major bump.

Standard library types like [`std::io::ErrorKind`](https://doc.rust-lang.org/std/io/enum.ErrorKind.html) use `#[non_exhaustive]`. So does [`http::StatusCode`](https://docs.rs/http/latest/http/status/struct.StatusCode.html) (via a different mechanism - it's a newtype around `u16`). The pattern is well-established.

### The hidden field trick (pre-non_exhaustive)

Before `#[non_exhaustive]` was stabilized (Rust 1.40), library authors used a private unit field:

```rust
pub struct Config {
    pub host: String,
    pub port: u16,
    _private: (),
}
```

The `_private: ()` field is zero-sized (no runtime cost) but prevents struct literal construction from outside the crate. You still see this in older crates. For new code, prefer `#[non_exhaustive]` - it's more explicit about intent.

## repr attributes and layout stability

If your struct or enum has a `repr` attribute, changing or removing it is a breaking change. This matters for FFI and for code that relies on specific memory layout.

```rust
// v1.0.0
#[repr(C)]
pub struct Header {
    pub magic: u32,
    pub version: u16,
    pub flags: u16,
}

// v1.1.0 - BREAKING: removed repr(C)
pub struct Header {
    pub magic: u32,
    pub version: u16,
    pub flags: u16,
}
```

Without `repr(C)`, Rust is free to reorder fields. Any FFI code, any `unsafe` pointer arithmetic, any code that transmutes or writes this struct to a file - all broken.

Going the other direction is safe though. Adding `repr(C)` to a struct that previously had default layout is a compatible change - it just pins down what the compiler was already free to choose.

The same logic applies to `repr(transparent)`, `repr(packed)`, and `repr(align)`. Removing them changes guarantees that downstream code may depend on.

## cargo-semver-checks: automated enforcement

Knowing the rules is one thing. Remembering to check them on every release is another. [cargo-semver-checks](https://github.com/obi1kenobi/cargo-semver-checks) automates this. It analyzes your crate's rustdoc JSON output and compares the public API surface between two versions, flagging semver violations.

### Installation and basic use

```bash
cargo install cargo-semver-checks --locked

# In your crate directory:
cargo semver-checks
```

By default, it compares your local code against the latest version published on crates.io. You can also specify baselines explicitly:

```bash
# Compare against a specific version
cargo semver-checks --baseline-version 1.4.0

# Compare against a git revision
cargo semver-checks --baseline-rev v1.4.0

# Compare against a local directory
cargo semver-checks --baseline-root ../my-crate-old
```

### What the output looks like

When it catches a violation, you get something like:

```
--- failure enum_variant_added: pub enum gained a new variant ---
   Description: A public enum without #[non_exhaustive] has a new variant.
        Change: variant `RateLimited` added to enum `Error`
    Suggestion: mark the enum #[non_exhaustive] before adding variants
      Location: src/lib.rs:42
      Reference: https://doc.rust-lang.org/cargo/reference/semver.html#enum-variant-added

Failed! 1 check failed, 126 checks passed.
```

It tells you what broke, where, and links to the specific section in the Cargo semver reference. The tool currently runs over 127 lints covering function signatures, trait changes, type removals, attribute changes, and more.

### Configuring lints

You can tune lint levels in your `Cargo.toml`:

```toml
[package.metadata.cargo-semver-checks.lints]
# Downgrade a specific lint to warning
function_must_use_added = "warn"

# Or suppress it entirely
enum_variant_added = "allow"
```

Workspace-level configuration is also supported:

```toml
[workspace.metadata.cargo-semver-checks.lints]
function_must_use_added = "warn"
```

### CI integration

The real power of cargo-semver-checks is in CI. You catch violations before merge, not after publish.

**GitHub Actions:**

```yaml
name: Semver Check

on:
  pull_request:

jobs:
  semver:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: dtolnay/rust-toolchain@stable
      - uses: obi1kenobi/cargo-semver-checks-action@v2
```

That's it. Three steps. Every PR that touches your public API gets checked automatically. If a PR introduces a breaking change, it fails the check, and the author can decide: is this intentional (bump major version) or accidental (fix the code)?

If you're already using conventional commits with [git-cliff](https://git-cliff.org) or [release-plz](https://release-plz.ievez.dev/) for automated releases, cargo-semver-checks adds a safety net. The commit message says "feat:" (minor), but the code change is actually breaking? The CI catches it.

### Current limitations

cargo-semver-checks is good but not omniscient. As of early 2026, it does not catch:

- Type changes in struct fields or function parameters (e.g., changing `u32` to `u64`)
- Lifetime or generic parameter changes on types
- Feature-gated API changes (it checks one feature combination at a time)
- Behavioral changes (different return values for same input)

It's worth knowing these gaps. The tool catches structural API changes - removed items, changed visibility, added required methods - but the type system edges still need manual review.

## Real-world examples

### hyper 0.14 to 1.0

The [hyper](https://hyper.rs) HTTP library went through one of the largest breaking changes in the Rust ecosystem when it released 1.0. The changes included:

- IO transport types switched from `tokio::io::{AsyncRead, AsyncWrite}` to hyper's own `hyper::rt::{Read, Write}` traits
- High-level server and client APIs were removed entirely, moved to [hyper-util](https://crates.io/crates/hyper-util)
- Request/Response types were replaced with types from the `http` crate
- OpenSSL support was dropped from the core crate

This was a deliberate, well-communicated major version bump. But the ripple effects were enormous. Every HTTP framework in the ecosystem - axum, warp, reqwest - had to update. The migration took months across the ecosystem, with many crates maintaining dual support for 0.14 and 1.0 during the transition.

### axum 0.7 to 0.8

[axum 0.8](https://tokio.rs/blog/2025-01-01-announcing-axum-0-8-0) changed path parameter syntax from `/:id` to `/{id}`. That's a one-line change per route, but it breaks every route definition in every axum project. They also removed the `#[async_trait]` requirement from extractors by leveraging return-position `impl Trait` in traits (stabilized in Rust 1.75). Cleaner code, but every custom extractor impl needed updating.

Both of these were intentional improvements that justified a version bump. The lesson: even well-designed breaking changes have a cost proportional to your user base.

## API design patterns for forward compatibility

Knowing what's breaking is half the battle. The other half is designing APIs that give you room to evolve without breaking changes.

### Use #[non_exhaustive] on public enums from day one

```rust
#[non_exhaustive]
pub enum CachePolicy {
    NoCache,
    MaxAge(Duration),
    Forever,
}
```

Future you can add `Stale(Duration)` in a minor release.

### Use builders instead of constructors with many parameters

```rust
pub struct ClientConfig {
    timeout: Duration,
    retries: u32,
    base_url: String,
}

impl ClientConfig {
    pub fn new(base_url: impl Into<String>) -> Self {
        Self {
            timeout: Duration::from_secs(30),
            retries: 3,
            base_url: base_url.into(),
        }
    }

    pub fn timeout(mut self, timeout: Duration) -> Self {
        self.timeout = timeout;
        self
    }

    pub fn retries(mut self, retries: u32) -> Self {
        self.retries = retries;
        self
    }
}
```

Every new field gets a new builder method with a default. Zero breakage.

### Seal traits you don't want others to implement

If a trait is meant for your crate's internal use - users call methods on it but never implement it themselves - seal it:

```rust
mod private {
    pub trait Sealed {}
}

pub trait Transport: private::Sealed {
    fn send(&self, data: &[u8]) -> Result<(), Error>;
}

// Only your crate can impl Sealed, so only your crate can impl Transport
impl private::Sealed for HttpTransport {}
impl Transport for HttpTransport { /* ... */ }
```

Now you can add methods to `Transport` without default implementations. No one outside your crate can implement the trait, so no one's `impl` block breaks. You see this pattern in [tower::Service](https://docs.rs/tower/latest/tower/trait.Service.html) (though tower doesn't seal it - they chose to allow external impls), and extensively in the standard library's internal traits.

### Prefer returning impl Trait over concrete types

```rust
// Fragile: locks you into returning exactly Vec<Item>
pub fn items(&self) -> Vec<Item> { /* ... */ }

// Flexible: you can change the underlying collection later
pub fn items(&self) -> impl Iterator<Item = &Item> { /* ... */ }
```

With `impl Iterator`, you can switch from `Vec` to `BTreeSet` internally without changing the public API.

### Make struct fields private with accessor methods

```rust
pub struct Stats {
    requests: u64,
    errors: u64,
    latency_ms: f64,
}

impl Stats {
    pub fn requests(&self) -> u64 { self.requests }
    pub fn errors(&self) -> u64 { self.errors }
    pub fn latency_ms(&self) -> f64 { self.latency_ms }
    pub fn error_rate(&self) -> f64 {
        if self.requests == 0 { 0.0 }
        else { self.errors as f64 / self.requests as f64 }
    }
}
```

You can add fields, rename internal fields, change internal types - none of it is visible to users. The accessor methods are your stable API surface.

## Versioning strategy for pre-1.0 crates

One more thing worth mentioning. Cargo treats `0.x.y` versions differently from `1.x.y`. For `0.x` crates, the **minor** version is treated as the breaking-change indicator. `0.1.x` to `0.2.0` is a breaking change. `0.1.0` to `0.1.1` is a compatible change.

This trips people up. `0.0.x` is even stricter - every patch bump is potentially breaking. Here's how it works:

| Version range | Compatible updates |
|---|---|
| `1.2.3` | `>=1.2.3, <2.0.0` |
| `0.2.3` | `>=0.2.3, <0.3.0` |
| `0.0.3` | `>=0.0.3, <0.0.4` |

Many production crates sit at `0.x` for years (looking at you, `rand 0.8`). That's fine - it signals that the API is still evolving. But be aware of how Cargo resolves those ranges. If you're at `0.4.2` and publish `0.5.0`, that's a major bump from Cargo's perspective.

## Wrapping up

Semver in Rust isn't just a convention - the type system enforces it at compile time. The compiler becomes your semver auditor whether you like it or not. Downstream code either compiles or it doesn't, and there's no wiggle room.

The practical takeaways:

1. **Adding anything to a public struct or enum is breaking** unless you've planned ahead with `#[non_exhaustive]` or private fields
2. **Adding trait methods is only safe with default implementations** - and even then, watch for name collisions
3. **Use `#[non_exhaustive]` on public enums and structs from day one** - you can't add it later without a major bump
4. **Run cargo-semver-checks in CI** - humans forget rules, tools don't
5. **Design for evolution**: builders, sealed traits, private fields with accessors, `impl Trait` returns

The Cargo [SemVer reference](https://doc.rust-lang.org/cargo/reference/semver.html) and [RFC 1105](https://rust-lang.github.io/rfcs/1105-api-evolution.html) are the canonical sources. If you maintain a public crate and haven't read them, set aside an hour. The rules are more nuanced than you'd expect, and knowing the gray areas - the "possibly breaking" category - is what separates a crate that evolves gracefully from one that requires a major version bump every other month.
