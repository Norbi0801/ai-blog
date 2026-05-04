+++
title = "Cargo workspaces - managing multi-crate Rust projects"
date = 2025-07-16
description = "How Cargo workspaces work under the hood: virtual vs root manifests, dependency inheritance, project organization patterns, publishing strategies, and the commands that make multi-crate projects manageable."

[taxonomies]
tags = ["rust", "cargo", "architecture", "tooling"]
+++

At some point, every Rust project outgrows a single crate. Maybe your binary is 15,000 lines and changing one module recompiles the whole thing. Maybe you have a library and a CLI that share types. Maybe you want to extract a reusable core that other projects can depend on. The moment you feel the urge to split, you need a workspace.

Cargo workspaces are not complicated, but they have depth that the official tutorial only scratches. This post covers the full picture: what workspaces actually are at the manifest level, how dependency inheritance saves you from toml duplication hell, how to organize a real multi-crate project, and how to publish workspace members without losing your mind.

<!-- more -->

## What a workspace actually is

A workspace is a set of Cargo packages that share a single `Cargo.lock` file and a single `target/` directory. That's it. Everything else - dependency inheritance, shared metadata, member selection - is built on top of this foundation.

The shared `target/` directory is the key design decision. When crate A and crate B both depend on `serde 1.0.219`, Cargo compiles serde once and both crates link against the same artifact. Without a workspace, each crate has its own `target/` directory and compiles serde independently. Multiply that by 50 transitive dependencies and you're doubling your clean build time for no reason.

The shared `Cargo.lock` enforces version consistency. If I already covered what `Cargo.lock` records and how version resolution works in [Understanding Cargo.lock](/blog/understanding-cargo-lock-when-to-commit-it-and-why) - the short version here is that a workspace produces exactly one lockfile at the workspace root, and every member resolves against it. No version drift between crates.

## Virtual manifests vs root manifests

There are two ways to declare a workspace, and the distinction matters more than people think.

### Root package workspace

The simpler form: you already have a crate, and you add `[workspace]` to its `Cargo.toml`:

```toml
# Cargo.toml
[package]
name = "my-app"
version = "0.1.0"
edition = "2024"

[workspace]
members = ["crates/*"]

[dependencies]
my-core = { path = "crates/my-core" }
```

Here `my-app` is both a regular package AND the workspace root. It has `[package]` and `[workspace]` in the same file. This is a root package workspace.

### Virtual workspace

The alternative: the root `Cargo.toml` has `[workspace]` but no `[package]`:

```toml
# Cargo.toml (virtual manifest)
[workspace]
members = [
    "crates/my-app",
    "crates/my-core",
    "crates/my-cli",
]
resolver = "2"
```

No package lives at the workspace root. Every crate lives in a subdirectory. The root `Cargo.toml` exists only to define the workspace.

### Which one should you use?

Virtual workspaces. Almost always.

Root package workspaces create a subtle hierarchy: one crate is "special" because it owns the workspace definition. This gets awkward when your project grows. What if you add a second binary? Now one binary is the workspace root and the other is "just a member," even though they're conceptually equal. It also means your root `Cargo.toml` mixes package configuration with workspace configuration, which makes inheritance (covered below) harder to reason about.

Virtual workspaces treat every member equally. The root manifest is pure workspace configuration. This is what most serious Rust projects use - [Bevy](https://github.com/bevyengine/bevy), [rustc itself](https://github.com/rust-lang/rust), [ripgrep](https://github.com/BurntSushi/ripgrep), [Deno](https://github.com/denoland/deno). It's the pattern I recommend.

One thing to note: virtual manifests require you to specify `resolver = "2"` explicitly. Root package workspaces with `edition = "2021"` or later get resolver v2 automatically, but virtual manifests have no `[package]` section to infer the edition from. If you forget, Cargo defaults to resolver v1, and you'll hit the feature unification pitfalls I wrote about in [Feature flags in Rust](/blog/feature-flags-in-rust-conditional-compilation-with-cfg). Just add `resolver = "2"` (or `"3"` on edition 2024) and forget about it.

## Dependency inheritance with workspace.dependencies

Before Rust 1.64, every crate in a workspace had to declare its own dependencies. If ten crates all used `serde = { version = "1.0", features = ["derive"] }`, you wrote that line ten times. Update the version? Ten files. Add a feature? Ten files. This was the number one complaint about workspaces.

[RFC 2906](https://rust-lang.github.io/rfcs/2906-cargo-workspace-deduplicate.html) fixed this. You now define dependencies once in the workspace root and inherit them in members:

```toml
# Root Cargo.toml (virtual manifest)
[workspace]
members = ["crates/*"]
resolver = "2"

[workspace.dependencies]
serde = { version = "1.0", features = ["derive"] }
tokio = { version = "1", features = ["rt-multi-thread", "macros"] }
anyhow = "1.0"
tracing = "0.1"
uuid = { version = "1", features = ["v4"] }

# Internal crates can be workspace dependencies too
my-core = { path = "crates/my-core" }
```

Then in each member:

```toml
# crates/my-app/Cargo.toml
[package]
name = "my-app"
version = "0.1.0"
edition = "2024"

[dependencies]
serde.workspace = true
tokio.workspace = true
anyhow.workspace = true
my-core.workspace = true

[dev-dependencies]
tokio = { workspace = true, features = ["test-util"] }
```

Notice the `[dev-dependencies]` line. You can inherit from the workspace AND add extra features. The features are additive - the member gets whatever the workspace defined plus whatever it adds locally. In this case `my-app`'s dev-dependencies get tokio with `rt-multi-thread`, `macros`, AND `test-util`.

There's a limitation worth knowing: you can't set `default-features = false` in the member if the workspace declaration has `default-features = true` (the default). The `default-features` value is inherited from `[workspace.dependencies]` and can't be overridden per-member. If you need a dependency without default features in some crates but with them in others, you can't use workspace inheritance for that dependency. Just declare it directly in those members.

Also: `optional = true` can't be set in `[workspace.dependencies]`. Optional dependencies are a member-level concern:

```toml
# In the member's Cargo.toml - this works
my-optional-dep = { workspace = true, optional = true }
```

## Package metadata inheritance with workspace.package

Dependencies aren't the only thing you can share. Common package metadata lives in `[workspace.package]`:

```toml
# Root Cargo.toml
[workspace.package]
version = "0.5.0"
edition = "2024"
authors = ["Your Name <you@example.com>"]
license = "MIT OR Apache-2.0"
repository = "https://github.com/you/my-project"
rust-version = "1.85"
```

Members inherit with the `.workspace = true` pattern:

```toml
# crates/my-core/Cargo.toml
[package]
name = "my-core"
version.workspace = true
edition.workspace = true
authors.workspace = true
license.workspace = true
repository.workspace = true
rust-version.workspace = true
description = "Core types for my-project"  # This stays per-crate
```

The `description` field is intentionally not inherited. Each crate needs its own description for crates.io. Same for `name` - obviously. But `version`, `edition`, `license`, `repository`, `rust-version` - sharing these eliminates an entire class of "oops I forgot to update that crate" bugs.

### Shared versioning vs independent versioning

Using `version.workspace = true` means all your crates share the same version number. This is the monorepo model - bump the workspace version, all crates move together. This works great when your crates are tightly coupled and always released together (like Bevy's dozens of `bevy_*` crates).

If your crates evolve independently - say, the core library is stable at 2.x while the CLI is still at 0.x - don't use `version.workspace = true`. Just set versions directly in each member. There's no shame in mixed inheritance. Use workspace inheritance where it helps, skip it where it doesn't.

## Organizing a real project

Here's a layout I've found works well for most mid-size projects:

```
my-project/
  Cargo.toml              # virtual manifest
  Cargo.lock              # shared, committed
  .cargo/
    config.toml           # linker settings, etc.
  crates/
    my-core/              # shared types, domain logic
      Cargo.toml
      src/lib.rs
    my-db/                # database layer
      Cargo.toml
      src/lib.rs
    my-api/               # HTTP handlers
      Cargo.toml
      src/lib.rs
    my-app/               # binary, ties everything together
      Cargo.toml
      src/main.rs
    my-cli/               # CLI binary (optional)
      Cargo.toml
      src/main.rs
```

The dependency graph flows downward:

```
my-app ──> my-api ──> my-core
   │                     ^
   └──> my-db ───────────┘

my-cli ──> my-core
   └──> my-db
```

The rules that make this work:

1. **`my-core` depends on nothing internal.** It contains your domain types, error types, shared traits. It should have minimal external dependencies - just `serde`, `thiserror`, maybe `uuid`. This crate changes rarely, so it stays cached during incremental builds.

2. **Leaf crates (`my-db`, `my-api`) depend only on `my-core`.** They don't depend on each other. This prevents circular-ish dependency chains and maximizes parallel compilation - Cargo can compile `my-db` and `my-api` simultaneously after `my-core` is done.

3. **Binary crates (`my-app`, `my-cli`) sit at the top.** They depend on everything but nothing depends on them. They're thin - just wiring. The `main.rs` initializes the runtime, sets up config, and delegates to library crates.

If you've read my post on [compile times](/blog/rust-compile-times-practical-tips-to-actually-speed-them-up), this layout is exactly the workspace split technique that reduced incremental rebuilds from 8.2s to 1.1s. The key insight: put the code you change most (application wiring) in the crate that has the fewest downstream dependents (the binary). Cargo only recompiles what changed and what depends on what changed.

### Path dependencies between members

Members reference each other with `path` dependencies:

```toml
# crates/my-app/Cargo.toml
[dependencies]
my-core = { path = "../my-core" }
my-db = { path = "../my-db" }
my-api = { path = "../my-api" }
```

Or, if you listed internal crates in `[workspace.dependencies]`:

```toml
# Root Cargo.toml
[workspace.dependencies]
my-core = { path = "crates/my-core" }

# crates/my-app/Cargo.toml
[dependencies]
my-core.workspace = true
```

The second approach is cleaner because the path is defined once. If you ever restructure your directory layout, you update one line in the root manifest instead of hunting through every member.

One subtlety: path dependencies are always resolved locally during development, but when you publish to crates.io, the `path` component is stripped and only the `version` matters. This is why published workspace members need a version field:

```toml
# Root Cargo.toml
[workspace.dependencies]
my-core = { path = "crates/my-core", version = "0.5.0" }
```

Without the `version`, publishing will fail because crates.io needs to know which version of `my-core` to require.

## Working with workspace members: the cargo commands

Most cargo commands accept package selection flags. Knowing them saves time.

### `-p` / `--package`: target a specific member

```bash
# Build only my-core
cargo build -p my-core

# Test only my-db
cargo test -p my-db

# Run the binary crate
cargo run -p my-app

# Check a specific crate (fast - no codegen)
cargo check -p my-api

# Clippy on one crate
cargo clippy -p my-core -- -D warnings
```

This is the flag you'll use most. When you're iterating on `my-db`, there's no reason to build `my-app` every time. `-p my-db` builds only that crate and its dependencies, skipping anything upstream.

### `--workspace`: operate on everything

```bash
# Test all crates
cargo test --workspace

# Check everything (good for CI)
cargo clippy --workspace -- -D warnings
```

### `--exclude`: operate on everything except

```bash
# Test everything except the slow integration test crate
cargo test --workspace --exclude my-integration-tests
```

The `--exclude` flag supports glob patterns: `--exclude "my-*-tests"`.

### `default-members`: what cargo does with no flags

When you run plain `cargo build` in a virtual workspace without specifying `-p`, Cargo needs to decide what to build. This is controlled by `default-members`:

```toml
[workspace]
members = ["crates/*"]
default-members = ["crates/my-app"]
resolver = "2"
```

Without `default-members`, a virtual workspace builds ALL members. With it, plain `cargo build` only builds the listed crates. I usually set this to the main binary - it's what I'm iterating on 90% of the time.

For non-virtual (root package) workspaces, the default member is the root package itself.

## Feature unification in workspaces

This is the sharp edge. I covered the mechanics of feature unification in my post on [feature flags](/blog/feature-flags-in-rust-conditional-compilation-with-cfg), but workspaces make it more visible.

When Cargo builds a workspace, it resolves the entire dependency graph at once. If `my-app` depends on `tokio` with `["full"]` and `my-core` depends on `tokio` with just `["macros"]`, Cargo doesn't compile tokio twice. It enables the *union* of all requested features: `["full"]` (which is a superset). This is correct behavior - tokio is compiled once, linked once, and everyone gets the features they need.

But it has consequences. If `my-core`'s tests pass when built alone (`cargo test -p my-core`), they might also pass when built as part of the workspace - not because `my-core` correctly declares its dependencies, but because `my-app`'s features bled through. Your `my-core` crate might accidentally rely on tokio features it never asked for. Everything works in the workspace, everything breaks when someone adds `my-core` as a standalone dependency.

Catch this in CI:

```bash
# Test each crate in isolation to verify its declared dependencies are complete
for crate in crates/*/; do
  echo "Testing $(basename $crate)..."
  cargo test --manifest-path "$crate/Cargo.toml"
done
```

This runs each crate against its own declared features, not the workspace union. If a crate depends on a feature it didn't declare, this will catch it.

If you're dealing with significant feature divergence across workspace members and it's costing you duplicate compilations, [cargo-hakari](https://docs.rs/cargo-hakari/latest/cargo_hakari/) generates a synthetic "workspace-hack" crate that unifies features explicitly. I measured its impact in [Rust compile times](/blog/rust-compile-times-practical-tips-to-actually-speed-them-up) - up to 50% reduction on workspaces with lots of divergence.

## Publishing workspace members

Publishing a single crate is `cargo publish`. Publishing a workspace is `cargo publish` times N, in the right order, with version bumps coordinated across inter-dependent crates. This is where workspaces go from "nice" to "I need tooling."

### The ordering problem

If `my-api` depends on `my-core`, you must publish `my-core` first. crates.io verifies that all dependencies exist and resolve correctly at publish time. If you publish `my-api` before `my-core`, the verification step fails because crates.io can't find `my-core` at the declared version.

For a simple workspace, you can do this by hand:

```bash
# 1. Bump versions in workspace.package or individual Cargo.tomls
# 2. Publish in dependency order
cargo publish -p my-core
cargo publish -p my-db
cargo publish -p my-api
cargo publish -p my-app
```

But remember: crates.io has a brief propagation delay. After publishing `my-core`, it might take a few seconds before `my-api` can resolve it. You might need to retry or add a small wait between publishes.

### Crates you don't want to publish

Not every workspace member belongs on crates.io. Integration test crates, example crates, internal tooling - mark them as unpublishable:

```toml
# crates/my-integration-tests/Cargo.toml
[package]
name = "my-integration-tests"
version = "0.0.0"
publish = false
```

`publish = false` tells both Cargo and humans: this crate is workspace-internal.

### Tooling for multi-crate releases

For workspaces with more than three or four publishable crates, manual publishing gets old fast. The ecosystem has solid tools:

[**cargo-workspaces**](https://github.com/pksunkara/cargo-workspaces) (the most popular): handles version bumps, git tagging, and publishing in dependency-compatible order. One command for the whole workflow:

```bash
cargo install cargo-workspaces

# Bump versions, create git tag, publish all members in order
cargo workspaces publish
```

[**release-plz**](https://github.com/MarcoIeni/release-plz): takes it further with changelog generation and GitHub release creation. It analyzes your git history, determines which crates changed, and only publishes what needs publishing. Integrates with CI as a GitHub Action.

[**cargo-release**](https://github.com/crate-ci/cargo-release): configurable release workflow with pre/post-release hooks, custom tag formats, and workspace-aware version bumps.

All three handle the dependency ordering problem. Pick whichever fits your workflow.

## Workspace-level configuration

A few things that live at the workspace root and apply to all members:

### Shared profiles

```toml
# Root Cargo.toml
[profile.dev]
opt-level = 0
debug = true

[profile.dev.build-override]
opt-level = 3  # Compile proc macros with optimizations

[profile.release]
lto = "thin"
strip = true
codegen-units = 1

[profile.dev.package.sqlx-macros]
opt-level = 3  # Speed up this specific slow dependency
```

Profile settings in the workspace root apply to all members. You can't override profiles per-member - they're workspace-global. This is intentional: since all members share one `target/` directory, they must share compilation settings.

### Shared patch and replace

```toml
# Root Cargo.toml
[workspace]
members = ["crates/*"]

[patch.crates-io]
# Use a local fork of a dependency across the entire workspace
some-crate = { path = "../my-fork-of-some-crate" }
```

`[patch]` and `[replace]` sections in the workspace root affect all members. This is useful for testing against unreleased versions or local forks without modifying each member's `Cargo.toml`.

### Lints

Since Rust 1.74, workspace-level lint configuration:

```toml
# Root Cargo.toml
[workspace.lints.rust]
unsafe_code = "forbid"

[workspace.lints.clippy]
all = "warn"
pedantic = "warn"
nursery = "warn"
unwrap_used = "deny"
```

Members opt in:

```toml
# crates/my-core/Cargo.toml
[lints]
workspace = true
```

This gives you consistent lint policy across the workspace without maintaining a `.clippy.toml` or passing flags to every `cargo clippy` invocation.

## Practical tips from real workspace maintenance

**Use `members = ["crates/*"]` with globs.** Adding a new crate is just `cargo new crates/my-new-thing --lib`. No need to edit the root `Cargo.toml` - the glob picks it up automatically. The `exclude` key handles exceptions:

```toml
[workspace]
members = ["crates/*"]
exclude = ["crates/experimental-thing-not-ready"]
```

**Keep your `.cargo/config.toml` at the workspace root.** Linker settings, target configuration, environment variables - they apply to all members and belong at the root. Cargo walks up the directory tree looking for `.cargo/config.toml`, so members in subdirectories inherit it automatically.

**Run `cargo test --workspace` in CI, but also test isolation.** The workspace test catches integration issues. The per-crate test catches accidental feature leaks. Do both.

**Use `cargo tree` at the workspace level to audit your full dependency graph:**

```bash
# Show the full dependency tree across all workspace members
cargo tree --workspace

# Find duplicates - same crate compiled at different versions
cargo tree --workspace --duplicates

# Find who depends on a specific crate
cargo tree --workspace --invert -p openssl-sys
```

`--duplicates` is particularly useful. If you see two versions of the same crate, your workspace might benefit from aligning version requirements in `[workspace.dependencies]`.

**Don't over-split.** A workspace with 50 crates where each crate is one module is not better than a workspace with 5 well-structured crates. Every crate boundary adds overhead: a `Cargo.toml` to maintain, inter-crate dependencies to declare, potential feature unification issues. Split when you have a clear reason - independent release cycles, significantly different dependency trees, reuse across projects, or compilation parallelism gains. Don't split for "cleanliness."

**Use `cargo doc --workspace --no-deps` to build documentation for all your crates at once.** The generated docs cross-link between workspace members automatically, which is great for internal documentation.

## What's happening under the hood

When you run `cargo build` in a workspace, Cargo's resolution pipeline works like this:

1. **Manifest discovery**: Cargo finds the workspace root (walking up from the current directory), reads `[workspace]`, expands globs, and loads every member's `Cargo.toml`.

2. **Unified resolution**: All members' dependencies are resolved together into a single dependency graph. This is where feature unification happens. The resolver (v1 or v2, depending on your `resolver` key) produces one set of resolved packages for the entire workspace.

3. **Lockfile sync**: The resolved graph is written to `Cargo.lock` at the workspace root. If `Cargo.lock` already exists, Cargo checks whether the existing resolutions still satisfy all constraints. If they do, it reuses them (this is why rebuilds are fast).

4. **Compilation plan**: Cargo builds a DAG (directed acyclic graph) of compilation units and determines maximum parallelism. Independent crates compile in parallel. This is why workspace layout matters for build speed - a wide, shallow dependency graph compiles faster than a deep, serial one.

5. **Shared artifacts**: All compiled artifacts go to `target/` at the workspace root. Debug artifacts go to `target/debug/`, release to `target/release/`. Binary crates produce executables in `target/debug/{crate-name}`. Library crates produce `.rlib` files that other workspace members link against.

The shared `target/` directory is why workspace builds are efficient. When `my-core` is compiled, its `.rlib` sits in `target/debug/deps/`. When Cargo compiles `my-app`, it finds `my-core`'s artifact already there. No recompilation. Change a line in `my-core`, and Cargo recompiles `my-core` + everything that depends on it. Change a line in `my-app`, and only `my-app` recompiles. The dependency graph determines the blast radius.

You can visualize this with `cargo build --timings`, which generates an HTML timeline showing exactly which crates compiled in parallel and which ones blocked the critical path. If you see a serialized chain of workspace members, it's a sign your internal dependency graph could be restructured for more parallelism.

## Wrapping up

Cargo workspaces solve three problems at once: they eliminate dependency duplication, they speed up builds through shared compilation artifacts, and they provide logical separation for projects that have outgrown a single crate. The workspace model is simple - shared `Cargo.lock`, shared `target/`, member selection with `-p` - but the details around dependency inheritance, feature unification, and publishing order are where the real knowledge lives.

Start with a virtual manifest. Put your crates in a `crates/` directory with a glob member pattern. Centralize your dependency versions with `[workspace.dependencies]`. Set `default-members` to whatever you're actively working on. And when it's time to publish, let `cargo-workspaces` or `release-plz` handle the ordering.

The official [Cargo workspace reference](https://doc.rust-lang.org/cargo/reference/workspaces.html) covers every configuration key, and [RFC 2906](https://rust-lang.github.io/rfcs/2906-cargo-workspace-deduplicate.html) explains the design decisions behind dependency inheritance. Both are worth reading if you want the complete picture.
