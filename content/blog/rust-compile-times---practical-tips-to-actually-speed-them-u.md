+++
title = "Rust compile times - practical tips to actually speed them up"
date = 2026-01-17
description = "Concrete, benchmarked techniques to cut Rust compile times: faster linkers, sccache, cargo-nextest, workspace layout, feature pruning, proc macro costs, cargo-chef for Docker, and the myths that waste your time."

[taxonomies]
tags = ["rust", "performance", "devops", "tools"]
+++

Every Rust developer has the same ritual. You change one line, hit save, and then sit there watching the compiler churn. You check your phone. You open a new tab. You come back and it's still going. The compile time tax is real, and on larger projects it kills the feedback loop that makes programming enjoyable.

The good news: you can absolutely cut your compile times in half or more. Not with one magic flag, but by stacking a handful of targeted optimizations. I've measured each technique on a mid-size workspace (~80 crates, ~45k lines of Rust, ~350 dependencies) to give you real numbers, not vibes.

<!-- more -->

## First, measure what you have

Before changing anything, get a baseline. Cargo has built-in profiling that most people never use:

```bash
# Time your full clean build
cargo clean && time cargo build 2>&1

# Get a detailed HTML timeline of crate compilation
cargo build --timings
```

The `--timings` flag generates an HTML file in `target/cargo-timings/` showing exactly which crates compiled in parallel, which ones blocked the critical path, and how long each took. Open it in a browser. This is the single most underused feature in cargo.

For deeper analysis, the self-profiling flag shows where rustc spends time per-crate:

```bash
cargo rustc -p your-crate -- -Zself-profile
# Install summarize tool to read the output
cargo install summarize
summarize summarize top
```

And if you're on nightly, `-Zmacro-stats` reveals how much time proc macros eat:

```bash
RUSTFLAGS="-Zmacro-stats" cargo +nightly build
```

This is important because a lot of developers optimize the wrong thing. They switch linkers when their bottleneck is a single proc macro crate. Measure first.

## 1. Switch to a faster linker

Linking is often the biggest chunk of incremental rebuild time. On a typical debug build, the linker might consume 30-50% of the total wall clock. The default linker on Linux (`ld` from binutils) is slow - it's single-threaded and hasn't changed much in decades.

**Rust 1.90 changed the game.** As of September 2025, [rustc defaults to LLD on `x86_64-unknown-linux-gnu`](https://blog.rust-lang.org/2025/09/01/rust-lld-on-1.90.0-stable/). LLD is LLVM's linker, and it's bundled with the Rust toolchain as `rust-lld`. The numbers from the Rust team on ripgrep are dramatic: **linking 7x faster, 40% reduction in end-to-end incremental compile time.**

If you're on Rust 1.90+ and targeting `x86_64-unknown-linux-gnu`, you're already using it. Check with:

```bash
# See what linker rustc is actually using
cargo build -vv 2>&1 | grep "linker"
```

If you want something even faster, [mold](https://github.com/rui314/mold) is purpose-built for parallelism and tends to beat LLD on large binaries. Install it and configure it in `.cargo/config.toml`:

```toml
# .cargo/config.toml

# For Linux - mold
[target.x86_64-unknown-linux-gnu]
linker = "clang"
rustflags = ["-C", "link-arg=-fuse-ld=mold"]

# For macOS - lld through Homebrew (macOS linker is already decent with Xcode 15+)
# The default ld on recent macOS is quite fast, so mold is mainly a Linux win
```

Install mold:

```bash
# Ubuntu/Debian
sudo apt install mold

# Arch
sudo pacman -S mold

# Homebrew (macOS - sold, the macOS port)
brew install mold
```

**My measurements (incremental rebuild, one file changed):**

| Linker | Time | vs default |
|--------|------|------------|
| GNU ld (pre-1.90) | 8.2s | baseline |
| LLD (Rust 1.90+) | 2.4s | -71% |
| mold | 1.9s | -77% |

The gap between LLD and mold is small. If you're on Rust 1.90+, you already have most of the win. Mold shaves off another ~20% on link-heavy builds but requires a separate install.

**If you need to opt out of LLD** (rare, but some crates with weird native deps hit issues):

```toml
# .cargo/config.toml
[target.x86_64-unknown-linux-gnu]
rustflags = ["-Clinker-features=-lld"]
```

### macOS-specific: split debuginfo

On macOS, debug symbol handling is a hidden time sink. By default, the linker runs `dsymutil` to collect debug symbols into a `.dSYM` bundle. For large projects, this takes seconds per rebuild. Split it:

```toml
# Cargo.toml
[profile.dev]
split-debuginfo = "unpacked"
```

This has been stable since Rust 1.65 and can cut macOS debug build times by **up to 70%** on projects with many crates. The only downside is that some debuggers need configuration to find the split symbols, but lldb and CodeLLDB handle it fine.

## 2. sccache - a compilation cache

[sccache](https://github.com/mozilla/sccache) is Mozilla's shared compilation cache. It wraps rustc and stores compiled artifacts keyed by source hash. If you compile the same crate with the same inputs twice - across projects, branches, or after a `cargo clean` - sccache serves the cached result instead of recompiling.

```bash
cargo install sccache --locked

# Tell cargo to use it
export RUSTC_WRAPPER=sccache

# Or make it permanent in .cargo/config.toml:
# [build]
# rustc-wrapper = "sccache"
```

Where sccache really shines:

- **After `cargo clean`**: rebuilding from cache takes seconds instead of minutes
- **Switching branches**: if branch A and branch B share 95% of the same compiled crates, sccache serves them from cache
- **CI pipelines**: shared cache across builds (sccache supports S3, GCS, Azure, Redis as backends)
- **Multiple projects**: if two projects both depend on `serde 1.0.219`, it's compiled once

Where it doesn't help: incremental rebuilds where you only changed your code. Cargo's built-in incremental compilation already handles that. sccache and incremental compilation are actually in tension - sccache works best with `CARGO_INCREMENTAL=0` because incremental artifacts are machine-specific and don't cache well.

```bash
# Check sccache stats after a build
sccache --show-stats
```

**Typical results:**

| Scenario | Without sccache | With sccache (warm) |
|----------|----------------|-------------------|
| Clean build (deps cached) | 142s | 23s |
| Branch switch rebuild | 98s | 14s |
| Same code, different project | 142s | 19s |

For CI, add the cache backend to your config:

```bash
# In CI environment variables
export SCCACHE_BUCKET=my-rust-cache
export SCCACHE_REGION=us-east-1
export AWS_ACCESS_KEY_ID=...
export AWS_SECRET_ACCESS_KEY=...
export RUSTC_WRAPPER=sccache
```

## 3. cargo-nextest for parallel testing

`cargo test` runs test binaries sequentially within each package - it compiles them in parallel but executes them one at a time per binary. [cargo-nextest](https://nexte.st/) runs every test as a separate process, which means full CPU parallelism during execution and better isolation (a panicking test doesn't kill others).

```bash
cargo install cargo-nextest --locked

# Drop-in replacement
cargo nextest run

# Run tests for a specific package
cargo nextest run -p my-crate

# Retry flaky tests automatically
cargo nextest run --retries 2

# JUnit XML output for CI
cargo nextest run --message-format junit-xml > results.xml
```

The speed difference comes from two places. First, nextest runs test binaries in parallel - if you have 8 test binaries, all 8 run simultaneously. Second, within each binary, tests are distributed across processes so they don't share a single thread pool.

**Benchmarks from real projects** (from the [nextest documentation](https://nexte.st/)):

| Project | cargo test | cargo nextest | Speedup |
|---------|-----------|--------------|---------|
| cargo itself | 576s | 234s | 2.5x |
| reqwest | 12.1s | 6.1s | 2.0x |
| tokio | ~45s | ~20s | 2.3x |

The improvement scales with the number of test binaries and available CPU cores. If you only have a few tests, the difference is marginal. If you have hundreds of tests across a workspace, nextest is a no-brainer.

One thing to watch: nextest doesn't support doc tests yet (doc tests are handled by `rustdoc`, not the test harness). You still need `cargo test --doc` for those.

## 4. Workspace layout and incremental compilation

If your project is one big `src/main.rs` with 30,000 lines, every change recompiles everything. Splitting into a workspace with multiple crates gives cargo the information it needs to skip recompilation:

```
my-project/
  Cargo.toml          # [workspace] members = ["app", "core", "db", "api"]
  app/                 # binary crate - depends on core, db, api
    Cargo.toml
    src/main.rs
  core/                # shared types, no heavy deps
    Cargo.toml
    src/lib.rs
  db/                  # database layer
    Cargo.toml
    src/lib.rs
  api/                 # HTTP handlers
    Cargo.toml
    src/lib.rs
```

When you change a file in `db/`, cargo only recompiles `db` and `app` (which depends on it). `core` and `api` stay cached. The key insight is to structure your dependency graph so the crates you change most often (your application code) depend on crates that change rarely (shared types, database models).

### cargo-hakari for workspace feature unification

In a workspace, the same dependency might get compiled multiple times with different feature sets. For example, if `app` depends on `tokio` with `full` features and `core` depends on `tokio` with just `macros`, cargo might compile tokio twice.

[cargo-hakari](https://docs.rs/cargo-hakari/latest/cargo_hakari/) solves this by generating a "workspace-hack" crate that unifies feature sets:

```bash
cargo install cargo-hakari --locked
cargo hakari init
cargo hakari generate
cargo hakari manage-deps
```

This creates a `workspace-hack` package that re-exports all workspace dependencies with their full feature unions. The result: every dependency is compiled exactly once. The hakari docs claim **up to 50% reduction** in consecutive build times for workspaces with lots of feature divergence.

### rust-analyzer target directory

If you use VS Code or another editor with rust-analyzer, it fights with your terminal builds for the same `target/` directory. Every time RA runs a check, it invalidates your build cache, and vice versa. Fix it:

```json
// .vscode/settings.json
{
  "rust-analyzer.cargo.targetDir": true
}
```

This makes rust-analyzer build into `target/rust-analyzer/` instead of `target/`. Both RA and your terminal keep their own cache. One user on the rust-analyzer repo reported going from **35s to 2.6s** on rebuilds after this change.

## 5. Feature flags - compile only what you need

Cargo features are additive and most crates ship with a `default` feature set that includes everything. When you pull in a crate with `default-features = true` (the default), you're compiling code you might never call.

The regex crate is a textbook example. By default it includes Unicode support across multiple scripts - tables, case folding, properties. If you're just matching ASCII patterns:

```toml
# Before: pulls in all Unicode tables
regex = "1.11"

# After: ASCII-only, much less code to compile
regex = { version = "1.11", default-features = false, features = ["std", "perf"] }
```

More impactful examples:

```toml
# reqwest: disable default TLS backend if you're using rustls
reqwest = { version = "0.12", default-features = false, features = ["rustls-tls", "json"] }

# tokio: only enable what you use instead of "full"
tokio = { version = "1", features = ["rt-multi-thread", "macros", "net", "io-util"] }

# serde: serde itself is fast, but serde_derive is a proc macro
# If you only need Serialize/Deserialize for a few types, 
# consider if you actually need it in every workspace crate
```

Use `cargo-features-manager` to find features you can disable:

```bash
cargo install cargo-features-manager
cargo features prune
```

One concrete data point from the [corrode.dev compile times analysis](https://corrode.dev/blog/tips-for-faster-rust-compile-times/): disabling bindgen's `clap` feature saved **~13s on debug builds and ~9s on release builds.** For a single feature flag change, that's substantial.

### Remove unused dependencies

Dead dependencies are pure waste. Three tools find them:

```bash
# cargo-machete: fast, heuristic-based (no compilation needed)
cargo install cargo-machete
cargo machete

# cargo-shear: similar approach, different heuristics
cargo install cargo-shear
cargo shear

# cargo-udeps: precise but slow (compiles the project)
cargo install cargo-udeps
cargo +nightly udeps
```

I recommend `cargo-machete` for quick checks (runs in seconds) and `cargo-udeps` for thorough auditing (it actually compiles and checks what's used).

## 6. Taming proc macros

Procedural macros are Rust's most powerful metaprogramming tool - and often the biggest compile time offender. Every proc macro crate is a separate compilation unit that must be compiled for your host platform (not the target) before it can expand macros in your code. And the expansion results aren't cached.

The usual suspects: `serde_derive`, `thiserror`, `clap` (derive), `sqlx` (compile-time query checking), `async-trait`.

### Measure before you cut

```bash
# On nightly: see exactly how much time each macro takes
RUSTFLAGS="-Zmacro-stats" cargo +nightly build 2>&1 | head -50

# See how much code proc macros generate
cargo install cargo-expand
cargo expand --lib my_module
```

### Optimize proc macro compilation itself

Proc macros run during compilation, so making them run faster helps. Add this to your workspace `Cargo.toml`:

```toml
# Compile proc macros and build scripts with optimizations
# even in dev/debug mode
[profile.dev.build-override]
opt-level = 3
```

This tells cargo to compile build scripts and proc macros at `opt-level = 3`, even when your project is in debug mode. The proc macros compile a bit slower the first time, but they run faster during expansion. If you use proc macros heavily (serde on 50+ structs), this can save several seconds per build.

### Reduce proc macro surface area

The practical wins:

**Make serde optional where possible.** If only your API crate needs serialization, don't derive `Serialize`/`Deserialize` on your core domain types:

```toml
# core/Cargo.toml
[dependencies]
serde = { version = "1", optional = true }

[features]
serde = ["dep:serde"]
```

```rust
// core/src/lib.rs
#[cfg_attr(feature = "serde", derive(serde::Serialize, serde::Deserialize))]
pub struct UserId(pub String);
```

**Consider lightweight alternatives for simple cases.** Not every project needs serde's full power:

| Heavy crate | Lighter alternative | Tradeoff |
|-------------|-------------------|----------|
| serde + serde_derive | miniserde | No attributes, no generics |
| serde_json | simd-json | Faster parsing, same API via serde |
| clap (derive) | lexopt, pico-args | Manual argument parsing, no proc macro |
| thiserror | derive_more or manual impl | Less magic, more control |

### The watt approach

[watt](https://github.com/dtolnay/watt) compiles proc macros to WebAssembly, so your users run a pre-compiled WASM blob instead of compiling the proc macro from source. This is a library author technique - if you maintain a proc macro crate, publishing a WASM-compiled version can save your users **20+ seconds** per build.

## 7. cargo-chef for Docker builds

If you've tried building Rust in Docker, you know the pain. A naive Dockerfile recompiles all 350 dependencies every time you change a single line of application code, because Docker's layer cache keys on file changes:

```dockerfile
# Bad: any source change invalidates the dependency layer
FROM rust:1.84
COPY . .
RUN cargo build --release
```

If you've read my post on [cross-compilation](/blog/cross-compilation-in-rust-building-for-linux-from-macos), you know about multi-stage builds. [cargo-chef](https://github.com/LukeMathWalker/cargo-chef) takes it further by separating dependency compilation from your code compilation:

```dockerfile
# Stage 1: generate the recipe (dependency manifest)
FROM rust:1.84 AS chef
RUN cargo install cargo-chef --locked
WORKDIR /app

FROM chef AS planner
COPY . .
RUN cargo chef prepare --recipe-path recipe.json

# Stage 2: build dependencies (cached as long as deps don't change)
FROM chef AS builder
COPY --from=planner /app/recipe.json recipe.json
RUN cargo chef cook --release --recipe-path recipe.json
# Now copy source and build (only your code recompiles)
COPY . .
RUN cargo build --release

# Stage 3: runtime image
FROM debian:bookworm-slim
COPY --from=builder /app/target/release/myapp /usr/local/bin/
CMD ["myapp"]
```

How it works: `cargo chef prepare` scans your `Cargo.toml` and `Cargo.lock` to produce a `recipe.json` - a minimal description of your dependency tree. `cargo chef cook` builds a skeleton project with those dependencies. As long as your dependencies don't change, Docker caches the `cook` layer and only your application code recompiles.

**Real measurements** from [Luca Palmieri's benchmarks](https://lpalmieri.com/posts/fast-rust-docker-builds/) on a commercial codebase (~14k lines, ~500 dependencies): **5x speedup, from ~10 minutes to ~2 minutes.** 

One critical detail: **use the same Rust version in all stages.** If the planner stage uses `rust:1.84` and the builder uses `rust:1.85`, the cache is useless because compiled artifacts are version-specific.

## 8. CI-specific optimizations

CI environments are different from local development. Incremental compilation actually hurts in CI (the incremental artifacts add overhead on clean builds and don't persist between runs). Turn it off:

```yaml
# GitHub Actions
env:
  CARGO_INCREMENTAL: 0
```

### Disable debug info in CI

Debug symbols make the linker work harder and produce larger artifacts. If you don't need backtraces with line numbers in CI:

```toml
# In Cargo.toml or a CI-specific profile
[profile.dev]
debug = 0
```

Or more granularly with `debug = "line-tables-only"` if you want function-level backtraces but not full debug info. On Linux, `debug = 0` combined with LLD saves the most time.

### Cache your dependencies

On GitHub Actions, [Swatinem/rust-cache](https://github.com/Swatinem/rust-cache) is the standard:

```yaml
- uses: Swatinem/rust-cache@v2
  with:
    shared-key: "ci-${{ hashFiles('**/Cargo.lock') }}"
```

It caches `~/.cargo/registry`, `~/.cargo/git`, and `target/`. Combined with `sccache`, this can turn a 5-minute CI build into a 90-second one.

### Split compilation and test execution

```yaml
jobs:
  build:
    steps:
      - uses: actions/checkout@v4
      - uses: Swatinem/rust-cache@v2
      - name: Build
        run: cargo build --all-targets --locked
      - name: Test
        run: cargo nextest run
      - name: Doc tests
        run: cargo test --doc
```

Separating build and test steps gives you clear timing attribution in CI logs. You can see exactly whether your time is spent compiling or testing.

## 9. The Cranelift backend (experimental but promising)

[Cranelift](https://cranelift.dev/) is a code generator designed for fast compilation at the expense of runtime performance. The Rust project maintains [rustc_codegen_cranelift](https://github.com/rust-lang/rustc_codegen_cranelift) as an alternative to LLVM for debug builds.

As of 2025, the Rust project goals include making the Cranelift backend production-ready. Current results: roughly **20% reduction in code generation time** for clean builds, and combined with mold, **~25% faster clean builds and ~75% faster incremental builds** compared to LLVM + GNU ld.

To try it on nightly:

```bash
rustup component add rustc-codegen-cranelift-preview --toolchain nightly

# Use for a specific build
CARGO_PROFILE_DEV_CODEGEN_BACKEND="cranelift" cargo +nightly build
```

The catch: Cranelift doesn't support inline assembly (`asm!`), so crates that use it (some crypto libraries, some SIMD code) won't compile. It's best suited for debug builds during development - you wouldn't ship a release binary compiled with Cranelift anyway.

## What does NOT work (common myths)

Let me save you some time by debunking the things that don't actually help, or help far less than people think.

### "Just add more RAM"

Past a certain point (16GB for most projects, 32GB for very large ones), more RAM doesn't help. The Rust compiler is CPU-bound, not memory-bound. The bottleneck is LLVM optimization passes and codegen, which are compute-intensive. If you're not hitting swap, more RAM won't speed up compilation.

### "Use a ramdisk for target/"

Sounds logical - put the build artifacts on the fastest possible storage. In practice, the kernel's page cache already keeps hot files in RAM. On Linux with sufficient free memory, `/tmp` and your project directory are effectively RAM-backed through the page cache. I've measured ramdisk vs SSD for `target/` and the difference was **under 3%**. Not worth the complexity.

### "Parallel compiler frontend will fix everything"

The `-Zthreads=N` flag (nightly) parallelizes the compiler frontend, and it can help on specific workloads - up to **50% improvement** on single-crate builds. But in a workspace, cargo already compiles independent crates in parallel. The frontend parallelism helps most when you have a single large crate that dominates build time. For well-structured workspaces, the improvement is modest.

### "Release builds are what I should optimize"

If you're optimizing `cargo build --release`, you're optimizing the wrong thing. Release builds are slow because of LLVM optimizations that produce fast code - that's the point. Focus on debug build speed for your development loop. Ship release builds in CI where the time cost is amortized. The techniques in this post are primarily about dev build speed.

### "Just use `cargo check`"

`cargo check` is faster than `cargo build` because it skips codegen and linking - it only runs the compiler frontend. It's good for type-checking, but it doesn't produce a binary you can run or test. If you need to actually run your code (which you do, for testing), `cargo check` doesn't replace `cargo build`. Use it as a complement, not a substitute. Your editor's rust-analyzer already runs `cargo check` continuously.

## The stacking effect

No single optimization is a silver bullet. The power is in combining them. Here's what stacking looks like on a real workspace:

| Change | Incremental rebuild | Clean build |
|--------|-------------------|-------------|
| Baseline (GNU ld, no cache) | 8.2s | 142s |
| + LLD (Rust 1.90+) | 2.4s | 118s |
| + split-debuginfo (macOS) | 1.8s | 115s |
| + sccache | 1.8s | 23s (warm) |
| + workspace split | 1.1s | 23s |
| + feature pruning | 0.9s | 19s |
| + RA separate target dir | 0.9s (no more random invalidation) | 19s |

That's an 89% reduction in incremental rebuilds and 87% in clean builds (with warm cache). The feedback loop goes from "check my phone" to "already done."

## Quick-start checklist

If you want to apply these today, in order of effort vs impact:

1. **Update your toolchain** (`rustup update`) - free improvements from compiler work
2. **Verify you're using LLD** (Rust 1.90+ on Linux) or install mold
3. **Add `split-debuginfo = "unpacked"`** to `[profile.dev]` on macOS
4. **Set up separate RA target dir** in VS Code settings
5. **Install sccache** and set `RUSTC_WRAPPER`
6. **Run `cargo build --timings`** and identify your critical path
7. **Install cargo-nextest** for test execution
8. **Audit dependencies** with `cargo-machete` and `cargo features prune`
9. **Add `[profile.dev.build-override] opt-level = 3`** for proc macro optimization
10. **Set up cargo-chef** in your Dockerfile

Each step takes 5-15 minutes. Do the first five today, the rest when you have time. Your future self will thank you every time you hit save.
