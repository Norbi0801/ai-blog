+++
title = "Rust conditional compilation beyond cfg"
date = 2026-01-20
description = "Past the basics of cfg: target_os, target_arch, features, custom cfg flags via build.rs, cfg_if, conditional dependencies, and testing platform-specific code without losing your mind."

[taxonomies]
tags = ["rust", "build", "cross-platform", "cargo"]
+++

Most Rust developers learn `#[cfg(test)]` on day one and `#[cfg(feature = "foo")]` somewhere around the time they ship a library. Then they hit a real problem - "I need this `unsafe` block on Linux but a different one on Windows, and on macOS I want a completely different code path that calls a system framework" - and discover that conditional compilation is a small language inside Rust with surprising depth.

This post is about that depth. We will cover the full set of stable cfg predicates, the `cfg!` macro vs the `#[cfg]` attribute, the `cfg_attr` form, custom cfg flags via `build.rs`, the `cfg_if` crate (and when you actually need it), conditional dependencies in `Cargo.toml`, how to test platform-specific code, and the new (since Rust 1.80) `cargo::rustc-check-cfg` mechanism that turns typos into hard errors.

<!-- more -->

## What cfg actually is

`cfg` is not a macro in the usual sense. It is a built-in compiler feature that prunes the AST before type checking. When you write:

```rust
#[cfg(target_os = "linux")]
fn read_proc_self_exe() -> std::io::Result<std::path::PathBuf> {
    std::fs::read_link("/proc/self/exe")
}
```

on macOS, the compiler does not type-check the body. The function vanishes from the AST entirely. This is why you can call platform-specific syscalls inside a `cfg`-gated function without `cargo check` complaining on a different host. The flip side is that you can hide bugs - a function only built on one OS will not get errors caught on another platform until CI runs there.

The configuration set the compiler uses lives in `--cfg` flags passed by Cargo. You can dump it with:

```bash
$ rustc --print cfg
debug_assertions
panic="unwind"
target_arch="aarch64"
target_endian="little"
target_env=""
target_family="unix"
target_os="macos"
target_pointer_width="64"
target_vendor="apple"
unix
```

Every `#[cfg(...)]` you write resolves against this set. The predicate language is small: bare names (`unix`, `test`), `name = "value"` pairs, and three combinators: `all(...)`, `any(...)`, `not(...)`. That's it. You can nest them.

```rust
#[cfg(all(unix, not(target_os = "macos"), any(target_arch = "x86_64", target_arch = "aarch64")))]
mod linux_bsd_64bit;
```

## The full menu of stable cfg predicates

You should know these by heart. They are all stable and documented in the [Rust Reference](https://doc.rust-lang.org/reference/conditional-compilation.html):

| Predicate | Example values |
|-----------|----------------|
| `target_arch` | `"x86_64"`, `"aarch64"`, `"wasm32"`, `"riscv64"` |
| `target_os` | `"linux"`, `"macos"`, `"windows"`, `"freebsd"`, `"android"`, `"ios"` |
| `target_family` | `"unix"`, `"windows"`, `"wasm"` |
| `target_env` | `"gnu"`, `"musl"`, `"msvc"`, `""` |
| `target_endian` | `"little"`, `"big"` |
| `target_pointer_width` | `"16"`, `"32"`, `"64"` |
| `target_vendor` | `"apple"`, `"pc"`, `"unknown"` |
| `target_feature` | `"avx2"`, `"sse4.2"`, `"neon"` |
| `target_has_atomic` | `"8"`, `"16"`, `"32"`, `"64"`, `"ptr"` |
| `feature` | any name from `[features]` in your `Cargo.toml` |
| `test` | set when building tests |
| `debug_assertions` | set in non-release builds |
| `panic` | `"unwind"` or `"abort"` |
| `unix` / `windows` | shorthand for `target_family` |
| `proc_macro` | set when compiling a proc-macro crate |

Two of these get misused regularly. `debug_assertions` is **not** "is this a debug build" - it's tied to the `debug-assertions` profile setting, which defaults on for `dev`/`test` and off for `release`/`bench`, but you can override it. If you put `[profile.release] debug-assertions = true` in `Cargo.toml`, your release build will see `debug_assertions` set. Use it for asserts and bounds checks, not for "am I in dev mode."

`target_feature` is set per-target-CPU. `cfg(target_feature = "avx2")` is true if AVX2 is compiled in unconditionally - which usually means you passed `-C target-cpu=native` or `-C target-feature=+avx2`. For runtime detection (the common case for SIMD), use `is_x86_feature_detected!` instead.

If you are unsure what these resolve to in your current environment, check [cross-compilation in Rust](/blog/cross-compilation-in-rust---building-for-linux-from-macos/) for how target triples map to these values.

## #[cfg] vs cfg! - attribute vs macro

There are two ways to use cfg, and they do different things.

`#[cfg(...)]` is an attribute. It removes code at parse time. The removed code is never seen by typeck, never lowered to MIR, never reaches the linker.

`cfg!(...)` is a macro that expands to a `bool` literal. It returns `true` or `false` based on the same predicate, but **both branches of the surrounding code are compiled**.

```rust
fn ping_host(host: &str) -> std::io::Result<()> {
    let arg = if cfg!(target_os = "windows") { "-n" } else { "-c" };
    std::process::Command::new("ping").args([arg, "1", host]).status()?;
    Ok(())
}
```

This works because both string literals exist on every platform. Compare with the attribute form:

```rust
#[cfg(target_os = "windows")]
fn ping_arg() -> &'static str { "-n" }
#[cfg(not(target_os = "windows"))]
fn ping_arg() -> &'static str { "-c" }
```

The attribute form is mandatory when one branch references a symbol that does not exist on the other platform. `cfg!` is fine when the code itself is portable but the value differs.

A subtle rule: `cfg!` is evaluated at compile time, so the compiler can fold it. Modern rustc absolutely does dead-code-eliminate the false branch in release builds, but it is still type-checked. If `cfg!(unix)` guards a `std::os::unix::fs::PermissionsExt` import, that import will fail to compile on Windows. Use the attribute.

## cfg_attr - applying attributes conditionally

`cfg_attr` is the most underused tool in this whole area. It applies an attribute only when the predicate is true.

```rust
// Derive Serialize only when the "serde" feature is on
#[cfg_attr(feature = "serde", derive(serde::Serialize, serde::Deserialize))]
pub struct Config {
    pub host: String,
    pub port: u16,
}
```

You can chain it for multiple attributes:

```rust
#[cfg_attr(feature = "serde", derive(serde::Serialize))]
#[cfg_attr(feature = "schemars", derive(schemars::JsonSchema))]
pub struct Config { /* ... */ }
```

This pattern keeps your `Cargo.toml` features clean and your structs un-cluttered when nobody is asking for serde.

## Conditional dependencies in Cargo.toml

The cfg predicate language extends to `Cargo.toml` for dependencies. Three forms matter:

**1. Feature-gated dependencies.** A feature can pull in a crate, and the crate becomes optional:

```toml
[dependencies]
serde = { version = "1", optional = true }

[features]
default = []
serde = ["dep:serde"]
```

The `dep:serde` syntax (Rust 1.60+) prevents Cargo from auto-creating a feature with the same name as the optional dep. Before 1.60, every `optional = true` dep silently added a feature, which surprised people.

**2. Target-specific dependencies.** This is the cfg version:

```toml
[target.'cfg(unix)'.dependencies]
nix = "0.29"

[target.'cfg(windows)'.dependencies]
windows-sys = { version = "0.59", features = ["Win32_System_Threading"] }

[target.'cfg(target_os = "linux")'.dependencies]
inotify = "0.10"
```

The full cfg predicate works here, including `all`/`any`/`not`. Cargo evaluates this at resolve time, so a Windows machine won't even download `nix`.

**3. Specific-target dependencies.** When you want to pin to an exact triple:

```toml
[target.x86_64-pc-windows-msvc.dependencies]
winapi = "0.3"
```

This is rarely the right choice. Prefer cfg predicates so the same `Cargo.toml` works for as many targets as possible.

## Custom cfg flags via build.rs

Here is where things get useful. Sometimes the built-in predicates are not enough - you want to gate code on "is libfoo version 3.5 or newer installed on this machine," or "is this build running inside Docker," or "did the user enable some local development feature." Custom cfg flags solve this.

A `build.rs` script can emit `cargo:rustc-cfg=NAME` (or `NAME="value"`) and rustc will treat that as if you had passed `--cfg`:

```rust
// build.rs
fn main() {
    // Tell Cargo we are aware of these custom cfg values, since Rust 1.80.
    println!("cargo::rustc-check-cfg=cfg(has_libsodium)");
    println!("cargo::rustc-check-cfg=cfg(big_iron)");

    if pkg_config::probe_library("libsodium").is_ok() {
        println!("cargo::rustc-cfg=has_libsodium");
    }

    if std::env::var("CARGO_CFG_TARGET_POINTER_WIDTH").as_deref() == Ok("64")
        && num_cpus::get() >= 32
    {
        println!("cargo::rustc-cfg=big_iron");
    }
}
```

Then in your code:

```rust
#[cfg(has_libsodium)]
mod sodium_backend;

#[cfg(not(has_libsodium))]
mod pure_rust_backend;
```

The `cargo::rustc-check-cfg=cfg(...)` lines are important. Since [Rust 1.80](https://blog.rust-lang.org/2024/05/06/check-cfg.html), Cargo turns on the `unexpected_cfgs` lint by default. If you write `#[cfg(has_libsodium)]` without declaring it, you get:

```
warning: unexpected `cfg` condition name: `has_libsodium`
   = help: expected names are: `clippy`, `debug_assertions`, ...
   = help: to expect this configuration use `--check-cfg=cfg(has_libsodium)`
```

This is genuinely good - it catches typos like `target_os = "windwos"` that previously silently never matched. Declare every custom cfg you emit.

For features, you do not need to do this manually. Cargo declares the feature names automatically. Only custom cfg names need explicit `check-cfg`.

## RUSTFLAGS for custom cfg values

You do not need a `build.rs` to set custom cfg. You can pass them directly through `RUSTFLAGS`:

```bash
RUSTFLAGS='--cfg my_local_dev' cargo build
```

Or in `.cargo/config.toml`:

```toml
[build]
rustflags = ["--cfg", "my_local_dev"]

[target.'cfg(target_os = "linux")']
rustflags = ["--cfg", "ebpf_supported"]
```

To declare them so `unexpected_cfgs` is happy without a `build.rs`, use the `[lints]` table:

```toml
[lints.rust]
unexpected_cfgs = { level = "warn", check-cfg = ['cfg(my_local_dev)', 'cfg(ebpf_supported)'] }
```

This is the cleanest way for "this flag is set by our CI matrix" or "this flag toggles a debug printout we never want in release."

One gotcha: changing `RUSTFLAGS` invalidates the entire build cache. If you flip cfg flags via env var on every build, you will rebuild everything. Setting them in `.cargo/config.toml` is fine because Cargo treats config-derived flags as part of the build fingerprint stably.

## cfg_if - the readability fix

You may have noticed that long chains of `#[cfg]` start to look like LISP:

```rust
#[cfg(target_os = "linux")]
fn page_size() -> usize { /* ... */ }

#[cfg(target_os = "macos")]
fn page_size() -> usize { /* ... */ }

#[cfg(target_os = "windows")]
fn page_size() -> usize { /* ... */ }

#[cfg(not(any(target_os = "linux", target_os = "macos", target_os = "windows")))]
fn page_size() -> usize { /* ... */ }
```

Every branch needs the explicit `not(any(...))` for the fallback. The [`cfg-if`](https://crates.io/crates/cfg-if) crate (currently 1.0, by Alex Crichton, ~250M downloads) gives you an `if/else if/else` form:

```rust
use cfg_if::cfg_if;

cfg_if! {
    if #[cfg(target_os = "linux")] {
        fn page_size() -> usize { 4096 }
    } else if #[cfg(target_os = "macos")] {
        fn page_size() -> usize { 16384 }
    } else if #[cfg(target_os = "windows")] {
        fn page_size() -> usize { 4096 }
    } else {
        compile_error!("unsupported platform");
    }
}
```

The macro expands to nested `#[cfg(all(..., not(...)))]` blocks. There is no runtime cost. Use it whenever you have three or more mutually exclusive arms - the resulting code is dramatically easier to maintain.

`compile_error!` is the right way to fail loudly when no arm matches. Without it, a new platform might silently miss your function and produce a confusing "function not found" error elsewhere.

## Organizing platform-specific code

For anything bigger than two functions, split by file. The standard pattern is:

```
src/
  lib.rs
  net/
    mod.rs
    unix.rs
    windows.rs
```

In `net/mod.rs`:

```rust
#[cfg(unix)]
mod unix;
#[cfg(unix)]
pub use unix::*;

#[cfg(windows)]
mod windows;
#[cfg(windows)]
pub use windows::*;
```

This keeps each platform's implementation in its own file with normal imports, no cfg per item. The downside is that `cargo check` only checks the platform you are on - if you break the Windows file, you won't know until CI runs on Windows. The fix is `--target` checks in CI:

```yaml
# .github/workflows/ci.yml (sketch)
strategy:
  matrix:
    os: [ubuntu-latest, macos-latest, windows-latest]
steps:
  - uses: dtolnay/rust-toolchain@stable
  - run: cargo check --all-targets
  - run: cargo test
```

Or, if you want a one-shot local sanity check, install all the targets you support and run `cargo check --target X` for each. `cargo check` does not link, so you can check Windows code from Linux without installing the MSVC toolchain.

## Testing platform-specific code

`#[cfg(test)]` is the obvious one - it gates a module so it only exists during `cargo test`. But you often want tests that only run on certain platforms:

```rust
#[cfg(test)]
mod tests {
    #[test]
    #[cfg(unix)]
    fn permissions_round_trip() {
        use std::os::unix::fs::PermissionsExt;
        let perms = std::fs::Permissions::from_mode(0o644);
        assert_eq!(perms.mode() & 0o777, 0o644);
    }

    #[test]
    #[cfg(target_os = "linux")]
    fn proc_self_status_exists() {
        assert!(std::path::Path::new("/proc/self/status").exists());
    }
}
```

The `#[cfg(test)]` on the module gates compilation; the `#[cfg(...)]` on the individual `#[test]` gates the specific test. You can also use `#[ignore]` for tests that compile everywhere but should only run sometimes (useful for slow integration tests).

For "test only when a feature is on":

```rust
#[test]
#[cfg(feature = "network-tests")]
fn talks_to_real_dns() { /* ... */ }
```

And for testing the same logic on multiple targets without separate test files, use `cfg!()` inside the test - it lets you assert different expected values per platform:

```rust
#[test]
fn line_separator_is_platform_native() {
    let sep = if cfg!(windows) { "\r\n" } else { "\n" };
    assert_eq!(some_function_under_test(), sep);
}
```

## A few patterns I keep coming back to

**Compile-time platform error.** When you absolutely do not support some platform, fail at build time instead of producing a binary that crashes:

```rust
#[cfg(not(any(unix, windows)))]
compile_error!("this crate only supports Unix and Windows targets");
```

**Feature unification gotcha.** Cargo unifies features across the dependency graph. If crate A depends on `tokio` with `["rt"]` and crate B depends on `tokio` with `["rt-multi-thread", "macros"]`, every consumer gets the union. You cannot turn off a feature by depending on the crate without it. Plan features additively - "adding a feature only adds capabilities, never removes them."

**Use namespaced features for optional deps.** The `dep:foo` syntax matters because once a feature like `feature = "json"` is present, anyone can write `--features json` even if they did not mean the optional dep. With `dep:`, you control what activates the dep.

**Group cfg into type aliases.** When the same long predicate shows up everywhere, hide it:

```rust
#[cfg(any(target_os = "linux", target_os = "android"))]
mod platform { pub type RawHandle = i32; }

#[cfg(windows)]
mod platform { pub type RawHandle = *mut core::ffi::c_void; }

pub use platform::RawHandle;
```

Now the rest of the crate uses `RawHandle` and the cfg lives in one place.

## What to read next

The [Cargo reference on features](https://doc.rust-lang.org/cargo/reference/features.html) is the definitive source for feature unification rules. The [Rust reference on conditional compilation](https://doc.rust-lang.org/reference/conditional-compilation.html) lists every stable predicate with examples. For build scripts, [the Cargo book chapter on build.rs](https://doc.rust-lang.org/cargo/reference/build-scripts.html) covers every output instruction including the new `cargo::` syntax. And if you maintain a crate that supports many platforms, read the [check-cfg announcement post](https://blog.rust-lang.org/2024/05/06/check-cfg.html) - declaring your custom cfg names properly will save you and your contributors a lot of head scratching.

Conditional compilation is one of those parts of Rust where the basics are easy, the intermediate stuff is poorly taught, and the advanced stuff (custom cfg, build.rs, feature unification) is where every nontrivial cross-platform crate lives. Once you have these in your toolbox, "this code only runs on Linux when feature X is on and the user has libfoo installed" stops being a scary requirement and starts being a half-hour task.
