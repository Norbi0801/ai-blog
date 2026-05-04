+++
title = "Writing documentation that developers actually read"
date = 2025-03-09
description = "Practical Rust documentation techniques: cargo doc, doc comments, doc tests, intra-doc links, README structure, mdBook, and cfg(doc) for doc-only content."

[taxonomies]
tags = ["rust", "documentation", "tools", "developer-experience"]
+++

Most documentation is written and never read. You know the type: auto-generated API references with no examples, READMEs that say "a library for doing X" without showing how, doc comments that restate the function signature in English. Nobody reads those. Not because developers hate docs - because those docs don't answer the questions developers actually have.

Good documentation answers three questions: *what does this do*, *when would I use it*, and *show me*. Rust's toolchain makes this easier than most languages because documentation is code. It compiles. It gets tested. It links to other items by name. And `cargo doc` turns it all into a searchable, cross-referenced website with zero configuration.

This post covers how to write Rust documentation that people actually use - from doc comments and doc tests to README structure, mdBook, and the less-known features like `cfg(doc)` and `#[doc(alias)]`.

<!-- more -->

## The two comment styles: /// and //!

Rust has two doc comment syntaxes. The difference is *what they document*.

`///` (outer doc comments) document the item that follows them:

```rust
/// Compresses the input using zstd at the given compression level.
///
/// Level ranges from 1 (fastest, least compression) to 22 (slowest, best
/// compression). The default level is 3, which is a good balance for
/// most payloads under 1MB.
///
/// Returns the compressed bytes, or an error if the input exceeds
/// the maximum supported size (2GB).
pub fn compress(input: &[u8], level: i32) -> Result<Vec<u8>, CompressError> {
    // ...
}
```

`//!` (inner doc comments) document the item they're *inside* of - typically a module or crate root:

```rust
//! # my_compress
//!
//! A thin wrapper around zstd with sane defaults and streaming support.
//!
//! ## Quick start
//!
//! ```rust
//! use my_compress::compress;
//!
//! let data = b"hello world hello world hello world";
//! let compressed = compress(data, 3).unwrap();
//! assert!(compressed.len() < data.len());
//! ```

pub mod streaming;
pub mod compress;
```

The `//!` block at the top of `lib.rs` becomes your crate's front page on [docs.rs](https://docs.rs). This is prime real estate. If someone lands on your crate's documentation, this is what they see first. Make it count - show a working example, not a paragraph of abstract description.

You can also use `/*! */` and `/** */` for block-style doc comments, but the line-style variants are overwhelmingly more common in the ecosystem.

## cargo doc - what actually happens

Running `cargo doc --open` does three things:

1. Parses every `///` and `//!` comment in your crate (and dependencies)
2. Generates HTML into `target/doc/`
3. Opens the result in your browser

Some useful flags:

```bash
# Build docs for your crate only (skip dependencies)
cargo doc --no-deps

# Include private items - useful when documenting internals for your team
cargo doc --document-private-items

# Pass flags to rustdoc directly
cargo rustdoc -- --default-theme ayu
```

Under the hood, `cargo doc` invokes `rustdoc`, which is a separate binary in the Rust toolchain. Rustdoc parses doc comments as Markdown (CommonMark, specifically), resolves intra-doc links, runs doc tests, and produces the HTML. The output format is stable and well-known - it's the same format you see on docs.rs.

One thing worth knowing: `cargo doc` by default only documents public items. If you want documentation for private functions, structs, and modules (for an internal team wiki, for example), `--document-private-items` includes everything. This is enabled automatically for binary crates since there's no public API to document otherwise.

## Stop restating signatures - show when and why

The single biggest documentation antipattern is restating what the code already says:

```rust
/// Gets the name.
pub fn name(&self) -> &str { ... }

/// Sets the name.
pub fn set_name(&mut self, name: String) { ... }

/// Returns true if empty.
pub fn is_empty(&self) -> bool { ... }
```

This adds zero information. The signature already tells me it returns `&str`, takes a `String`, returns `bool`. What I actually want to know:

```rust
/// The display name shown in the UI header and notification emails.
///
/// Defaults to the username if not explicitly set. Names are trimmed
/// and truncated to 100 characters on save.
pub fn name(&self) -> &str { ... }

/// Overwrites the display name.
///
/// Pass an empty string to reset to the default (username).
/// Names longer than 100 characters are silently truncated.
///
/// # Panics
///
/// Panics if the account is in a read-only state. Check
/// [`Account::is_locked`] before calling this in contexts where
/// the lock state is unknown.
pub fn set_name(&mut self, name: String) { ... }
```

Now I know the business rules, the edge cases, and the failure modes. I know what "empty" means (reset to default). I know there's a 100-character limit. I know to check `is_locked` first. None of that is in the type signature.

The [Rust API Guidelines](https://rust-lang.github.io/api-guidelines/documentation.html) put it well: document what the function *does*, not what it *is*. Include information that's not obvious from the signature.

## Doc tests - your examples are your tests

This is Rust's killer documentation feature. Every code block in a doc comment is compiled and executed as a test when you run `cargo test`. Not a separate test file. Not a markdown linter. The actual compiler checks your examples.

```rust
/// Parses a duration string like "5s", "100ms", or "2m30s".
///
/// ```
/// use my_crate::parse_duration;
/// use std::time::Duration;
///
/// assert_eq!(parse_duration("5s").unwrap(), Duration::from_secs(5));
/// assert_eq!(parse_duration("100ms").unwrap(), Duration::from_millis(100));
/// assert!(parse_duration("garbage").is_err());
/// ```
pub fn parse_duration(s: &str) -> Result<Duration, ParseError> {
    // ...
}
```

Run `cargo test --doc` and this example is compiled and executed. If `parse_duration` changes its behavior or signature, the doc test fails. No more stale examples that don't compile.

### Controlling what runs and what shows

Not every line in a doc test example is interesting to the reader. You can hide boilerplate with `#`:

```rust
/// Connects to the database and runs a health check.
///
/// ```no_run
/// # use my_crate::Database;
/// # fn main() -> Result<(), Box<dyn std::error::Error>> {
/// let db = Database::connect("postgres://localhost/myapp").await?;
/// let healthy = db.health_check().await?;
/// assert!(healthy);
/// # Ok(())
/// # }
/// ```
pub async fn health_check(&self) -> Result<bool, DbError> {
    // ...
}
```

Lines starting with `# ` are compiled but hidden in the rendered documentation. The reader sees the clean three-line example. The compiler sees the full program with imports, `main` function, and error handling.

The annotations on the code fence control compilation behavior:

| Annotation | Compiles? | Runs? | Use case |
|---|---|---|---|
| ` ``` ` | Yes | Yes | Normal examples |
| ` ```no_run ` | Yes | No | Examples that need network/filesystem |
| ` ```compile_fail ` | Yes (must fail) | No | Showing what *doesn't* work |
| ` ```ignore ` | No | No | Pseudocode, non-Rust snippets |
| ` ```should_panic ` | Yes | Yes (must panic) | Demonstrating panic conditions |

The `compile_fail` annotation is underrated. It lets you document invariants enforced by the type system:

```rust
/// Handles are not `Send` - they cannot be transferred across threads.
///
/// ```compile_fail
/// use my_crate::Handle;
///
/// let handle = Handle::new();
/// std::thread::spawn(move || {
///     handle.do_thing(); // ERROR: Handle is not Send
/// });
/// ```
pub struct Handle { /* ... */ }
```

This test *passes* when the code *fails to compile*. If someone accidentally makes `Handle: Send`, the doc test breaks. You've encoded a design invariant into a test that lives right next to the documentation explaining it.

### Doc test performance

Doc tests used to be slow because each one was compiled as a separate binary. Since Rust 1.73, rustdoc merges doc tests into a single binary by default (the `--merge=shared` strategy), which dramatically improved compilation time. If you have 200 doc tests, that's one binary now instead of 200. You can still opt out per-test with `standalone` if a test needs its own binary (e.g., it sets a global allocator).

## Intra-doc links - let the compiler resolve your references

Manually writing links to other items is fragile. URLs change, items move, modules get renamed. Intra-doc links solve this by letting you reference items by their Rust path:

```rust
/// Creates a new [`Config`] with default values.
///
/// For customization, use [`ConfigBuilder`] instead.
/// See the [module-level documentation](crate::config) for examples.
///
/// If you need to validate the config before use, see
/// [`Config::validate`].
pub fn default_config() -> Config {
    // ...
}
```

Rustdoc resolves `[`Config`]` to the actual `Config` type in scope, generates the correct URL, and - critically - **fails the build if the link target doesn't exist**. Rename `Config` to `Settings`? Every doc comment linking to `Config` becomes a broken link, and `cargo doc` tells you exactly where.

The syntax supports disambiguation when names collide:

```rust
/// See [`fmt`](mod@fmt) for the module, or [`fmt`](macro@fmt) for the macro.
///
/// The [`into`](fn@Into::into) method on [`Into`](trait@Into).
```

Available disambiguators include `struct@`, `enum@`, `trait@`, `fn@`, `mod@`, `macro@`, `type@`, `const@`, and `field@`. In practice, you rarely need them - rustdoc is smart enough to resolve most names from context.

You can also link across crates. If your crate depends on `serde`, you can write `[`Serialize`](serde::Serialize)` and rustdoc generates a link to serde's docs.

## Standard documentation sections

The Rust ecosystem has conventions for doc comment sections that tools and developers recognize. Use them consistently:

```rust
/// Brief one-line summary.
///
/// More detailed explanation if needed. This can be multiple paragraphs.
///
/// # Examples
///
/// ```
/// // Show the common case
/// ```
///
/// # Errors
///
/// Returns [`MyError::NotFound`] if the key doesn't exist.
/// Returns [`MyError::Expired`] if the entry's TTL has passed.
///
/// # Panics
///
/// Panics if called from within an async context without a runtime.
///
/// # Safety
///
/// (for unsafe functions) The caller must ensure the pointer is valid
/// and properly aligned.
pub fn get(&self, key: &str) -> Result<Value, MyError> {
    // ...
}
```

The `# Panics` and `# Errors` sections aren't just convention - the [Rust API Guidelines](https://rust-lang.github.io/api-guidelines/documentation.html) (C-FAILURE) explicitly require them. Clippy's `missing_panics_doc` and `missing_errors_doc` lints enforce this. If your function can panic or returns `Result`, document when and why.

The `# Safety` section is mandatory for `unsafe` functions. It's the contract between the function author and the caller. Skip it, and nobody can safely call your function.

## Enforcing documentation with lints

Rust ships with built-in lints that catch missing documentation:

```rust
// In lib.rs - warn on any public item without docs
#![warn(missing_docs)]

// Stricter: deny missing docs (fails compilation)
#![deny(missing_docs)]
```

This catches every public function, struct, enum, trait, module, and constant that's missing a `///` comment. It's aggressive, but it works. Most well-maintained crates in the ecosystem use at least `#![warn(missing_docs)]`.

There are also Clippy lints for doc quality:

```toml
# In Cargo.toml or clippy.toml
[lints.clippy]
missing_panics_doc = "warn"
missing_errors_doc = "warn"
missing_safety_doc = "deny"
```

`missing_safety_doc` is the most critical one. An `unsafe fn` without a `# Safety` section is a bug, not a style issue.

## #[doc] attributes - the power features

Beyond `///` sugar, the `#[doc]` attribute has several forms that solve specific problems.

### #[doc(alias)]

Users search for your types by different names. Maybe they're coming from Python and call it a "dictionary" instead of "HashMap". Or they're wrapping a C library and users search by the C function name:

```rust
#[doc(alias = "CreateWindowEx")]
#[doc(alias = "create window")]
pub fn new_window(title: &str, width: u32, height: u32) -> Window {
    // ...
}
```

Now searching for "CreateWindowEx" or "create window" in the rustdoc search bar finds `new_window`. This is invaluable for FFI wrappers where users know the C API but not your Rust naming.

### #[doc(hidden)]

Hides an item from documentation entirely. The item is still public (accessible by code), but invisible in docs:

```rust
/// Public API.
pub fn connect(url: &str) -> Connection { /* ... */ }

/// Implementation detail - public for macro use, not part of the stable API.
#[doc(hidden)]
pub fn __internal_connect_raw(url: &str, flags: u32) -> RawConnection { /* ... */ }
```

This is commonly used for items that must be `pub` for technical reasons (macro expansion, trait coherence) but aren't part of the public API contract. If you've read my post on [semantic versioning](/blog/semantic-versioning-what-breaking-changes-actually-means-in-rust), you know that removing a `pub` item is a breaking change - but removing a `#[doc(hidden)]` item is generally considered acceptable in a minor version, since users shouldn't have been depending on it.

### #[doc = include_str!("...")]

Pull documentation from an external file:

```rust
#[doc = include_str!("../README.md")]
pub struct Crate;
```

This is how many crates keep their README and crate-level docs in sync. Write the README once, include it as the crate's front page. The `include_str!` macro runs at compile time, so the file must exist or compilation fails.

## cfg(doc) - doc-only content

Sometimes you want to show items in documentation that don't exist in normal compilation. Platform-specific APIs are the classic case:

```rust
/// Unix-specific extensions for file permissions.
#[cfg(any(unix, doc))]
pub mod unix_ext {
    /// Sets the file mode bits (chmod).
    pub fn set_mode(path: &str, mode: u32) -> std::io::Result<()> {
        // ...
        # Ok(())
    }
}
```

The `cfg(any(unix, doc))` means this module is compiled on Unix *and* when building documentation on any platform. Without the `doc` part, `cargo doc` on Windows would skip the entire module - your cross-platform docs would be incomplete.

Rustdoc sets the `doc` cfg flag whenever it runs. This lets you include platform-specific items, feature-gated items, or even purely illustrative types that only exist for documentation purposes:

```rust
/// Example connection type used in documentation.
/// Not available at runtime.
#[cfg(doc)]
pub struct ExampleConnection;
```

The standard library uses this extensively. Look at [`std::os::unix`](https://doc.rust-lang.org/std/os/unix/index.html) - it's visible in the docs even when you're browsing from Windows, because it's gated on `cfg(any(unix, doc))`.

## README structure that works

Your crate's README is often the first thing someone sees - on GitHub, on crates.io, or via a search engine. A README that answers the right questions in the right order keeps people reading. Here's the structure that works:

**1. What is this?** One sentence. Not a paragraph. "A rate limiter for Rust HTTP servers with sliding window and token bucket algorithms."

**2. Why would I use this?** Two to three sentences on the problem it solves and what makes it different from alternatives. "Most rate limiters only support fixed windows, which cause burst traffic at window boundaries. This crate implements sliding window counters with O(1) memory per key."

**3. Quick start.** A complete, copy-pasteable example that goes from `cargo add` to working code:

```markdown
## Quick start

```bash
cargo add my-rate-limiter
```

```rust
use my_rate_limiter::SlidingWindow;

let limiter = SlidingWindow::new(100, Duration::from_secs(60));

if limiter.check("user-123").is_ok() {
    // Request allowed
} else {
    // Rate limited
}
```
```

**4. More examples.** The quick start shows the happy path. Follow-up examples show configuration, error handling, integration with common frameworks (axum, actix-web), and edge cases.

**5. API overview.** Not a full API reference (that's what cargo doc is for). A high-level map: "The main types are `SlidingWindow`, `TokenBucket`, and `RateLimiterLayer`. The layer integrates with tower middleware."

**6. MSRV, license, contributing.** Boilerplate that people expect to find.

The key principle: the README is a funnel. Each section qualifies the reader further. Most people make their decision in the first 30 seconds - in the "what" and "why" sections. Only people who are seriously evaluating the crate read to the API overview. Structure accordingly.

## mdBook for larger documentation

When your project outgrows a README and some doc comments, [mdBook](https://rust-lang.github.io/mdBook/) is the standard tool. The Rust Programming Language book, the Cargo book, the Rustdoc book - they're all mdBook. It takes a directory of Markdown files and produces a static website with search, theming, and navigation.

Setup is minimal:

```bash
cargo install mdbook
mdbook init my-project-docs
cd my-project-docs
mdbook serve --open
```

This creates a `book.toml` config and a `src/` directory with `SUMMARY.md` (the table of contents) and `chapter_1.md`. The `SUMMARY.md` drives the sidebar navigation:

```markdown
# Summary

- [Introduction](./introduction.md)
- [Getting started](./getting-started.md)
  - [Installation](./installation.md)
  - [First project](./first-project.md)
- [Architecture](./architecture.md)
- [API reference](./api/README.md)
  - [Config](./api/config.md)
  - [Client](./api/client.md)
```

Indentation creates nested sections. mdBook watches for file changes and live-reloads.

The key advantage over a wiki or Google Doc: mdBook content lives in your repo, gets versioned with your code, and goes through pull review. Documentation drift - where the docs say one thing and the code does another - is the number one reason developers stop trusting docs. Keeping docs next to code and reviewing them together is the best defense.

### Code blocks in mdBook

mdBook can test Rust code blocks the same way rustdoc does. Add this to `book.toml`:

```toml
[rust]
edition = "2024"
```

Then run `mdbook test`. Every untagged Rust code block gets compiled and executed. Same `#` hiding syntax as doc tests. Same `ignore`, `no_run`, and `compile_fail` annotations. Your book's examples stay honest.

## Linking between items - building a web, not a list

Documentation becomes genuinely useful when items reference each other. Instead of explaining the same concept in five places, explain it once and link to it everywhere else.

In doc comments, use intra-doc links aggressively:

```rust
/// A connection pool that manages [`Connection`] instances.
///
/// Use [`Pool::builder`] to configure pool size and timeouts.
/// Each connection is validated using [`Connection::health_check`]
/// before being returned to a caller.
///
/// For the underlying connection type, see [`Connection`].
/// For error types, see [`PoolError`].
pub struct Pool { /* ... */ }
```

Every `[`..`]` is a clickable link in the generated HTML. The reader can navigate from `Pool` to `Connection` to `PoolError` without using the search bar. This turns flat documentation into a navigable graph.

In module-level docs (`//!`), provide a guided tour:

```rust
//! # Connection management
//!
//! This module provides [`Pool`] for connection pooling and
//! [`Connection`] for individual database connections.
//!
//! ## Typical usage
//!
//! 1. Create a pool with [`Pool::builder`]
//! 2. Acquire connections with [`Pool::get`]
//! 3. Connections return to the pool on drop
//!
//! ## Error handling
//!
//! All fallible operations return [`PoolError`], which implements
//! [`std::error::Error`] and can be converted from [`std::io::Error`].
```

This is the "table of contents" for the module. Someone landing on the module page immediately sees the key types, the workflow, and the error story. No scrolling through alphabetically sorted items hoping to find the entry point.

## The documentation quality checklist

Before publishing a crate, run through this:

```bash
# 1. Build docs and check for broken intra-doc links
cargo doc --no-deps 2>&1 | grep "warning"

# 2. Run doc tests
cargo test --doc

# 3. Check for missing docs on public items
# (add #![warn(missing_docs)] to lib.rs first)
cargo clippy

# 4. Preview what docs.rs will show
cargo doc --no-deps --open
```

If you want to enforce all of this in CI, the [Rust API Guidelines checklist](https://rust-lang.github.io/api-guidelines/checklist.html) is the canonical reference. The documentation section covers:

- Crate level docs with examples (C-CRATE-DOC)
- All public items documented (C-DOC)
- Examples on all public items (C-EXAMPLE)
- Panic conditions documented (C-FAILURE)
- Function semantics documented beyond the type signature (C-DOC-SEMANTICS)
- Links to related items (C-LINK)

## Documentation as a design signal

There's a meta-point worth making. If you struggle to document a function, the function might be badly designed. If the "when to use this" section requires three paragraphs of caveats, the API probably needs simplification. If you can't write a clean example, the ergonomics need work.

I've lost count of how many times writing documentation exposed a design problem. "Wait, the user has to call `init()` before `connect()` but after `configure()`? Why?" The act of explaining the API to someone who's never seen it forces you to confront the accidental complexity you've been ignoring.

This is why documentation-first development works. Write the doc comment before the implementation. Write the example before the function body. If the example is awkward, redesign the API. The example *is* the UX test.

When I evaluate crates - and I wrote about what to look for in my post on [build vs buy decisions](/blog/build-vs-buy---when-to-use-a-library-and-when-to-write-your-/) - documentation quality is a direct signal of maintenance quality. A crate with thorough docs, tested examples, and linked items tells me the author thinks about users. A crate with `/// TODO: document this` tells me the author ships and moves on.

Write docs that answer the questions you'd have as a newcomer. Test the examples. Link the items. The tooling is there - `cargo doc`, doc tests, intra-doc links, mdBook. Rust makes documentation a first-class artifact, not an afterthought. Use it.
