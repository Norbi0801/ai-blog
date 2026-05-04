+++
title = "GitHub Actions for Rust projects - a CI template you can steal"
date = 2025-09-19
description = "A complete, copy-paste GitHub Actions CI setup for Rust: fmt, clippy, test, multi-platform builds, caching, release binaries, Dependabot, and conventional commit enforcement."

[taxonomies]
tags = ["rust", "ci", "devops", "github-actions"]
+++

Rust's compiler is slow. Not slow like "takes a few extra seconds" slow. A medium-sized project with 200 dependencies takes 3-8 minutes for a clean debug build. Add three OS targets, clippy, rustfmt, and a test suite, and your CI pipeline is looking at 20-40 minutes per push if you set it up naively.

The good news: you can get this down to 5-10 minutes with proper caching and job parallelism. The bad news: most Rust CI setups I see on GitHub are either too minimal (just `cargo test` on Ubuntu) or overcomplicated (custom Docker images, hand-rolled caching, shell script spaghetti).

This post is the middle ground. A complete CI configuration you can drop into any Rust project, understand every line of, and extend as needed. Every YAML block here is production-tested.

<!-- more -->

## What a Rust CI pipeline actually needs

Before jumping into YAML, here's what a solid Rust CI should check on every pull request:

1. **Formatting** - `cargo fmt --check`. Catches inconsistent style before review.
2. **Linting** - `cargo clippy`. Catches common mistakes, unidiomatic code, and potential bugs.
3. **Compilation** - `cargo build`. Confirms the code actually compiles (sometimes clippy passes but build doesn't, especially with feature flags).
4. **Tests** - `cargo test`. Runs your test suite.
5. **Multi-platform** - at minimum Linux, macOS, and Windows. A crate that compiles on Linux but panics on Windows due to path separators is a common surprise.

You want these as separate jobs, not sequential steps in one job. Separate jobs run in parallel, and they fail independently - you see "clippy failed" instead of waiting 10 minutes for the build to finish before discovering a formatting issue.

## The CI workflow

Here's the full `.github/workflows/ci.yml`. I'll walk through each section after.

```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

# Cancel in-progress runs for the same branch/PR.
# Saves runner minutes when you push multiple commits quickly.
concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

env:
  CARGO_TERM_COLOR: always
  # Treat all warnings as errors in CI.
  # This catches new clippy lints that were added in a toolchain update.
  RUSTFLAGS: "-D warnings"

jobs:
  fmt:
    name: Formatting
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v6

      - uses: dtolnay/rust-toolchain@stable
        with:
          components: rustfmt

      - run: cargo fmt --all --check

  clippy:
    name: Clippy
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v6

      - uses: dtolnay/rust-toolchain@stable
        with:
          components: clippy

      - uses: Swatinem/rust-cache@v2

      - run: cargo clippy --all-targets --all-features -- -D warnings

  test:
    name: Test (${{ matrix.os }})
    runs-on: ${{ matrix.os }}
    strategy:
      fail-fast: false
      matrix:
        os: [ubuntu-latest, macos-latest, windows-latest]
    steps:
      - uses: actions/checkout@v6

      - uses: dtolnay/rust-toolchain@stable

      - uses: Swatinem/rust-cache@v2
        with:
          # Each OS gets its own cache bucket.
          shared-key: "test-${{ matrix.os }}"

      - run: cargo build --all-features
      - run: cargo test --all-features

  # Optional: check that your documented MSRV actually works.
  # Remove this job if you don't advertise an MSRV.
  msrv:
    name: MSRV (1.80.0)
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v6

      - uses: dtolnay/rust-toolchain@1.80.0

      - uses: Swatinem/rust-cache@v2

      - run: cargo build --all-features
```

### What's happening here

**`concurrency`** is the single most underused GitHub Actions feature for Rust projects. When you push three commits in a row, it cancels the first two runs and only runs the third. Without this, you're burning 3x the runner minutes for no reason.

**`RUSTFLAGS: "-D warnings"`** as a global env promotes every warning to an error. This means CI fails if clippy introduces new lints in a toolchain update. Strict, but it keeps the codebase clean. If this is too aggressive for your team, move it to just the clippy job.

**`fail-fast: false`** on the test matrix is important. The default (`true`) cancels all matrix jobs when one fails. That means if Windows fails, you won't see the macOS result. With `false`, all three finish, and you see exactly which platforms have problems.

**No `cargo build` in the fmt job.** Formatting doesn't need to compile anything. It checks syntax structure only. This job takes 5-10 seconds total.

**The MSRV job** uses a pinned toolchain version (`@1.80.0`) instead of `@stable`. This catches the scenario where you accidentally use a feature that requires a newer compiler than what you advertise in your `Cargo.toml`'s `rust-version` field. If you don't publish a crate or don't care about MSRV, remove this job.

### A note on runner changes

If you've been copying CI templates from blog posts written in 2024, update your runner assumptions. As of early 2026:

- `ubuntu-latest` is Ubuntu 24.04 (x64, 4 CPU, 16 GB RAM on public repos)
- `macos-latest` is macOS 15 on **Apple Silicon (arm64)**. It is no longer Intel. macOS 13 (the last Intel default) was retired in December 2025
- `windows-latest` is Windows Server 2025 (x64)
- ARM64 Linux runners (`ubuntu-24.04-arm`) are now available for both public and private repos

If you have an older workflow referencing `macos-13` for Intel builds, those runners are gone. If you need `x86_64-apple-darwin` binaries, you'll have to cross-compile from the ARM64 runner. I covered this setup in detail in [Cross-compilation in Rust](/blog/cross-compilation-in-rust-building-for-linux-from-macos).

## Caching with Swatinem/rust-cache

The `Swatinem/rust-cache@v2` action (currently at v2.9.1) is the de facto standard for caching Rust builds in GitHub Actions. It's smarter than a raw `actions/cache` because it understands Rust's build system.

What it caches:

- `~/.cargo/registry/index` and `~/.cargo/registry/cache` - downloaded crate metadata and `.crate` files
- `~/.cargo/git/db` - git dependency checkouts
- `target/` - but only the dependency artifacts, not your crate's own build output

That last point is the key design decision. Your own crate's artifacts are rebuilt every run (they're what changed, after all). Dependency artifacts are reused because they didn't change. This keeps the cache meaningful without growing unbounded.

The cache key includes your `Cargo.lock` hash, the Rust toolchain version, and the runner OS. When you update a dependency or your toolchain, the cache invalidates automatically.

For a workspace with multiple crates, add `workspaces` to tell it where your `Cargo.lock` lives:

```yaml
- uses: Swatinem/rust-cache@v2
  with:
    workspaces: "my-workspace -> target"
```

For projects that install additional cargo tools (like `cargo-nextest` or `cargo-tarpaulin`), cache them too:

```yaml
- uses: Swatinem/rust-cache@v2
  with:
    cache-targets: true
    cache-all-crates: true
```

A clean build of a project with 200 dependencies takes ~4 minutes. With a warm cache, it drops to ~40 seconds. That's a 6x speedup, and it's free.

## The release workflow

CI checks quality on every push. The release workflow builds binaries and publishes them when you tag a release. This goes in `.github/workflows/release.yml`.

If you've read my [cross-compilation post](/blog/cross-compilation-in-rust-building-for-linux-from-macos), you've already seen a matrix build. This extends it with artifact collection and GitHub Release uploads:

```yaml
name: Release

on:
  push:
    tags: ['v*']

permissions:
  contents: write

jobs:
  build:
    name: Build (${{ matrix.target }})
    runs-on: ${{ matrix.os }}
    strategy:
      fail-fast: false
      matrix:
        include:
          - target: x86_64-unknown-linux-gnu
            os: ubuntu-latest
            archive: tar.gz

          - target: aarch64-unknown-linux-gnu
            os: ubuntu-24.04-arm
            archive: tar.gz

          - target: x86_64-unknown-linux-musl
            os: ubuntu-latest
            archive: tar.gz

          - target: aarch64-apple-darwin
            os: macos-latest
            archive: tar.gz

          - target: x86_64-pc-windows-msvc
            os: windows-latest
            archive: zip

    steps:
      - uses: actions/checkout@v6

      - uses: dtolnay/rust-toolchain@stable
        with:
          targets: ${{ matrix.target }}

      - uses: Swatinem/rust-cache@v2
        with:
          shared-key: "release-${{ matrix.target }}"

      - name: Install musl tools
        if: contains(matrix.target, 'musl')
        run: sudo apt-get update && sudo apt-get install -y musl-tools

      - name: Build
        run: cargo build --release --target ${{ matrix.target }}

      - name: Package (unix)
        if: matrix.archive == 'tar.gz'
        run: |
          cd target/${{ matrix.target }}/release
          tar czf ../../../myapp-${{ matrix.target }}.tar.gz myapp
          cd -

      - name: Package (windows)
        if: matrix.archive == 'zip'
        shell: pwsh
        run: |
          Compress-Archive `
            -Path target/${{ matrix.target }}/release/myapp.exe `
            -DestinationPath myapp-${{ matrix.target }}.zip

      - name: Upload artifact
        uses: actions/upload-artifact@v7
        with:
          name: myapp-${{ matrix.target }}
          path: myapp-${{ matrix.target }}.*

  release:
    name: Publish Release
    needs: build
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v6

      - name: Download all artifacts
        uses: actions/download-artifact@v7
        with:
          path: artifacts
          merge-multiple: true

      - name: Create GitHub Release
        uses: softprops/action-gh-release@v2
        with:
          generate_release_notes: true
          files: artifacts/*
```

### Design decisions

**ARM64 Linux builds on `ubuntu-24.04-arm`** instead of cross-compiling. GitHub made ARM runners available in January 2026 for all repos. Native builds are faster and simpler than `cross` or `cargo-zigbuild`. No Docker, no emulation, no toolchain configuration.

**No `x86_64-apple-darwin` target.** Apple is phasing out Intel Macs. GitHub retired Intel macOS runners in December 2025. If you still need Intel Mac binaries, you'd have to cross-compile from the ARM runner (add `rustup target add x86_64-apple-darwin` and build with `--target x86_64-apple-darwin` on `macos-latest`). For most projects in 2026, arm64-only is fine.

**`generate_release_notes: true`** tells the `softprops/action-gh-release` action to auto-generate release notes from the commit history since the last tag. This works especially well if you use conventional commits - which I wrote about in detail in [Conventional commits - why and how to write meaningful commit messages](/blog/conventional-commits-why-and-how-to-write-meaningful-commit-messages).

**`merge-multiple: true`** on the download step flattens all artifacts into a single directory. Without this, you get nested folders like `artifacts/myapp-x86_64-unknown-linux-gnu/myapp-x86_64-unknown-linux-gnu.tar.gz`. With it, everything lands directly in `artifacts/`.

**Separate build and release jobs.** The `release` job has `needs: build`, so it only runs after all builds succeed. If the Windows build fails, no release is created. You don't end up with a half-populated release page.

### cargo-dist as an alternative

If you want something more automated, [cargo-dist](https://github.com/axodotdev/cargo-dist) (v0.31.0) generates the entire release workflow for you. Run `cargo dist init`, answer the prompts, and it creates a `.github/workflows/release.yml` with plan/build/host/publish stages, installers (shell, PowerShell, Homebrew), checksums, and more. It's opinionated but thorough. The manual approach above gives you full control; cargo-dist gives you batteries-included with less to maintain.

## Dependabot

Two things you want Dependabot watching: your Cargo dependencies and the GitHub Actions you use.

Create `.github/dependabot.yml`:

```yaml
version: 2
updates:
  # Rust/Cargo dependencies
  - package-ecosystem: "cargo"
    directory: "/"
    schedule:
      interval: "weekly"
      day: "monday"
    groups:
      # Batch minor/patch updates into one PR.
      # Major versions get individual PRs (they might break things).
      minor-and-patch:
        update-types:
          - "minor"
          - "patch"
    commit-message:
      prefix: "build"

  # GitHub Actions
  - package-ecosystem: "github-actions"
    directory: "/"
    schedule:
      interval: "weekly"
      day: "monday"
    commit-message:
      prefix: "ci"
```

The `groups` key is the important one. Without it, Dependabot opens one PR per updated crate. If you have 30 dependencies with patch updates, that's 30 PRs. With grouping, minor and patch updates are batched into a single PR. Major version bumps still get individual PRs because they're more likely to require code changes.

The `commit-message.prefix` follows the conventional commit spec. Dependabot will create commits like `build: bump serde from 1.0.210 to 1.0.215` and `ci: bump actions/checkout from v5 to v6`. Your changelog generator picks these up automatically.

One gotcha: Dependabot's `versioning-strategy: "increase-if-necessary"` is **not supported** for the cargo ecosystem ([dependabot-core#4009](https://github.com/dependabot/dependabot-core/issues/4009)). Only `increase` and `widen` work. This means Dependabot will bump the version in `Cargo.toml` even when the existing version range already covers the new version. Minor annoyance, no real harm.

## Enforcing conventional commits

If your team uses conventional commits (and you should - they make `generate_release_notes` and automated changelogs actually useful), enforce the format in CI. Add `.github/workflows/commits.yml`:

```yaml
name: Commits

on:
  pull_request:
    branches: [main]

jobs:
  conventional:
    name: Conventional Commits
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v6
        with:
          fetch-depth: 0

      - name: Check commit messages
        uses: webiny/action-conventional-commits@v1.3.0
        with:
          allowed-commit-types: >
            feat,fix,docs,style,refactor,perf,test,build,ci,chore,revert
```

`fetch-depth: 0` fetches the full git history so the action can inspect all commits in the PR, not just the latest one. The `allowed-commit-types` list matches the Angular convention. Adjust to your team's preferences.

If you're not familiar with the spec itself, I wrote a full breakdown in [Conventional commits - why and how to write meaningful commit messages](/blog/conventional-commits-why-and-how-to-write-meaningful-commit-messages).

## Hardening your workflows

A few things that separate "works" from "production-grade":

### Pin actions by commit SHA

Version tags like `@v6` are mutable. The maintainer can push a malicious update to the tag. SHA pinning eliminates this risk:

```yaml
# Instead of:
- uses: actions/checkout@v6
# Use:
- uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v6.0.2
```

Dependabot automatically updates SHA pins when new versions are released, so you get the security benefit without the maintenance burden.

### Minimal permissions

GitHub Actions workflows get `contents: read` and `metadata: read` by default on pull requests, but `contents: write` on push events. Restrict permissions explicitly:

```yaml
permissions:
  contents: read

jobs:
  # ...
```

Only the release workflow needs `contents: write` (to create the release). CI workflows should never have write access to your repo.

### Dependency review

GitHub's dependency review action catches known vulnerabilities in new dependencies before they're merged:

```yaml
  dependency-review:
    name: Dependency Review
    runs-on: ubuntu-latest
    if: github.event_name == 'pull_request'
    steps:
      - uses: actions/checkout@v6
      - uses: actions/dependency-review-action@v4
        with:
          fail-on-severity: moderate
```

This checks the diff of `Cargo.lock` against the GitHub Advisory Database. If a PR introduces a crate with a known CVE, the check fails. It's free for public repos and runs in ~10 seconds.

## Putting it all together

Here's the complete file tree of what you're adding to your project:

```
.github/
  dependabot.yml
  workflows/
    ci.yml          # fmt, clippy, test (3 OSes), MSRV
    release.yml     # build matrix + GitHub Release
    commits.yml     # conventional commit enforcement
```

Three files. No custom Docker images. No shell scripts. No third-party CI platforms.

The CI workflow runs on every push and PR, takes 5-8 minutes with a warm cache, and checks formatting, linting, compilation, and tests across Linux, macOS, and Windows. The release workflow triggers on version tags and produces downloadable binaries for every major platform. Dependabot keeps your dependencies and actions up to date with grouped PRs.

Copy the YAML blocks from this post into your repo, replace `myapp` with your binary name, adjust the MSRV version to match your `Cargo.toml`, and push. Your next PR will have green checkmarks.

## What this doesn't cover

A few things I intentionally left out to keep this focused:

- **Code coverage** with `cargo-tarpaulin` or `cargo-llvm-cov` and Codecov/Coveralls integration. Worth adding but not essential.
- **Benchmarking** with `criterion` and `github-action-benchmark`. Only relevant if you have performance-sensitive code.
- **Security auditing** with `cargo-audit` in CI. I covered SARIF and security tooling integration in [Understanding SARIF](/blog/understanding-sarif-the-standard-format-for-security-tools).
- **Docker image builds.** If you're containerizing your Rust app, check out [Docker multi-stage builds for Rust](/blog/docker-multi-stage-builds-for-rust-from-2gb-to-20mb).
- **Publishing to crates.io.** That's a `cargo publish` step with a `CARGO_REGISTRY_TOKEN` secret. Simple enough that it doesn't need its own section.

Each of those could be a section on its own. Maybe a follow-up post. For now, the template above covers what 90% of Rust projects need from CI.
