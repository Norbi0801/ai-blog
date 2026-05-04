+++
title = "Feature flags in Rust - conditional compilation with cfg"
date = 2025-09-13
description = "How Rust's cfg system, Cargo features, and conditional compilation actually work - from compiler internals to practical patterns for organizing optional functionality."

[taxonomies]
tags = ["rust", "cargo", "conditional-compilation", "tooling"]
+++

Most Rust developers first encounter `cfg` through `#[cfg(test)]` - the attribute that gates test modules so they don't end up in your release binary. But the conditional compilation system goes much deeper than test gating. It's how tokio lets you pick exactly which subsystems to compile. It's how serde makes `derive` optional. It's how the entire `no_std` ecosystem works.

The `cfg` system is Rust's answer to C's `#ifdef`, but integrated into the language grammar rather than being a text preprocessor. When a `cfg` predicate evaluates to false, the annotated item is removed from the AST before type-checking even starts. No dead code in the binary, no runtime branching, no cost.

This post covers the full picture: how `cfg` works at the compiler level, how Cargo features map to `cfg` flags, how to structure optional functionality in your crates, and the footgun that catches everyone - feature unification.

<!-- more -->

## What the compiler actually does with `#[cfg(...)]`

When you write `#[cfg(feature = "sqlite")]` on a function, the compiler doesn't generate an if-else branch. It does source-level elimination. The item either exists in the AST or it doesn't.

```rust
#[cfg(feature = "sqlite")]
fn connect_sqlite(path: &str) -> Connection {
    // ...
}
```

Cargo translates features into `rustc` flags. When you build with `--features sqlite`, Cargo passes `--cfg feature="sqlite"` to `rustc`. The compiler then evaluates every `#[cfg(...)]` predicate during the parsing/expansion phase. If the predicate is true, the attribute is stripped and the item stays. If false, the entire item - function, struct, module, impl block, whatever - is erased from the AST. It never reaches type-checking. It never reaches codegen. It doesn't exist.

This is fundamentally different from what you might do in other languages:

```rust
// This is NOT how cfg works. Don't do this.
fn connect(path: &str) -> Connection {
    if cfg!(feature = "sqlite") {
        // sqlite path
    } else {
        // fallback path
    }
}
```

Wait - `cfg!()` (with the exclamation mark) is actually valid Rust, but it does something different. It's a macro that evaluates to `true` or `false` at compile time. Both branches still need to type-check. The compiler might optimize away the dead branch, but it has to be valid code. `#[cfg(...)]` removes code before the compiler even looks at it.

Here's a concrete example showing the difference:

```rust
// This compiles fine even if "sqlite" is not enabled.
// The function simply doesn't exist.
#[cfg(feature = "sqlite")]
fn sqlite_only() -> SqliteConnection {
    SqliteConnection::new()
}

// This FAILS to compile if SqliteConnection doesn't exist,
// regardless of the feature flag. Both branches must type-check.
fn maybe_sqlite() -> Option<Box<dyn Connection>> {
    if cfg!(feature = "sqlite") {
        Some(Box::new(SqliteConnection::new())) // Error if type doesn't exist
    } else {
        None
    }
}
```

You can inspect which cfg flags the compiler sets by default:

```bash
$ rustc --print cfg
debug_assertions
panic="unwind"
target_arch="x86_64"
target_endian="little"
target_env="gnu"
target_family="unix"
target_os="linux"
target_pointer_width="64"
target_vendor="unknown"
unix
```

And for a different target:

```bash
$ rustc --print cfg --target aarch64-apple-darwin
target_arch="aarch64"
target_endian="little"
target_env=""
target_family="unix"
target_os="macos"
target_pointer_width="64"
target_vendor="apple"
unix
```

These are the built-in options. Features you define in Cargo.toml get added on top via `--cfg feature="name"`.

## cfg predicates - the boolean logic

cfg supports three combinators: `all()`, `any()`, and `not()`. They compose like you'd expect:

```rust
// Only on 64-bit Linux
#[cfg(all(target_os = "linux", target_pointer_width = "64"))]
fn linux_64_only() { /* ... */ }

// On any Unix OR if the "portable" feature is enabled
#[cfg(any(unix, feature = "portable"))]
fn unix_or_portable() { /* ... */ }

// Everything except Windows
#[cfg(not(windows))]
fn no_windows() { /* ... */ }

// Complex: 64-bit Unix with either the sqlite or postgres feature
#[cfg(all(
    unix,
    target_pointer_width = "64",
    any(feature = "sqlite", feature = "postgres")
))]
fn specific_setup() { /* ... */ }
```

You can also use `cfg` on more than just functions. It works on struct fields, enum variants, impl blocks, `use` statements, modules, trait implementations - basically any item:

```rust
struct Config {
    host: String,
    port: u16,
    #[cfg(feature = "tls")]
    tls_cert: PathBuf,
    #[cfg(feature = "tls")]
    tls_key: PathBuf,
}

// The impl block only exists when "metrics" is enabled
#[cfg(feature = "metrics")]
impl Config {
    fn metrics_endpoint(&self) -> String {
        format!("{}:{}/metrics", self.host, self.port)
    }
}
```

One thing to be careful about: gating struct fields behind features. If your struct is public, consumers can't construct it without knowing which features are enabled. We'll revisit this as an anti-pattern later.

## The full list of built-in cfg options

The Rust compiler sets these automatically based on the compilation target:

| Option | Values | Example |
|--------|--------|---------|
| `target_arch` | `"x86"`, `"x86_64"`, `"arm"`, `"aarch64"`, `"riscv32"`, `"wasm32"`, ... | `#[cfg(target_arch = "aarch64")]` |
| `target_os` | `"linux"`, `"windows"`, `"macos"`, `"ios"`, `"android"`, `"none"`, ... | `#[cfg(target_os = "linux")]` |
| `target_family` | `"unix"`, `"windows"`, `"wasm"` | `#[cfg(target_family = "unix")]` |
| `unix` / `windows` | boolean (no value) | `#[cfg(unix)]` |
| `target_env` | `""`, `"gnu"`, `"msvc"`, `"musl"` | `#[cfg(target_env = "musl")]` |
| `target_endian` | `"little"`, `"big"` | `#[cfg(target_endian = "big")]` |
| `target_pointer_width` | `"16"`, `"32"`, `"64"` | `#[cfg(target_pointer_width = "32")]` |
| `target_vendor` | `"apple"`, `"pc"`, `"unknown"` | `#[cfg(target_vendor = "apple")]` |
| `target_feature` | `"avx"`, `"avx2"`, `"sse2"`, `"neon"`, ... | `#[cfg(target_feature = "avx2")]` |
| `target_has_atomic` | `"8"`, `"16"`, `"32"`, `"64"`, `"ptr"` | `#[cfg(target_has_atomic = "64")]` |
| `panic` | `"unwind"`, `"abort"` | `#[cfg(panic = "unwind")]` |
| `test` | boolean | `#[cfg(test)]` |
| `debug_assertions` | boolean | `#[cfg(debug_assertions)]` |
| `proc_macro` | boolean | `#[cfg(proc_macro)]` |
| `doc` | boolean | `#[cfg(doc)]` |
| `doctest` | boolean | `#[cfg(doctest)]` |

If you've read the [embedded Rust post](/blog/embedded-rust-from-zero-to-blinky), you'll recall that `#![no_std]` drops the standard library. The interplay with `cfg` is key: many crates use `#[cfg(feature = "std")]` to gate std-dependent code, keeping the core functionality available for `no_std` targets.

## cfg(test) - the one everyone knows

`#[cfg(test)]` gates code that should only exist during `cargo test`. The compiler sets the `test` cfg when you build with `--test`. This is how the standard test module pattern works:

```rust
pub fn add(a: i32, b: i32) -> i32 {
    a + b
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_add() {
        assert_eq!(add(2, 3), 5);
    }
}
```

The entire `mod tests` block - the imports, the test functions, any helper utilities inside it - is stripped from non-test builds. This matters for binary size and compile time in release builds.

You can also use `cfg(test)` outside of test modules. A pattern I've seen in database code:

```rust
pub struct DbPool {
    inner: Pool<Postgres>,
    #[cfg(test)]
    pub test_transaction: Option<Transaction>,
}

impl DbPool {
    #[cfg(test)]
    pub fn with_test_transaction(tx: Transaction) -> Self {
        Self {
            inner: /* ... */,
            test_transaction: Some(tx),
        }
    }
}
```

This adds test-only fields and constructors that vanish in production builds. Useful, but be careful - if your test code uses a fundamentally different code path, you're not really testing the production behavior.

The `debug_assertions` cfg is related but different. It's on by default in debug builds (without `--release`) but off in release builds. Good for expensive runtime checks:

```rust
fn process_batch(items: &[Item]) {
    // This assertion disappears in release builds
    #[cfg(debug_assertions)]
    {
        for item in items {
            assert!(item.is_valid(), "invalid item: {:?}", item);
        }
    }

    // Actual processing...
}
```

## Cargo features - the `[features]` table

This is where conditional compilation gets practical. Features are how crate authors expose optional functionality to users. They're defined in `Cargo.toml` and translated to `--cfg` flags by Cargo.

Here's a real-world example for a hypothetical storage crate:

```toml
[package]
name = "storagebox"
version = "0.1.0"
edition = "2021"

[features]
default = ["memory"]
memory = []
sqlite = ["dep:rusqlite"]
postgres = ["dep:tokio-postgres", "dep:deadpool-postgres"]
full = ["memory", "sqlite", "postgres"]

[dependencies]
rusqlite = { version = "0.34", optional = true }
tokio-postgres = { version = "0.7", optional = true }
deadpool-postgres = { version = "0.14", optional = true }
```

A few things happening here:

**`default`** lists features enabled when the user writes `storagebox = "0.1"` without specifying anything. Here, the in-memory backend is on by default. Users can opt out with `default-features = false`.

**`dep:` syntax** (stable since Rust 1.60) prevents optional dependencies from implicitly creating features. Without `dep:`, adding `rusqlite = { optional = true }` automatically creates a `rusqlite` feature. That exposes an internal dependency name as your public API - not great. The `dep:` prefix says "this is an internal dependency reference, not a public feature name."

**`full`** is a convenience feature that enables everything. Tokio popularized this pattern - `tokio = { features = ["full"] }` is easier than listing 12 individual features.

**Feature-gated code** then uses `#[cfg(feature = "...")]`:

```rust
#[cfg(feature = "memory")]
pub mod memory;

#[cfg(feature = "sqlite")]
pub mod sqlite;

#[cfg(feature = "postgres")]
pub mod postgres;

pub trait Storage: Send + Sync {
    fn get(&self, key: &str) -> Result<Option<Vec<u8>>>;
    fn set(&self, key: &str, value: &[u8]) -> Result<()>;
    fn delete(&self, key: &str) -> Result<bool>;
}

// Each module implements Storage for its backend
#[cfg(feature = "sqlite")]
impl Storage for sqlite::SqliteStorage {
    // ...
}
```

Users then pick their backend:

```toml
# Just sqlite, no memory backend
storagebox = { version = "0.1", default-features = false, features = ["sqlite"] }

# Everything
storagebox = { version = "0.1", features = ["full"] }
```

## cfg_attr - conditional attributes

Sometimes you don't want to gate an entire item - you want to conditionally apply an attribute. That's `cfg_attr`:

```rust
#[cfg_attr(feature = "serde", derive(Serialize, Deserialize))]
pub struct Config {
    pub host: String,
    pub port: u16,
}
```

If `serde` is enabled, this expands to `#[derive(Serialize, Deserialize)]`. If not, the derive is simply absent. The struct exists either way.

The syntax supports multiple attributes in a single `cfg_attr`:

```rust
#[cfg_attr(feature = "serde", derive(Serialize, Deserialize), serde(rename_all = "camelCase"))]
pub struct ApiResponse {
    pub status_code: u16,
    pub error_message: Option<String>,
}
```

When the `serde` feature is enabled, both `derive(Serialize, Deserialize)` and `serde(rename_all = "camelCase")` are applied.

A common use case for `cfg_attr` is controlling docs.rs builds. If you want docs.rs to show feature-gated items with badges:

```rust
#![cfg_attr(docsrs, feature(doc_cfg))]

#[cfg(feature = "sqlite")]
#[cfg_attr(docsrs, doc(cfg(feature = "sqlite")))]
pub mod sqlite {
    //! SQLite storage backend.
    //!
    //! Enable the `sqlite` feature to use this module.
}
```

Then in `Cargo.toml`:

```toml
[package.metadata.docs.rs]
all-features = true
rustdoc-args = ["--cfg", "docsrs"]
```

This makes docs.rs build with all features enabled and annotate feature-gated items so users know which feature they need.

Another practical `cfg_attr` pattern - platform-specific link attributes:

```rust
#[cfg_attr(target_os = "linux", link(name = "ssl"))]
#[cfg_attr(target_os = "macos", link(name = "Security", kind = "framework"))]
extern "C" {
    // ...
}
```

## Feature unification - the footgun

This is the thing that bites everyone who designs features for the first time. The rule is simple but the consequences are not:

**When a dependency appears multiple times in the dependency graph, Cargo builds it once with the union of all requested features.**

Say you have this dependency graph:

```
your-app
├── crate-a  (depends on serde with features = ["derive"])
└── crate-b  (depends on serde with features = ["std"])
```

Cargo doesn't build serde twice - once with `derive` and once with `std`. It builds serde once with both `derive` and `std` enabled. This is feature unification. It ensures a single copy of each dependency in the final binary.

Why is this a footgun? Because it means **features must be additive**. Enabling a feature should only add functionality, never remove or change existing behavior. If feature A and feature B are mutually exclusive, some dependency graph out there will enable both simultaneously and break.

Here's a concrete scenario. Imagine a TLS crate with two backends:

```toml
[features]
openssl = ["dep:openssl-sys"]
rustls = ["dep:rustls"]
```

If you design these as mutually exclusive and use `cfg` to pick one:

```rust
// DON'T DO THIS
#[cfg(all(feature = "openssl", not(feature = "rustls")))]
fn connect() -> OpenSslStream { /* ... */ }

#[cfg(all(feature = "rustls", not(feature = "openssl")))]
fn connect() -> RustlsStream { /* ... */ }
```

This breaks when both features are enabled (which feature unification can cause). The `not(feature = "...")` guard silently removes both functions, and downstream code fails with cryptic "function not found" errors.

The correct pattern is to use `compile_error!` to give a clear message:

```rust
#[cfg(all(feature = "openssl", feature = "rustls"))]
compile_error!(
    "Features `openssl` and `rustls` are mutually exclusive. \
     Please enable only one."
);
```

But even this is fragile in a large dependency graph. The real solution is to design features so mutual exclusivity isn't needed. Look at how reqwest handles TLS backends - it uses internal (private) feature flags prefixed with `__` and a default that users can override:

```toml
[features]
default = ["default-tls"]
default-tls = ["dep:hyper-tls", "dep:native-tls-crate", "__tls"]
rustls-tls = ["dep:hyper-rustls", "dep:rustls", "__tls"]
__tls = []  # Internal flag: "some TLS backend is enabled"
```

Both can be enabled simultaneously because the actual TLS selection happens through the dependency tree, not through mutually exclusive cfg gates.

### Resolver v2 and when unification happens

Before Rust 2021 edition, feature unification was aggressive. Three problem areas:

1. **Dev-dependencies leaked features.** If your tests used `tokio = { features = ["full"] }` but your library only needed `tokio = { features = ["rt"] }`, the `full` features bled into the library build. Users of your library got features they didn't ask for.

2. **Platform-specific deps unified cross-platform.** A `[target.'cfg(windows)'.dependencies]` entry with specific features would affect Linux builds too.

3. **Build-dependencies shared features with normal deps.** If your build script needed `serde = { features = ["derive"] }` for code generation, that `derive` feature activated for your normal code too.

Feature resolver v2 (default since edition 2021, opt-in since Rust 1.51) fixes all three:

```toml
[package]
edition = "2021"  # Implies resolver = "2"
# Or explicitly:
# resolver = "2"
```

With resolver v2:
- Dev-dependency features only unify when actually building tests/examples/benchmarks
- Platform-specific dependency features are ignored for targets not being built
- Build-dependencies and proc-macros get separate feature sets from normal dependencies

The trade-off: resolver v2 may compile the same crate multiple times with different feature sets. If your build-dependencies use `serde` with `derive` and your normal code uses `serde` without it, you get two compilations of serde. Use `cargo tree --duplicates` to detect this.

## Inspecting feature propagation

When something goes wrong with features - wrong items appearing or disappearing, unexpected compilation errors - you need to see what's actually enabled. Cargo has good tools for this:

```bash
# Show the full dependency tree with features
$ cargo tree -e features
my-app v0.1.0
├── serde v1.0.219
│   └── serde_derive v1.0.219 (proc-macro)
├── tokio v1.45.1
│   ├── bytes v1.10.1
│   ├── mio v1.0.4
│   │   └── libc v0.2.172
│   ├── parking_lot v0.12.3
...

# Compact view: package + its features
$ cargo tree -f "{p} {f}"
my-app v0.1.0
serde v1.0.219 default,derive,std
tokio v1.45.1 default,fs,full,io-std,io-util,macros,...

# Why is a specific feature enabled on a dependency?
$ cargo tree -e features -i serde
serde v1.0.219
├── serde feature "default"
│   └── my-app v0.1.0
├── serde feature "derive"
│   └── my-app v0.1.0
└── serde feature "std"
    └── serde feature "default"
```

That last command (`-i` for inverted) is gold for debugging. It shows you exactly who is enabling each feature on a dependency.

## --check-cfg - catching typos at compile time

Since Rust 1.80, the compiler checks your `cfg` predicates against a list of expected values. This means typos get caught:

```rust
#[cfg(feature = "sqllite")]  // Typo!
fn connect() { /* ... */ }
```

```
warning: unexpected `cfg` condition value: `sqllite`
 --> src/lib.rs:1:7
  |
1 | #[cfg(feature = "sqllite")]
  |       ^^^^^^^^^^^^^^^^^^^
  |
  = note: expected values for `feature` are: `default`, `memory`, `sqlite`, `postgres`, `full`
  = help: did you mean: `sqlite`?
```

Cargo automatically passes your declared features to `--check-cfg`. But if you use custom cfg values (not features), you need to declare them. Two ways:

In a build script:

```rust
// build.rs
fn main() {
    println!("cargo::rustc-check-cfg=cfg(has_jemalloc)");
    if has_jemalloc() {
        println!("cargo::rustc-cfg=has_jemalloc");
    }
}
```

Or statically in `Cargo.toml` (cleaner for known values):

```toml
[lints.rust]
unexpected_cfgs = { level = "warn", check-cfg = ['cfg(loom)', 'cfg(fuzzing)'] }
```

This is especially useful for CI testing tools like [loom](https://github.com/tokio-rs/loom) that set custom cfg flags. Without declaring them, you'll get warnings on every `#[cfg(loom)]` annotation.

## Patterns: how popular crates organize features

Looking at how well-maintained crates structure their features teaches more than any guide.

### Tokio - granular opt-in, empty default

```toml
[features]
default = []

# Individual subsystems
fs = []
io-util = ["bytes"]
io-std = []
macros = ["tokio-macros"]
net = ["libc", "mio/os-poll", "mio/os-ext", "mio/net", "socket2", "windows-sys/..."]
process = ["bytes", "libc", "mio/os-poll", "mio/os-ext", "mio/net", "signal-hook-registry", "windows-sys/..."]
rt = []
rt-multi-thread = ["rt"]
signal = ["libc", "mio/os-poll", "mio/os-ext", "mio/net", "signal-hook-registry", "windows-sys/..."]
sync = []
time = []

# Convenience
full = ["fs", "io-util", "io-std", "macros", "net", "parking_lot", "process", "rt", "rt-multi-thread", "signal", "sync", "time"]
```

Nothing enabled by default. Users pick exactly what they need, or use `full` for convenience. This keeps compile times down for projects that only need, say, the sync primitives and timers.

Features form a DAG: `rt-multi-thread` requires `rt`. `process` requires `signal-hook-registry`. Cargo handles transitive activation automatically.

### Serde - std by default, opt-out for no_std

```toml
[features]
default = ["std"]
std = ["alloc", "serde_derive/std"]
alloc = []
derive = ["serde_derive"]
rc = []
unstable = []
```

The layering here is `core` (no features) < `alloc` (heap types) < `std` (full standard library). This is the standard pattern for crates that support `no_std`:

```toml
# Full std support (default)
serde = "1.0"

# no_std with heap allocation (Vec, String, Box)
serde = { version = "1.0", default-features = false, features = ["alloc"] }

# Bare no_std (only fixed-size types)
serde = { version = "1.0", default-features = false }
```

The `rc` feature opts into `Serialize`/`Deserialize` impls for `Rc<T>` and `Arc<T>`. It's separate because shared ownership has implications for deserialization (you might unintentionally create independent copies of data that was previously shared). Making it opt-in forces users to think about this.

### Reqwest - `dep:` syntax and private features

```toml
[features]
default = ["default-tls", "charset", "http2", "macos-system-configuration"]
default-tls = ["dep:hyper-tls", "dep:native-tls-crate", "__tls"]
rustls-tls = ["dep:hyper-rustls", "dep:rustls", "__tls"]

# Private features (not part of public API)
__tls = []
__rustls = []
```

The `__` prefix convention signals "this is an internal implementation detail, don't depend on it." It's not enforced by Cargo - anyone *can* enable `__tls` - but it communicates intent. The `dep:` syntax keeps dependency names out of the feature namespace.

## The no_std pattern

If you've read the [embedded Rust post](/blog/embedded-rust-from-zero-to-blinky), you know that `#![no_std]` drops the standard library. The standard pattern for crates that want to support both std and no_std environments:

```rust
// lib.rs
#![cfg_attr(not(feature = "std"), no_std)]

#[cfg(feature = "alloc")]
extern crate alloc;

// Re-export the right types depending on available features
#[cfg(feature = "std")]
use std::vec::Vec;

#[cfg(all(not(feature = "std"), feature = "alloc"))]
use alloc::vec::Vec;
```

```toml
[features]
default = ["std"]
std = ["alloc"]
alloc = []
```

The key insight: `std` is a positive opt-in feature that *adds* standard library support. It's not `no_std = []` that *removes* it. This follows the additivity rule - enabling `std` adds functionality (the full standard library), it doesn't change existing behavior.

## Anti-patterns

After seeing the patterns, here are the traps.

### Mutually exclusive features

Already covered above, but worth repeating: don't do this. Feature unification will enable both, and your code will break or silently do the wrong thing. If you absolutely must have mutual exclusion, use `compile_error!` to fail fast:

```rust
#[cfg(all(feature = "backend-a", feature = "backend-b"))]
compile_error!("Only one backend can be enabled at a time.");
```

But question whether you actually need mutual exclusion. Usually you can restructure to avoid it.

### Feature-gated public struct fields

```rust
// This is painful for users
pub struct Config {
    pub host: String,
    #[cfg(feature = "tls")]
    pub cert_path: PathBuf,
}
```

Now constructing `Config` requires knowing which features are enabled:

```rust
// With "tls" enabled:
let config = Config { host: "localhost".into(), cert_path: "cert.pem".into() };

// Without "tls":
let config = Config { host: "localhost".into() };
```

If a user's code works without "tls" and then another dependency enables it through feature unification, their code breaks - they're suddenly missing a required field. Use a builder pattern or make feature-gated fields `Option<T>` in a separate struct.

### Negative feature flags

```toml
[features]
no-std = []        # Bad: negative flag
disable-logging = [] # Bad: removing functionality
```

Negative flags fight the additivity rule. If `no-std` is supposed to *remove* std support, what happens when one crate in your dependency graph enables it and another doesn't? Feature unification enables it, and now std is removed for everyone.

The fix: make it positive. Instead of `no-std`, use `std` as the feature that *adds* std support, and make it a default feature. Users who want no_std can opt out with `default-features = false`.

### Features that change behavior instead of adding it

```rust
// DON'T: feature changes return type
#[cfg(feature = "async")]
pub async fn fetch(url: &str) -> Result<Response> { /* ... */ }

#[cfg(not(feature = "async"))]
pub fn fetch(url: &str) -> Result<Response> { /* ... */ }
```

If feature unification enables `async`, downstream code expecting the sync version breaks. Instead, provide both under different names:

```rust
pub fn fetch(url: &str) -> Result<Response> { /* ... */ }

#[cfg(feature = "async")]
pub async fn fetch_async(url: &str) -> Result<Response> { /* ... */ }
```

The sync version always exists. The async version is additive.

### Too many features

Every feature you add doubles the testing matrix. 5 features = 32 combinations. 10 features = 1,024. You can't test all of them. In practice, test at least:

- No features (`default-features = false`)
- Default features only
- All features (`--all-features`)
- Each individually significant feature in isolation

If you find yourself with 15+ features, consider whether some of them should be separate crates instead.

## Practical example: a crate with sqlite and full features

Putting it all together. Here's a realistic `Cargo.toml` for a storage library:

```toml
[package]
name = "kv-store"
version = "0.3.0"
edition = "2021"
description = "A key-value store with pluggable backends"

[features]
default = ["memory"]
memory = []
sqlite = ["dep:rusqlite"]
postgres = ["dep:tokio-postgres", "dep:deadpool-postgres"]
serde = ["dep:serde", "dep:serde_json"]
full = ["memory", "sqlite", "postgres", "serde"]

[dependencies]
thiserror = "2.0"

# Optional backends
rusqlite = { version = "0.34", optional = true, features = ["bundled"] }
tokio-postgres = { version = "0.7", optional = true }
deadpool-postgres = { version = "0.14", optional = true }

# Optional serialization
serde = { version = "1.0", optional = true, features = ["derive"] }
serde_json = { version = "1.0", optional = true }

[dev-dependencies]
tokio = { version = "1", features = ["full"] }
tempfile = "3.16"

[package.metadata.docs.rs]
all-features = true
rustdoc-args = ["--cfg", "docsrs"]
```

And the corresponding `lib.rs`:

```rust
#![cfg_attr(docsrs, feature(doc_cfg))]

use thiserror::Error;

#[derive(Error, Debug)]
pub enum StoreError {
    #[error("key not found: {0}")]
    NotFound(String),
    #[error("backend error: {0}")]
    Backend(String),
    #[cfg(feature = "sqlite")]
    #[error("sqlite error: {0}")]
    Sqlite(#[from] rusqlite::Error),
    #[cfg(feature = "postgres")]
    #[error("postgres error: {0}")]
    Postgres(#[from] tokio_postgres::Error),
}

/// The core trait all backends implement.
pub trait Store: Send + Sync {
    fn get(&self, key: &str) -> Result<Option<Vec<u8>>, StoreError>;
    fn set(&self, key: &str, value: &[u8]) -> Result<(), StoreError>;
    fn delete(&self, key: &str) -> Result<bool, StoreError>;
    fn keys(&self) -> Result<Vec<String>, StoreError>;
}

#[cfg(feature = "memory")]
#[cfg_attr(docsrs, doc(cfg(feature = "memory")))]
pub mod memory;

#[cfg(feature = "sqlite")]
#[cfg_attr(docsrs, doc(cfg(feature = "sqlite")))]
pub mod sqlite;

#[cfg(feature = "postgres")]
#[cfg_attr(docsrs, doc(cfg(feature = "postgres")))]
pub mod postgres;

// Re-export backends at the crate root for convenience
#[cfg(feature = "memory")]
pub use memory::MemoryStore;

#[cfg(feature = "sqlite")]
pub use sqlite::SqliteStore;

#[cfg(feature = "postgres")]
pub use postgres::PostgresStore;

// Serde support: add serialize/deserialize helpers when enabled
#[cfg(feature = "serde")]
#[cfg_attr(docsrs, doc(cfg(feature = "serde")))]
pub mod serialized {
    use super::*;
    use serde::{de::DeserializeOwned, Serialize};

    /// Store a serializable value as JSON.
    pub fn set_json<S: Store, T: Serialize>(
        store: &S,
        key: &str,
        value: &T,
    ) -> Result<(), StoreError> {
        let bytes = serde_json::to_vec(value)
            .map_err(|e| StoreError::Backend(e.to_string()))?;
        store.set(key, &bytes)
    }

    /// Retrieve and deserialize a JSON value.
    pub fn get_json<S: Store, T: DeserializeOwned>(
        store: &S,
        key: &str,
    ) -> Result<Option<T>, StoreError> {
        match store.get(key)? {
            Some(bytes) => {
                let value = serde_json::from_slice(&bytes)
                    .map_err(|e| StoreError::Backend(e.to_string()))?;
                Ok(Some(value))
            }
            None => Ok(None),
        }
    }
}
```

Note the structure: the `Store` trait is always available. Backends are feature-gated modules. Serde helpers are additive - they don't change any existing API, they add new functions. Everything follows the additivity principle.

## Testing feature combinations in CI

You can't test all 2^N combinations, but you should test the important ones. A GitHub Actions workflow:

```yaml
jobs:
  test-features:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        features:
          - ""                                    # no default features
          - "default"                             # default features
          - "sqlite"                              # single backend
          - "postgres"                            # single backend
          - "memory,serde"                        # cross-feature interaction
          - "full"                                # everything
    steps:
      - uses: actions/checkout@v4
      - uses: dtolnay/rust-toolchain@stable
      - name: Test with features '${{ matrix.features }}'
        run: |
          if [ -z "${{ matrix.features }}" ]; then
            cargo test --no-default-features
          elif [ "${{ matrix.features }}" = "default" ]; then
            cargo test
          else
            cargo test --no-default-features --features "${{ matrix.features }}"
          fi
```

For local development, a quick smoke test:

```bash
# Does it compile with nothing?
cargo check --no-default-features

# Does it compile with everything?
cargo check --all-features

# Run the full test suite
cargo test --all-features
```

If `cargo check --no-default-features` fails, you have a hard dependency on something that should be optional. This is the most common bug in feature-gated code.

## cfg in build scripts

Build scripts (`build.rs`) can set custom cfg flags based on anything they can detect at build time:

```rust
// build.rs
fn main() {
    // Declare the cfg so --check-cfg doesn't warn
    println!("cargo::rustc-check-cfg=cfg(has_jemalloc)");

    // Detect jemalloc at build time
    if pkg_config::probe_library("jemalloc").is_ok() {
        println!("cargo::rustc-cfg=has_jemalloc");
    }

    // Set cfg based on environment variable
    println!("cargo::rustc-check-cfg=cfg(custom_allocator)");
    if std::env::var("USE_CUSTOM_ALLOCATOR").is_ok() {
        println!("cargo::rustc-cfg=custom_allocator");
    }
}
```

Then in your code:

```rust
#[cfg(has_jemalloc)]
use tikv_jemallocator::Jemalloc;

#[cfg(has_jemalloc)]
#[global_allocator]
static GLOBAL: Jemalloc = Jemalloc;
```

Note the `cargo::rustc-check-cfg` directive (stabilized in Rust 1.80, replacing the older `cargo:rustc-check-cfg` with a single colon). Without it, the compiler warns about unexpected cfg values.

## Wrapping up

The cfg system is one of those Rust features that seems simple on the surface but has real depth. The key takeaways:

**`#[cfg(...)]` is compile-time elimination, not runtime branching.** Gated code doesn't exist in the binary. This is zero-cost in the truest sense.

**Features must be additive.** Feature unification will enable combinations you didn't plan for. Design accordingly. Use `dep:` syntax to keep your feature namespace clean.

**Use resolver v2** (edition 2021+). It prevents dev-dependencies and build-dependencies from leaking features into your library consumers.

**`cfg_attr` for conditional attributes.** Don't gate an entire struct behind a feature when you only need conditional derives.

**`--check-cfg` catches typos.** Since Rust 1.80, misspelled feature names are caught at compile time.

**Test the edges:** `--no-default-features`, `--all-features`, and each significant feature individually. If it doesn't compile with zero features, you have a bug.

The Cargo book's [features chapter](https://doc.rust-lang.org/cargo/reference/features.html) and the [conditional compilation reference](https://doc.rust-lang.org/reference/conditional-compilation.html) are the canonical sources. For a deeper treatment of the SemVer implications of features, the [Cargo SemVer reference](https://doc.rust-lang.org/cargo/reference/semver.html) documents exactly which feature changes are breaking.
