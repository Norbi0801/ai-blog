+++
title = "How to publish your first Rust crate on crates.io"
date = 2025-10-08
description = "Everything you need to know before hitting cargo publish for the first time: metadata, naming, semver, dry runs, yanking, badges, and automation with cargo-release."

[taxonomies]
tags = ["rust", "open-source", "crates-io", "tooling"]
+++

There are over 170,000 crates on [crates.io](https://crates.io/). Every Rust developer consumes them daily. But publishing one? That's the step most people never take. They have a utility module that could be a crate, or a small library they extracted from a project, but the process feels opaque enough to postpone forever.

It shouldn't. Publishing a crate takes about five minutes once you know what Cargo expects. The hard part isn't the mechanics - it's understanding the decisions that are permanent (your crate name, your published versions) versus the ones you can change later (metadata, docs, ownership). This post walks through all of it, from zero to a published crate, with the gotchas that the official docs gloss over.

<!-- more -->

## Before you touch Cargo.toml

### Get an account and API token

Go to [crates.io](https://crates.io/) and log in with your GitHub account. That's it - there's no separate registration.

Then generate an API token. Navigate to [Account Settings > API Tokens](https://crates.io/settings/tokens) and create a new token. Give it a descriptive name like `laptop-publish` - you might have multiple tokens for different machines or CI systems later.

```bash
cargo login <your-token-here>
```

This stores the token in `~/.cargo/credentials.toml`. Cargo reads it automatically on every `cargo publish`. You only do this once per machine.

One thing to note: the token has full publish access to every crate you own. There's no per-crate scoping. If your token leaks, revoke it immediately from the web UI and generate a new one. Scoped tokens (publish-only, per-crate) have been a long-requested feature but aren't available yet for traditional API tokens. If you want more granular control, look at Trusted Publishing (covered later in this post).

### Pick your crate name carefully

This is the one decision you can't undo. Once a crate name is taken, it's taken forever. Even if you yank every version, the name stays reserved under your account.

**Check availability before you get attached.** The simplest way:

```bash
cargo search my-cool-crate
```

If it returns nothing, the name is likely free. But "likely" isn't "definitely" - `cargo search` queries the index, which can lag behind. For a definitive check, just visit `https://crates.io/crates/my-cool-crate` directly. A 404 means it's available.

**Naming conventions in the Rust ecosystem:**

- Use lowercase with hyphens: `my-crate`, not `my_crate` or `MyCrate`. Cargo treats hyphens and underscores as equivalent (`my-crate` and `my_crate` are the same name), but hyphens are the convention on crates.io. In your Rust code, `use my_crate::...` - Cargo handles the translation.
- Prefix with your project name if it's a family of crates: `tokio-util`, `serde-json`, `axum-extra`.
- Avoid generic names like `utils`, `helpers`, `core`. They'll either be taken already or tell users nothing about what your crate does.
- Don't squat names. crates.io has an [explicit policy against name squatting](https://crates.io/policies) - publishing empty crates to reserve names can get your crates removed by the crates.io team.

**Think about discoverability.** When someone searches for "json parser" or "http client," will your crate name help or hurt? A name like `speedy-json` tells users more than `sjp`. The `keywords` and `categories` fields in `Cargo.toml` help too, but the name is the first thing people see.

## The Cargo.toml metadata

Here's a `Cargo.toml` ready for publishing. I'll go through each field:

```toml
[package]
name = "my-cool-crate"
version = "0.1.0"
edition = "2024"
authors = ["Your Name <you@example.com>"]
description = "A one-line description of what this crate does"
license = "MIT OR Apache-2.0"
repository = "https://github.com/yourname/my-cool-crate"
homepage = "https://github.com/yourname/my-cool-crate"
documentation = "https://docs.rs/my-cool-crate"
readme = "README.md"
keywords = ["parser", "json", "serialization"]
categories = ["parser-implementations"]
exclude = ["tests/fixtures/*", ".github/*"]

[dependencies]
# ...
```

### Required fields

Cargo will refuse to publish without these:

- **`name`** - your crate name, matching what you checked for availability.
- **`version`** - semver version string. Start with `0.1.0` (more on semver below).
- **`description`** - a single sentence. This appears in search results on crates.io and in `cargo search` output. Make it count.
- **`license`** or **`license-file`** - an [SPDX expression](https://spdx.org/licenses/). The most common choice in the Rust ecosystem is `MIT OR Apache-2.0` (dual license), which is what the Rust project itself uses. If you're not sure which license to pick, I wrote a detailed breakdown in [AGPL vs MIT vs Apache - choosing a license for your open source project](/blog/agpl-vs-mit-vs-apache-choosing-a-license-for-your-open-source-project/).

### Recommended fields

Not required, but you'll look unprofessional without them:

- **`repository`** - link to your source code. crates.io renders this as a clickable link.
- **`homepage`** - can be the same as `repository` if you don't have a separate site.
- **`documentation`** - defaults to `docs.rs/your-crate` if omitted. docs.rs automatically builds and hosts documentation for every published crate. You only need to set this if you host docs elsewhere.
- **`readme`** - path to your README file, relative to `Cargo.toml`. crates.io renders this as the main page for your crate. More on this below.
- **`keywords`** - up to 5 keywords. Used for search on crates.io. Lowercase, no spaces, hyphens allowed.
- **`categories`** - must match the [predefined category slugs](https://crates.io/category_slugs). You can't make up your own categories. Common ones: `command-line-utilities`, `web-programming`, `parser-implementations`, `data-structures`, `cryptography`.
- **`edition`** - the Rust edition. Set it to `2024` for new crates in 2026.
- **`rust-version`** - the minimum supported Rust version (MSRV). Example: `rust-version = "1.82"`. This tells users upfront if your crate will work with their toolchain. Cargo checks this during dependency resolution and will reject your crate if the user's Rust is too old.

### The `exclude` and `include` fields

By default, Cargo packages everything in your project directory except `target/` and a few other known paths. This means test fixtures, CI configs, screenshots, and other junk can end up in your `.crate` file.

crates.io has a **10 MB size limit** on uploaded packages. Even if you're under that, there's no reason to ship test data to every user who runs `cargo add your-crate`.

Use `exclude` to blacklist paths:

```toml
exclude = ["tests/fixtures/*", "benches/data/*", ".github/*", "*.png"]
```

Or use `include` to whitelist only what you need (more restrictive, less chance of accidentally shipping something):

```toml
include = ["src/**/*", "Cargo.toml", "LICENSE*", "README.md"]
```

You can inspect what will be packaged before publishing:

```bash
cargo package --list
```

This prints every file that would go into the `.crate` archive. Review the list. If you see `tests/fixtures/10mb-binary-blob.bin` in there, fix your excludes.

## Dry run first. Always.

```bash
cargo publish --dry-run
```

This does everything a real publish does except the upload:

1. Packages your crate into a `.crate` file (same as `cargo package`)
2. Extracts that `.crate` into a temporary directory
3. Builds it from scratch in that temp directory

That third step is the important one. It catches problems like:

- Files you reference in your code but forgot to include in the package
- `build.rs` scripts that depend on files outside the package
- Path dependencies that won't exist on the user's machine

What it does *not* catch (this is a [known limitation](https://github.com/rust-lang/cargo/issues/14249)): missing metadata fields like `license` or `description`. The crates.io server validates those, not the local dry-run. So you can have a successful dry-run and still get rejected on upload if your `description` is missing.

Run it, fix any warnings, run it again. Only then:

```bash
cargo publish
```

That's it. Your crate is live. It will appear on crates.io within seconds, and docs.rs will start building your documentation within a few minutes.

## Semver - what your version number actually promises

You set `version = "0.1.0"` in your `Cargo.toml`. What does that mean?

[Semantic versioning](https://semver.org/) (semver) encodes compatibility promises into version numbers: `MAJOR.MINOR.PATCH`.

- **PATCH** (0.1.0 -> 0.1.1): bug fixes only. No new API, no changed behavior.
- **MINOR** (0.1.0 -> 0.2.0): new features, backwards compatible. Existing code that compiled against 0.1.0 still compiles.
- **MAJOR** (0.x.y -> 1.0.0): breaking changes. Functions removed, types changed, behavior altered.

Sounds simple. The subtlety is in how Cargo interprets this for **pre-1.0 crates** (anything with major version 0).

### The 0.x special case

Standard semver says "anything goes before 1.0.0 - no stability guarantees." Cargo adapts this with a left-shift rule:

| Version change | Meaning in semver (1.x+) | Meaning in Cargo (0.x) |
|---|---|---|
| 0.1.0 -> 0.1.1 | patch (compatible) | patch (compatible) |
| 0.1.0 -> 0.2.0 | minor (compatible) | **breaking** (treated like major) |
| 0.0.1 -> 0.0.2 | patch (compatible) | **breaking** (treated like major) |

In practice, this means: if your crate is `0.x.y`, bumping the middle number (minor) signals a breaking change. Cargo's dependency resolver treats `0.1.x` and `0.2.x` as incompatible.

This is why most crates stay at `0.x` for a long time. Going to `1.0` is a signal that your API is stable and you're committed to semver discipline. Many popular crates took years to reach 1.0 - `serde` hit 1.0 in 2017, three years after its initial release. `tokio` reached 1.0 in December 2020.

### What counts as a breaking change?

The obvious ones: removing a public function, changing a function's return type, adding a required parameter. The [Cargo semver reference](https://doc.rust-lang.org/cargo/reference/semver.html) has an exhaustive list, but here are the ones that catch people off guard:

- **Adding a new public item** can break code that uses glob imports (`use your_crate::*`) if the new name conflicts with something the user already has in scope.
- **Adding a variant to a non-`#[non_exhaustive]` enum** breaks match statements.
- **Adding a field to a struct** (if it's not `#[non_exhaustive]`) breaks struct literal construction.
- **Tightening trait bounds** on a generic function.
- **Bumping a dependency's major version** if that dependency's types appear in your public API.

For catching accidental breakage, [`cargo-semver-checks`](https://github.com/obi1kenobi/cargo-semver-checks) is excellent. It scans your crate's public API against the previously published version and flags violations:

```bash
cargo install cargo-semver-checks
cargo semver-checks
```

It's not perfect - it works from rustdoc JSON output, so it can miss some categories of breakage (like type changes in struct fields). But it catches the majority of common mistakes, and there's active work to [integrate it directly into `cargo publish`](https://rust-lang.github.io/rust-project-goals/2024h2/cargo-semver-checks.html).

## Your README renders on crates.io

When someone visits your crate's page on crates.io, the first thing they see is your README rendered as HTML. This is your crate's landing page. It should make a developer want to try your crate within 30 seconds of reading it.

What to include:

1. **One sentence saying what this crate does.** Not "a Rust library for X" - just what it does. "Parse JSON five times faster than serde_json, with no unsafe code."
2. **A usage example.** Cargo.toml dependency line + a minimal code snippet that compiles. This is the most important part. Developers evaluate crates by scanning the example.
3. **Installation instructions.** Just `cargo add your-crate` - keep it simple.
4. **Feature flags**, if you have any. A table with flag names and what they enable.
5. **MSRV** (minimum supported Rust version), if you track it.
6. **License.** A one-liner at the bottom.

If you want a deeper dive into writing effective documentation, I covered `cargo doc`, doc comments, doc tests, and README structure in [Writing documentation that developers actually read](/blog/writing-documentation-that-developers-actually-read/).

### Badges

Badges at the top of your README give users instant status information. Here are the useful ones for a crate:

```markdown
[![Crates.io](https://img.shields.io/crates/v/my-cool-crate)](https://crates.io/crates/my-cool-crate)
[![docs.rs](https://img.shields.io/docsrs/my-cool-crate)](https://docs.rs/my-cool-crate)
[![CI](https://github.com/yourname/my-cool-crate/actions/workflows/ci.yml/badge.svg)](https://github.com/yourname/my-cool-crate/actions/workflows/ci.yml)
[![License](https://img.shields.io/crates/l/my-cool-crate)](https://crates.io/crates/my-cool-crate)
[![Downloads](https://img.shields.io/crates/d/my-cool-crate)](https://crates.io/crates/my-cool-crate)
```

These pull live data from [shields.io](https://shields.io/). The crates.io version badge auto-updates when you publish a new version. The docs.rs badge shows whether your documentation builds successfully. The CI badge shows your latest test status.

Don't go overboard. Five badges max. Nobody needs a badge for "Rust 2024 edition" or "works on Linux." Badges should convey information that changes over time.

One gotcha: crates.io used to have a `[badges]` section in `Cargo.toml` for configuring badges. That feature was [deprecated and removed](https://github.com/rust-lang/crates.io/issues/2436). Put badges in your README directly.

### Links from your crate page

crates.io pulls several links from your `Cargo.toml` and displays them in the sidebar:

- **Repository** - from the `repository` field
- **Documentation** - from `documentation`, or auto-links to docs.rs
- **Homepage** - from `homepage`

Fill these in. A crate without a repository link looks abandoned or untrustworthy, even if the code is solid.

## Yanking vs unpublishing

You published version `0.2.0` and realized it has a bug. Or a security issue. Or you accidentally published with a debug dependency. What do you do?

### What you can do: yank

```bash
cargo yank --version 0.2.0
```

Yanking marks a version as "don't use this for new projects." The practical effect:

- `cargo add your-crate` will skip yanked versions and install the latest non-yanked version instead.
- **Existing `Cargo.lock` files that already reference 0.2.0 continue to work.** The yanked version isn't deleted - it's still downloadable. Builds that were working before won't break.
- You can un-yank if you change your mind: `cargo yank --version 0.2.0 --undo`.

### What you cannot do: delete

You cannot delete a published version. You cannot overwrite a published version. You cannot delete your crate. This is by design.

The reasoning comes from npm's `left-pad` incident in 2016. Azer Koculu unpublished a package from npm, and thousands of builds worldwide broke instantly because their dependency no longer existed. npm had to intervene and restore the package against the author's wishes.

crates.io learned from this. From the [crates.io policies](https://crates.io/policies): "a publish is permanent. The version can never be overwritten, and the code cannot be deleted." The crates.io team can remove crates that violate policies (malware, name squatting), but that's an admin action, not something authors can do.

This permanence has implications:

- **Never publish secrets.** If you accidentally publish an API key in your source code, yanking the version doesn't remove it. The source is still in the crate archive. Rotate the secret immediately.
- **Think before you publish.** A throwaway test publish pollutes your crate's version history forever. Use `--dry-run` for testing.
- **Version numbers are consumed permanently.** If you publish `0.3.0` and yank it, you can't publish a different `0.3.0` later. You'll need to use `0.3.1`.

## The full publish workflow

Putting it all together. Here's the sequence I follow before every publish:

```bash
# 1. Make sure tests pass
cargo test

# 2. Make sure clippy is clean
cargo clippy -- -D warnings

# 3. Make sure formatting is correct
cargo fmt -- --check

# 4. Check for semver violations against the last published version
cargo semver-checks

# 5. Review what files will be packaged
cargo package --list

# 6. Dry run - packages and verifies the crate builds from the archive
cargo publish --dry-run

# 7. If everything looks good, publish for real
cargo publish
```

Steps 1-3 should already be in your CI. If you need a CI setup, I wrote a complete template in [GitHub Actions for Rust projects](/blog/github-actions-for-rust-projects-a-ci-template-you-can-steal/). Step 4 requires `cargo install cargo-semver-checks` but is worth the effort.

## Automating releases with cargo-release

Running seven commands manually for every release gets old fast. [`cargo-release`](https://github.com/crate-ci/cargo-release) wraps the entire release workflow into a single command:

```bash
cargo install cargo-release
```

Basic usage:

```bash
# Bump patch version, commit, tag, publish
cargo release patch

# Bump minor version
cargo release minor

# Bump to a specific version
cargo release 1.0.0
```

What `cargo release patch` does, step by step:

1. Verifies your working directory is clean (no uncommitted changes)
2. Bumps the version in `Cargo.toml` (e.g., `0.1.0` -> `0.1.1`)
3. Updates `Cargo.lock`
4. Commits the version bump
5. Creates a git tag (`v0.1.1`)
6. Pushes the commit and tag to your remote
7. Runs `cargo publish`

All of this is configurable. You can add pre-release hooks (to update a CHANGELOG, for example), skip the git push, customize the tag format, or run additional verification steps.

Configuration goes in `Cargo.toml` or a `release.toml` file:

```toml
# In Cargo.toml
[package.metadata.release]
sign-commit = true
sign-tag = true
pre-release-hook = ["git-cliff", "-o", "CHANGELOG.md"]
pre-release-commit-message = "release: v{{version}}"
tag-message = "v{{version}}"
```

Or if you prefer a separate file, create `release.toml` in your project root:

```toml
sign-commit = true
sign-tag = true
pre-release-hook = ["git-cliff", "-o", "CHANGELOG.md"]
pre-release-commit-message = "release: v{{version}}"
tag-message = "v{{version}}"
```

For first-time publishers, I'd recommend running your first release or two manually to understand each step. Once you've done it a few times and trust the process, switch to `cargo-release` to eliminate the repetitive parts.

### Dry run with cargo-release

```bash
cargo release patch --no-publish
```

The `--no-publish` flag does everything except the actual `cargo publish`. Useful for verifying the version bump, commit message, and tag format look right. When you're satisfied:

```bash
cargo release patch --execute
```

The `--execute` flag is required (previously the default was to actually run, but newer versions default to dry-run for safety).

## Trusted Publishing from CI

If you want to publish from GitHub Actions without storing API tokens as secrets, crates.io now supports [Trusted Publishing](https://doc.rust-lang.org/cargo/reference/registries.html#trusted-publishing) via OpenID Connect (OIDC). Instead of a long-lived API token, your GitHub Actions workflow gets a short-lived token (30 minutes) scoped to the specific repository and workflow that's running.

### Setting it up

1. Go to your crate's page on crates.io -> Settings -> Trusted Publishing
2. Add a trusted publisher configuration:
   - Repository owner: `yourname`
   - Repository name: `my-cool-crate`
   - Workflow filename: `release.yml`
   - Environment: `release` (optional, but recommended for an extra approval gate)

3. In your GitHub Actions workflow:

```yaml
name: Release
on:
  push:
    tags:
      - 'v*'

jobs:
  publish:
    runs-on: ubuntu-latest
    environment: release  # matches the crates.io config
    permissions:
      id-token: write     # required for OIDC
      contents: read
    steps:
      - uses: actions/checkout@v5

      - uses: dtolnay/rust-toolchain@stable

      - name: Authenticate with crates.io
        uses: rust-lang/crates-io-auth-action@v1

      - name: Publish
        run: cargo publish
```

The `rust-lang/crates-io-auth-action` exchanges your GitHub OIDC token for a temporary crates.io publish token. No secrets stored in your repo settings. No tokens to rotate. If someone forks your repo and runs the workflow, the OIDC claim won't match your crates.io configuration, so the publish will fail.

This is now the recommended approach for CI publishing. The crates.io team also added an option to enforce Trusted Publishing - you can disable traditional API token publishing entirely, ensuring that only your CI pipeline can publish new versions.

Trusted Publishing also supports GitLab CI/CD, not just GitHub Actions. The setup is analogous, using GitLab's OIDC token instead.

## Managing ownership

You published the crate. Now you want your co-maintainer to be able to publish too.

```bash
# Add another user as an owner
cargo owner --add github-username

# Add a GitHub team (org:team format)
cargo owner --add github:rust-lang:core

# Remove an owner
cargo owner --remove github-username

# List current owners
cargo owner --list
```

Owners can publish new versions and add/remove other owners. Be careful with this - there's no "publish-only" role. Anyone you add as an owner gets full control.

A common pattern for organizations: create a GitHub team for crate maintainers and add the team as an owner. This way, managing who can publish is managed through your GitHub org, not through individual crates.io commands.

## After publishing: what to expect

### docs.rs builds your documentation

Within minutes of publishing, [docs.rs](https://docs.rs/) will build your crate's documentation from your doc comments and host it. No configuration needed - it runs `cargo doc` against your crate and serves the output.

If your crate needs non-default features enabled for docs, add this to `Cargo.toml`:

```toml
[package.metadata.docs.rs]
all-features = true
```

Or specify exactly which features:

```toml
[package.metadata.docs.rs]
features = ["serde", "async"]
```

If docs.rs fails to build your crate (missing system dependencies, build script issues), you'll see a build failure on docs.rs. Check your crate's docs.rs page after publishing. A crate without working documentation is a crate that won't get adopted.

### Version downloads start counting

crates.io tracks download counts per version and total. These numbers appear on your crate page and in search results. Early on, the numbers will be small - that's fine. Download counts grow organically as people discover your crate through search, blog posts, word of mouth, or as a transitive dependency of something popular.

### People will open issues

If your crate is useful, people will use it. If people use it, they'll find bugs, request features, and ask questions. This is a good thing, but it requires maintenance effort. If you're not prepared to maintain the crate long-term, say so in the README: "This crate is provided as-is, with no guarantee of ongoing maintenance." Honesty upfront is better than silence on a year-old issue. For more on the maintainer side of open source, I covered the dynamics in [Open source etiquette - how to contribute and maintain projects](/blog/open-source-etiquette-how-to-contribute-and-maintain-projects/).

## Common mistakes I've seen

**Publishing with path dependencies.** If your `Cargo.toml` has `some-dep = { path = "../some-dep" }`, Cargo will reject the publish. All dependencies must come from a registry. If `some-dep` is your own crate, publish it first, then reference it by version.

**Forgetting to commit generated files.** If your crate uses `build.rs` to generate code, make sure the generated files are either included in the package or that `build.rs` runs correctly in isolation. The dry-run catches this, which is why you run it.

**Publishing from a dirty working directory.** Always publish from a clean git state. Uncommitted changes can accidentally make it into the `.crate` file. `cargo-release` enforces this by default.

**Starting at version 1.0.0.** Unless your API is genuinely stable and battle-tested, start at `0.1.0`. Going to 1.0 is a public commitment to semver stability. You can always go to 1.0 later. You can never go back.

**No README.** Your crate page on crates.io will be blank. Nobody will use a crate with a blank page.

**Giant `.crate` files.** You included your test fixtures, screenshots, or the entire `node_modules` directory from a sibling project. Run `cargo package --list` and check the size. The output also shows the compressed size - aim for well under 1 MB for a typical library crate.

## Quick reference

Here's the shortest possible path from "I have code" to "it's on crates.io":

```bash
# One-time setup
cargo login <token-from-crates.io>

# In your project
# 1. Fill out Cargo.toml metadata (name, version, description, license)
# 2. Write a README.md
# 3. Test it
cargo publish --dry-run

# 4. Ship it
cargo publish
```

Four steps. The rest of this post is about doing it well, but doing it at all is the most important part. Your first crate doesn't need to be perfect. It doesn't need badges, CI publishing, or a release automation setup. It needs to exist, have a description, and have a README with an example.

Everything else you can iterate on. The crate name and version history are permanent. The rest is just metadata updates and new versions.
