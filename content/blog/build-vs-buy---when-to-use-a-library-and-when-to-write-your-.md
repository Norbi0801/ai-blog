+++
title = "Build vs buy - when to use a library and when to write your own"
date = 2025-02-04
description = "A practical framework for deciding when to pull in a crate and when to write the code yourself, plus tools to keep your dependency graph honest."

[taxonomies]
tags = ["rust", "architecture", "security", "tools"]
+++

Every crate in your `Cargo.toml` is a bet. You're betting that the maintainer will keep it updated, that its transitive dependencies won't introduce vulnerabilities, that its API won't change in ways that break your build, and that it'll still exist five years from now. Sometimes that bet pays off handsomely. Sometimes you end up vendoring a fork at 2 AM because a single-maintainer crate hasn't been touched in 18 months and `cargo audit` is screaming.

The Rust ecosystem has over 247,000 crates on [crates.io](https://crates.io). The registry serves nearly a billion downloads per day. That's a massive amount of code you could pull in with a single line in `Cargo.toml`. The question isn't whether libraries exist for your problem - they almost certainly do. The question is whether you *should* use them.

<!-- more -->

## Every dependency is a maintenance burden

Before getting into when to use a library and when not to, it's worth understanding what a dependency actually costs you. It's not just disk space.

**Build time.** Each crate gets compiled. Its proc macros run at compile time. Its transitive dependencies get compiled too. A single `cargo add` can pull in dozens of crates you never asked for. Run `cargo tree` on any non-trivial project and count the lines:

```bash
$ cargo tree | wc -l
247

$ cargo tree --duplicates
chrono v0.4.38
├── chrono v0.4.38
│   └── time v0.1.45
│       └── libc v0.2.155
└── chrono v0.4.31
    └── time v0.1.44
        └── libc v0.2.150
```

Those duplicates? They each get compiled separately. Two versions of `chrono` means two versions of everything `chrono` depends on. Your CI pipeline doesn't care about the reason - it just takes longer.

**Binary size.** The linker does its best with dead code elimination, but it's not magic. More code pulled in means more code in your final binary, especially if you're not using LTO (link-time optimization). For embedded targets or Lambda functions where binary size directly maps to cold start latency, this matters.

**Attack surface.** Every crate can include a `build.rs` that runs arbitrary code during compilation. Every proc macro runs inside the compiler. A malicious crate doesn't need to be in your runtime code to compromise your system - it just needs to be in your build graph. In September 2025, two crates named `faster_log` and `async_println` were [caught stealing Solana and Ethereum private keys](https://blog.rust-lang.org/2025/09/24/crates.io-malicious-crates-fasterlog-and-asyncprintln/) via typosquatting. They accumulated over 8,400 downloads before being removed. The `rustdecimal` typosquat of `rust_decimal` specifically targeted GitLab CI pipelines. These aren't hypothetical risks.

**Upgrade pressure.** Crates have their own release cycles. When `serde` bumps a minor version, everything that depends on it potentially needs updating. When a crate you depend on changes its MSRV, you might be forced to update your toolchain. When a dependency of a dependency has a security advisory, you're now debugging transitive version resolution at 5 PM on a Friday.

## When to use a library

Despite all of that, there are categories of code where writing your own is almost always the wrong call.

### Cryptography

Never roll your own crypto. This isn't a Rust-specific rule - it's a universal engineering principle. The gap between "correct implementation" and "secure implementation" in cryptography is enormous. A constant-time comparison function that compiles to variable-time assembly on one target defeats the entire purpose. Use [ring](https://crates.io/crates/ring), [rustls](https://crates.io/crates/rustls), or the [RustCrypto](https://github.com/RustCrypto) crates. They've been audited. Your weekend implementation hasn't.

```rust
// Don't do this
fn verify_hmac(expected: &[u8], actual: &[u8]) -> bool {
    expected == actual // timing side-channel, game over
}

// Do this
use ring::hmac;
fn verify_hmac(key: &hmac::Key, msg: &[u8], tag: &[u8]) -> bool {
    hmac::verify(key, msg, tag).is_ok() // constant-time comparison
}
```

### Parsers for established formats

JSON, TOML, YAML, XML, CSV, HTTP - these formats have specs that are hundreds of pages long with edge cases that take years to discover. `serde_json` handles [RFC 8259](https://datatracker.ietf.org/doc/html/rfc8259) correctly, including surrogate pairs in `\uXXXX` escapes, deeply nested structures, and the whole zoo of number representations. If you're curious about what building a parser from scratch looks like and what you learn from it, I covered that in [Writing a Markdown parser in Rust](/blog/writing-a-markdown-parser-in-rust/) - but even that post concluded by comparing the hand-rolled version to what [pulldown-cmark](https://crates.io/crates/pulldown-cmark) handles that a weekend implementation can't.

### Async runtimes and networking

Writing your own async runtime means implementing a task scheduler, an I/O reactor (epoll/kqueue/IOCP), timer wheels, and cooperative scheduling. Tokio has been battle-tested across millions of production deployments. The runtime internals are subtle - work-stealing schedulers have correctness properties that are genuinely hard to get right. Use tokio. Use hyper. Use tonic. Focus your energy on the code that makes your product different.

### Frameworks with ecosystem gravity

Some crates have become the gravitational center of their domain. `serde` for serialization. `tracing` for structured logging. `clap` for CLI argument parsing. Even if you could write your own argument parser in 200 lines, you'd lose `clap`'s derive macros, shell completion generation, and the fact that every Rust developer already knows the API. The familiarity dividend is real, especially on teams.

## When to write your own

### Glue code and adapters

If the "library" is 30 lines of code that maps one type to another, just write it. The cost of adding a dependency - tracking its releases, auditing its dependencies, managing version conflicts - can easily exceed the cost of writing and maintaining a small module.

```rust
// You don't need a crate for this
pub fn slugify(input: &str) -> String {
    input
        .chars()
        .map(|c| if c.is_alphanumeric() { c.to_ascii_lowercase() } else { '-' })
        .collect::<String>()
        .split('-')
        .filter(|s| !s.is_empty())
        .collect::<Vec<_>>()
        .join("-")
}
```

Yes, there are crates that do slugification. They handle Unicode normalization, transliteration, locale-specific rules. But if your slugs are for URL paths on an English-language blog, the function above does the job. The 6 lines are easier to audit than 6 transitive dependencies.

### Simple utilities

String padding, retry logic with exponential backoff, basic configuration parsing from environment variables - these are things where a purpose-built function that exactly matches your requirements is better than a generic library that handles 50 cases you don't need. The JavaScript ecosystem learned this the hard way with `left-pad`, an 11-line function that became a transitive dependency of thousands of packages.

```rust
// You don't need a crate for retry with backoff
pub async fn retry<F, Fut, T, E>(mut f: F, max_retries: u32) -> Result<T, E>
where
    F: FnMut() -> Fut,
    Fut: std::future::Future<Output = Result<T, E>>,
{
    let mut attempt = 0;
    loop {
        match f().await {
            Ok(val) => return Ok(val),
            Err(e) if attempt >= max_retries => return Err(e),
            Err(_) => {
                let delay = std::time::Duration::from_millis(100 * 2u64.pow(attempt));
                tokio::time::sleep(delay).await;
                attempt += 1;
            }
        }
    }
}
```

Twenty lines, no dependencies beyond tokio (which you're already using), and you can tune the behavior exactly to your needs. Add jitter? Add it. Log retries? Add a tracing span. A generic retry crate would give you configuration options you don't need and abstractions you have to learn.

### Core business logic

This is the big one. The code that makes your product unique should never come from a crate. Your pricing engine, your matching algorithm, your domain-specific validation rules - these are the things you're paid to get right. Outsourcing them to a library means outsourcing your competitive advantage to someone else's API design decisions and release schedule.

If you've read the post on [the adapter pattern](/blog/the-adapter-pattern-in-rust-wrapping-external-apis/), this connects directly. Even when you do use external crates for infrastructure concerns, wrapping them behind a trait boundary keeps your business logic independent of any specific library's types. That's not just good architecture - it's insurance against the day when `some_crate` stops being maintained.

## Evaluating a crate before adding it

When you've decided a dependency is worth it, don't just `cargo add` the first result. Run through this checklist:

**Downloads and trajectory.** Not just total downloads - look at recent downloads. A crate with 10 million total downloads but 200 downloads last week is a crate people are migrating away from. Check the graph on [crates.io](https://crates.io) or [lib.rs](https://lib.rs).

**Last commit, not last release.** Some well-maintained crates simply don't need frequent releases. But if the GitHub repo hasn't had a commit in 12+ months and has open issues piling up, that's a signal. Check the issues tab. Are bugs being triaged? Are PRs being reviewed?

**Bus factor.** How many people have commit access? A crate maintained by a single developer has a bus factor of one. If that person loses interest, changes careers, or gets hit by the proverbial bus, the crate is orphaned. Look at the contributor graph. If one person authored 98% of commits, you're depending on their continued enthusiasm.

**License compatibility.** `MIT` and `Apache-2.0` are safe for almost any project. `GPL-3.0` has copyleft implications. `AGPL-3.0` affects network services. Some crates use `BSL` or custom licenses. Check not just the crate's license - check its transitive dependencies. One `GPL` crate three levels deep in your dependency tree can have legal implications for your entire binary.

**Transitive dependency count.** Run `cargo tree -p the_crate` before adding it. If a logging library pulls in 47 transitive dependencies, maybe look for a simpler one. Each transitive dependency is a crate you're implicitly trusting with your build pipeline and runtime.

```bash
# How many unique crates does this dependency pull in?
$ cargo tree -p serde_json | sort -u | wc -l
6

# Compare with a heavier alternative
$ cargo tree -p some-json-framework | sort -u | wc -l
89
```

**`unsafe` usage.** Run `cargo geiger` on the crate. Code with `unsafe` blocks needs more scrutiny - not because `unsafe` is inherently bad, but because it's where Rust's safety guarantees stop. A crate with extensive `unsafe` should have a clear reason for it (FFI, performance-critical hot paths) and ideally some form of safety documentation.

## Tooling: cargo-audit and cargo-deny

Two tools should be in every Rust project's CI pipeline.

### cargo-audit

[cargo-audit](https://crates.io/crates/cargo-audit) checks your `Cargo.lock` against the [RustSec Advisory Database](https://rustsec.org/). It catches known vulnerabilities and flags unmaintained crates:

```bash
$ cargo install cargo-audit
$ cargo audit

Crate:     chrono
Version:   0.4.19
Title:     Potential segfault in localtime_r invocations
Date:      2020-11-10
ID:        RUSTSEC-2020-0159
URL:       https://rustsec.org/advisories/RUSTSEC-2020-0159
Solution:  Upgrade to >=0.4.20

error: 1 vulnerability found!
```

As of early 2026, crates.io itself now surfaces [security advisories directly on crate pages](https://alpha-omega.dev/blog/surfacing-security-advisories-on-crates-io-bringing-vulnerability-data-to-the-point-of-discovery/) - around 700 crates have advisories displayed on their Security tab. But `cargo audit` in CI catches problems before they reach production.

### cargo-deny

[cargo-deny](https://github.com/EmbarkStudios/cargo-deny) is broader. It checks four categories: advisories (like `cargo-audit`), licenses, bans (blocking specific crates), and sources (ensuring crates come from trusted registries). A minimal `deny.toml`:

```toml
[advisories]
vulnerability = "deny"
unmaintained = "warn"

[licenses]
allow = [
    "MIT",
    "Apache-2.0",
    "BSD-2-Clause",
    "BSD-3-Clause",
    "ISC",
    "Unicode-3.0",
]

[bans]
multiple-versions = "warn"
deny = [
    # Block specific problematic crates
    { name = "openssl", wrappers = ["openssl-sys"] },
]

[sources]
unknown-registry = "deny"
unknown-git = "deny"
allow-registry = ["https://github.com/rust-lang/crates.io-index"]
```

The `[sources]` section is particularly important. It prevents anyone from accidentally adding a dependency from a random Git repository or private registry. If every dependency must come from crates.io, at least you have the crates.io team's review process and the community's eyes as a minimal layer of defense.

Put both tools in your CI:

```yaml
# In your GitHub Actions workflow
- name: Security audit
  run: |
    cargo install cargo-audit cargo-deny
    cargo audit
    cargo deny check
```

## Could Rust have a left-pad moment?

In 2016, a developer unpublished `left-pad` from npm, breaking thousands of JavaScript projects worldwide. npm responded by changing their deletion policy - you can no longer unpublish packages that other packages depend on.

Cargo's design anticipated this. On crates.io, you can *yank* a version but you cannot *delete* it. A [yanked crate](https://doc.rust-lang.org/cargo/commands/cargo-yank.html) version still exists on the registry. If it's already in your `Cargo.lock`, `cargo build` will still download and use it. Yanking only prevents *new* projects from adding the yanked version as a dependency. The actual removal of code from crates.io requires intervention from the Rust infrastructure team - no individual maintainer can unilaterally delete their crate's history.

This, combined with `Cargo.lock`, means your builds are reproducible as long as the registry is reachable. The lockfile pins exact versions of every dependency, direct and transitive. If you commit `Cargo.lock` (which you should for binaries and applications), your build today will use the same versions as your build next year, regardless of what happens upstream.

For extra resilience, consider a registry mirror. Cargo's [sparse protocol](https://rust-lang.github.io/rfcs/2789-sparse-index.html) (default since Rust 1.70) fetches only the index entries you need rather than cloning the entire registry. If you're running a team or organization, tools like [Cloudsmith](https://cloudsmith.com), [Artifactory](https://jfrog.com/artifactory/), or [Sonatype Nexus](https://www.sonatype.com/) can proxy and cache crates.io, giving you a local copy that survives registry outages. Configure it in `.cargo/config.toml`:

```toml
[source.crates-io]
replace-with = "mirror"

[source.mirror]
registry = "sparse+https://your-mirror.internal/index/"
```

## A decision framework

When you're staring at a problem and wondering "crate or custom?", ask these questions in order:

1. **Is this a solved, hard problem?** (crypto, compression, parsing standard formats, async I/O) - Use a library. The accumulated person-years of effort in established crates aren't something you can replicate.

2. **Is this my core domain?** (business logic, proprietary algorithms, competitive differentiators) - Write it yourself. Wrap infrastructure crates behind traits, but keep the logic yours.

3. **How many lines would the custom version be?** If it's under 100 lines and you understand every edge case, write it. The dependency overhead isn't worth it.

4. **Does the crate pass the evaluation checklist?** If downloads are declining, the maintainer is MIA, the license is weird, or it pulls in 60 transitive dependencies - look for alternatives or write your own.

5. **Can I wrap it?** If you do take the dependency, isolate it behind a trait boundary. If the [adapter pattern](/blog/the-adapter-pattern-in-rust-wrapping-external-apis/) taught us anything, it's that the cost of switching libraries drops to near zero when only one module knows about the external crate's types.

There's no universal answer. A startup shipping fast might accept more dependencies to move quickly. A team building safety-critical embedded firmware might vendor every dependency and audit it line by line. The framework above gives you a way to make the decision consciously rather than defaulting to `cargo add` every time.

The goal isn't zero dependencies - it's *intentional* dependencies. Every crate in your `Cargo.toml` should be there because you evaluated it and decided the trade-off was worth it, not because it was the first search result on crates.io.
