+++
title = "Git workflows for solo developers"
date = 2025-09-16
description = "You don't need GitFlow - here's a practical trunk-based workflow with pre-commit hooks, rebase strategies, and the rescue tools that save solo developers from themselves."

[taxonomies]
tags = ["git", "devops", "tools", "rust"]
+++

Every Git workflow article assumes you're on a team of ten. There's a release branch, a develop branch, hotfix branches, release candidate tags, and a branching diagram that looks like a subway map. GitFlow was designed for coordinating multiple developers shipping versioned software on fixed release cycles. If you're working solo - side projects, freelance work, early-stage products - that's overhead you don't need.

Solo development has different constraints. There's no merge conflict from a coworker pushing to the same branch. There's no release manager gating deployments. The risk isn't coordination failure - it's losing work, shipping broken code because nobody reviewed it, and ending up with a git history so messy that `git log` is useless when you come back to the project in three months.

Here's the workflow I use for every solo project. It's simple, it catches mistakes before they hit the remote, and it keeps your history clean enough that `git bisect` actually works when you need it.

<!-- more -->

## Trunk-based development with short-lived feature branches

The core idea: `main` is always deployable. Every change starts on a feature branch, gets completed, and merges back into `main`. No `develop` branch. No `release/*` branches. No long-lived branches at all.

```
main ─────●──────●──────●──────●──────●─────
           \    /         \    /        \
            feat/auth     fix/parse     feat/cache
```

The rules are minimal:

1. `main` is protected - you never commit directly to it
2. Feature branches are short-lived - hours to a few days, not weeks
3. Each branch does one thing - a feature, a fix, a refactor
4. Merge back when it's done, then delete the branch

Branch naming follows a simple convention: `feat/description`, `fix/description`, `refactor/description`, `docs/description`. The prefix maps directly to your commit types. More on that in a moment.

Starting a new feature:

```bash
git checkout main
git pull origin main
git checkout -b feat/rate-limiter
```

That's it. No branching off `develop`. No checking which release train you're targeting. You branch from `main`, you work, you merge back to `main`.

### Why not just commit directly to main?

You could. For tiny projects, some developers do. But feature branches give you three things even as a solo dev:

**Atomic rollback.** If your feature branch turns out to be a bad idea, you delete the branch and `main` is untouched. No need to revert a chain of commits.

**Clean diffs for self-review.** Before merging, run `git diff main...feat/rate-limiter`. You see exactly what this feature changes. It's your own code review. I catch bugs this way regularly - things that are obvious when you see the full diff but invisible when you're deep in implementation.

**CI integration.** If you have GitHub Actions or any CI, you can run checks on feature branches before merging. Even solo, automated tests catching a regression before it hits `main` is worth the 30-second overhead of creating a branch.

## Conventional commits (the short version)

I wrote a deep dive on the Conventional Commits spec, tooling, and automation in [Conventional Commits - why and how to write meaningful commit messages](/blog/conventional-commits-why-and-how-to-write-meaningful-commit-messages/). If you haven't read it, that post covers the spec, scopes, breaking change notation, and tools like [git-cliff](https://git-cliff.org/) and [cocogitto](https://docs.cocogitto.io/) for automated changelogs.

The short version for your daily workflow: every commit message starts with a type prefix.

```
feat(api): add rate limiting middleware
fix(parser): handle empty input without panicking
refactor: extract database connection into shared module
docs: add deployment section to README
test: add integration tests for auth flow
chore: update dependencies
```

This isn't busywork. When you run `git log --oneline` six months later, the type prefix tells you instantly what each commit did without reading the diff. When you use `git bisect` to find a regression, you can skip `docs:` and `chore:` commits mentally and focus on `feat:` and `fix:`.

For solo work, I keep it simple - no scopes unless the project has clear modules. `feat:`, `fix:`, `docs:`, `refactor:`, `test:`, `chore:`. That covers everything.

## Pre-commit hooks that actually help

Pre-commit hooks run automatically before every commit. For Rust projects, two checks catch 90% of issues: formatting and linting.

### The manual approach: a shell script

Create `.git/hooks/pre-commit` and make it executable:

```bash
#!/bin/sh

# Check formatting
echo "Running cargo fmt..."
cargo fmt -- --check
if [ $? -ne 0 ]; then
    echo ""
    echo "ERROR: cargo fmt found formatting issues."
    echo "Run 'cargo fmt' to fix them, then try again."
    exit 1
fi

# Run clippy
echo "Running cargo clippy..."
cargo clippy --all-targets --all-features -- -D warnings
if [ $? -ne 0 ]; then
    echo ""
    echo "ERROR: cargo clippy found issues."
    echo "Fix the warnings above, then try again."
    exit 1
fi
```

```bash
chmod +x .git/hooks/pre-commit
```

The `--check` flag on `cargo fmt` makes it report differences without modifying files. If formatting is off, it exits with a non-zero code and the commit is blocked. You fix the formatting, re-stage, and commit again.

The `-D warnings` flag on clippy promotes all warnings to errors. This is strict, but for solo work where there's no reviewer catching `unwrap()` calls or unused imports, it's worth it.

### The problem with .git/hooks

The `.git/hooks` directory isn't tracked by Git. If you clone the repo on another machine, your hooks are gone. There are several solutions.

**cargo-husky** is the simplest for Rust projects. Add it to your dev dependencies:

```toml
[dev-dependencies.cargo-husky]
version = "1"
default-features = false
features = ["precommit-hook", "run-cargo-fmt", "run-cargo-clippy"]
```

When you run `cargo test`, [cargo-husky](https://github.com/rhysd/cargo-husky) uses Cargo's build script to install the hooks automatically. Anyone who clones the repo and runs tests gets the hooks for free. The downside is that it's opinionated about what the hooks do - the features map to fixed commands.

**A tracked hooks directory** gives you full control:

```bash
mkdir .githooks
```

Move your hook script to `.githooks/pre-commit`, then configure Git to use that directory:

```bash
git config core.hooksPath .githooks
```

Add this to your README or a setup script. The `.githooks` directory is committed, so it travels with the repo. You can put whatever logic you want in the hooks.

**[hk](https://hk.jdx.dev/)** is a newer option written in Rust. It manages hooks through a configuration file, runs linters in parallel with proper file locking (so two linters don't fight over the same file), and automatically stashes unstaged changes before running hooks so linters only see what you're actually committing. That last point matters - without it, your hook might pass because of unstaged fixes in your working tree that aren't part of the commit.

### A note on speed

Running `cargo clippy` on every commit means a compilation check on every commit. For small projects, this takes seconds. For larger projects with many dependencies, the first run after changing code compiles incrementally, which might take 10-30 seconds. Subsequent commits without code changes are nearly instant because the build cache is warm.

If the wait bothers you, move the heavy checks to a `pre-push` hook instead and keep only `cargo fmt -- --check` in `pre-commit`. Formatting checks are instant regardless of project size.

## Rebase vs merge: what actually happens

This is one of those topics where everyone has an opinion but few explain what Git is actually doing. Let's look at the mechanics.

### Merge creates a commit with two parents

When you run `git merge feat/rate-limiter` from `main`, Git creates a new merge commit that has two parents: the tip of `main` and the tip of `feat/rate-limiter`. The history looks like this:

```
main:  A ── B ── C ──────── M
                  \        /
feat:              D ── E ─┘
```

Commit `M` is the merge commit. It records that `D` and `E` were integrated into `main` at this point. The original commits `D` and `E` retain their hashes, timestamps, and messages.

Internally, Git stores each commit as a blob pointing to a tree (the snapshot of all files) plus metadata (author, message, parent hashes). A merge commit is just a commit with two parent hashes instead of one. Git computes the merged tree by finding the common ancestor (`C`), diffing both branches against it, and combining the changes. If both branches modified the same lines, you get a conflict.

### Rebase replays commits on a new base

When you run `git rebase main` from `feat/rate-limiter`, Git takes each commit on your branch (that isn't on `main`), computes the diff that commit introduced, and applies that diff on top of the current tip of `main`. The result is new commits with new hashes:

```
Before rebase:
main:  A ── B ── C
                  \
feat:              D ── E

After rebase:
main:  A ── B ── C
                  \
feat:              D' ── E'
```

`D'` and `E'` have the same diffs as `D` and `E`, but they have different parent hashes (and therefore different commit hashes). They're entirely new objects in Git's object store. The old `D` and `E` still exist in the reflog until garbage collection removes them.

If `main` advanced while you were working on the feature branch:

```
Before:
main:  A ── B ── C ── F ── G
                  \
feat:              D ── E

After rebase:
main:  A ── B ── C ── F ── G
                              \
feat:                          D' ── E'
```

Now your feature branch looks like it was started from the latest `main`. The history is linear - no merge commit needed. When you fast-forward merge (`git merge --ff-only feat/rate-limiter`), `main` simply moves its pointer to `E'`.

### For solo developers: rebase wins

When you're the only one working on the repo, nobody else has checked out your feature branch. The "golden rule" of rebase - never rebase commits that others have based work on - doesn't apply. You can rebase freely.

The linear history you get from rebasing makes every other Git operation better:

- `git log --oneline` reads like a narrative instead of a tangle of merge commits
- `git bisect` walks a straight line instead of exploring both sides of merge commits
- `git blame` shows the actual commits that changed each line, not merge commits

My workflow: rebase the feature branch onto `main` before merging, then fast-forward merge.

```bash
# On feat/rate-limiter
git fetch origin
git rebase origin/main

# If conflicts, resolve them, then:
# git add <resolved files>
# git rebase --continue

# Switch to main and fast-forward
git checkout main
git merge --ff-only feat/rate-limiter
git push origin main
git branch -d feat/rate-limiter
```

### Interactive rebase: cleaning up before merge

Before merging a feature branch, I often clean up the commit history with interactive rebase. This is where `git rebase -i` shines, but since we can't use interactive mode in scripts, here's what it does conceptually.

Say your feature branch has these commits:

```
feat: add rate limiter struct
fix: forgot to export the module
feat: add sliding window algorithm
fix: off-by-one in window calculation
chore: remove debug println
```

That's five commits, but the story is really two: "add rate limiter struct and export" and "add sliding window algorithm." The fix and chore commits are noise - artifacts of development, not meaningful history.

With `git rebase -i main`, you can squash the fixup commits into their parent commits. The result:

```
feat: add rate limiter struct
feat: add sliding window algorithm
```

Two clean commits that each represent a complete, working change. If either introduces a bug, `git bisect` will point at exactly the right commit.

## Squash merging: the simpler alternative

If interactive rebase feels like too much ceremony, squash merging is the pragmatic choice. It combines all commits from a feature branch into a single commit on `main`:

```bash
git checkout main
git merge --squash feat/rate-limiter
git commit -m "feat: add rate limiter with sliding window algorithm"
```

The `--squash` flag stages all the changes from the feature branch but doesn't create a commit. You write a single clean commit message that summarizes the entire feature. The individual development commits (including all the "fix typo" and "wip" noise) disappear from `main`'s history.

GitHub and GitLab both offer "Squash and merge" as a merge option on pull requests - even on solo repos, opening a PR against your own `main` and squash-merging gives you a clean history with links back to the PR for context.

**When to squash vs when to keep commits:** Squash when the feature is a single logical unit. Keep individual commits (after cleaning them up with interactive rebase) when the feature has distinct, independently meaningful steps that someone might want to revert or bisect individually.

## The rescue tools

These three commands have saved me more times than I can count. They're not part of any workflow diagram, but every solo developer should know them cold.

### git stash: context-switching without losing work

You're halfway through implementing a feature when you notice a bug in production. You're not ready to commit - the code doesn't even compile yet. `git stash` saves your working tree changes to a stack and reverts your working directory to the last commit.

```bash
# Save current work
git stash push -m "wip: rate limiter - half done"

# Fix the bug on a new branch
git checkout -b fix/critical-bug
# ... fix, commit, push, merge ...

# Come back and restore your work
git checkout feat/rate-limiter
git stash pop
```

Key things to know:

**Always use messages.** `git stash push -m "description"` saves you from staring at a list of `stash@{0}: WIP on feat/rate-limiter: a1b2c3d` entries trying to remember which one is which.

**Include untracked files.** By default, `git stash` only saves tracked files. New files you created aren't stashed. Use `git stash push -u` (or `--include-untracked`) to grab everything.

**Prefer `apply` over `pop` for risky restores.** `git stash pop` applies the stash and deletes it from the stack if successful. If the apply has conflicts, the stash is kept, but you're now in a messy state. `git stash apply` keeps the stash regardless. Apply, verify everything looks right, then `git stash drop` manually.

```bash
# Safer approach
git stash apply stash@{0}
# Check that everything looks good...
git stash drop stash@{0}
```

**If you have more than 3-4 stashes, use branches instead.** Stashes are unnamed, unordered, and easy to forget. If you're stashing that frequently, create throwaway branches - they show up in `git branch` and can have meaningful names.

### git bisect: binary search for bugs

You know the code worked last Tuesday. It's broken now. Somewhere in the 47 commits between then and now, something broke. You could check each commit manually - or you could let Git binary-search through them.

```bash
git bisect start
git bisect bad                    # current commit is broken
git bisect good a1b2c3d           # this commit from Tuesday was fine
```

Git checks out the commit halfway between good and bad. You test it (run the failing test, reproduce the bug, whatever). Then:

```bash
git bisect good    # this commit is fine, bug is in the later half
# or
git bisect bad     # this commit has the bug, it's in the earlier half
```

Git narrows the range by half each step. For 47 commits, you'll find the offending commit in about 6 steps (`log2(47) ≈ 5.6`) instead of 47.

The real power is automation. If you can express "broken" as a command that returns a non-zero exit code, Git will run the entire bisect for you:

```bash
git bisect start
git bisect bad HEAD
git bisect good a1b2c3d
git bisect run cargo test --test integration_tests
```

Git will check out commits, run your test command, and report the first bad commit. For a Rust project, `cargo test` is perfect for this - if the test suite has a test that catches the regression, bisect finds the culprit automatically.

```
running bisect...
a1b2c3d is the first bad commit
commit a1b2c3d
Author: you
Date:   Thu Apr 23

    refactor: change request parsing to use nom

 src/parser.rs | 47 +++++++++++++++++++++-----------
```

After bisect is done:

```bash
git bisect reset    # go back to where you started
```

This is why clean commit history matters. If every commit compiles and passes tests (because you squashed the "wip" and "fix typo" noise), `git bisect run` works flawlessly. If some commits are half-broken intermediate states, bisect has to skip them and the binary search loses efficiency.

### git reflog: your safety net

The reflog is Git's undo history. Every time HEAD moves - commits, checkouts, rebases, resets, merges - Git records the previous position. It's local only (not pushed to remotes) and entries expire after 90 days by default.

The most common rescue scenario: you ran `git reset --hard` and lost commits.

```bash
# Oh no, I just reset and lost my last 3 commits
git reflog
```

Output:

```
a1b2c3d HEAD@{0}: reset: moving to HEAD~3
f4e5d6c HEAD@{1}: commit: feat: add caching layer
b7a8c9d HEAD@{2}: commit: feat: implement cache invalidation
e0f1a2b HEAD@{3}: commit: test: add cache integration tests
```

Your commits aren't gone - they're just not reachable from any branch. The reflog still points to them:

```bash
# Restore to before the reset
git reset --hard HEAD@{1}
```

Other reflog rescues:

**Recovering from a bad rebase:**
```bash
git reflog
# Find the entry just before "rebase: ..." entries
git reset --hard HEAD@{5}    # back to pre-rebase state
```

**Finding a deleted branch:**
```bash
git reflog | grep "checkout: moving from feat/experiment"
# Find the last commit hash on that branch
git checkout -b feat/experiment-recovered a1b2c3d
```

The reflog is why it's almost impossible to permanently lose committed work in Git. As long as the commit exists in the object store (before garbage collection, which typically runs after 90 days for unreachable objects), the reflog can find it.

One important caveat: the reflog only tracks work you've committed or stashed. If you had changes in your working directory that you never committed and you ran `git checkout -- .` or `git reset --hard`, those changes are gone for real. The lesson: commit early, commit often, clean up later with rebase.

## .gitignore for Rust projects

[GitHub's official Rust.gitignore template](https://github.com/github/gitignore/blob/main/Rust.gitignore) is minimal. Here's what `cargo new` generates plus what you actually need for a real project:

```gitignore
# === Cargo / Rust ===
# Build output directory
/target

# Cargo.lock - commit for binaries, ignore for libraries
# Uncomment the next line if this is a library crate:
# Cargo.lock

# Backup files from rustfmt
**/*.rs.bk

# MSVC Windows debug symbols
*.pdb

# Mutation testing output
**/mutants.out*/

# === Editor / IDE ===
# VS Code - settings are personal, not project config
# (commit .vscode/settings.json if you have shared project settings)
.vscode/
!.vscode/settings.json
!.vscode/extensions.json

# JetBrains (RustRover, CLion, IntelliJ)
.idea/
*.iml

# Vim
*.swp
*.swo
*~

# === OS ===
.DS_Store
Thumbs.db

# === Project-specific ===
# SQLite databases (if applicable)
*.db
*.db-wal
*.db-shm

# Environment files with secrets
.env
.env.local
.env.production

# Coverage reports
/coverage/
lcov.info
tarpaulin-report.html

# Profiling data
perf.data
perf.data.old
flamegraph.svg
```

A few patterns worth explaining:

**`/target` with the leading slash.** Without the slash, Git ignores any directory named `target` anywhere in the repo. The slash restricts it to the root `target/` directory only. This matters if your project has a legitimate subdirectory or file named "target" somewhere. In a Cargo workspace, all member crates share the root `/target` directory, so the leading-slash version works for workspaces too.

**Cargo.lock: commit or ignore?** The [Cargo documentation](https://doc.rust-lang.org/cargo/faq.html#why-have-cargolock-in-version-control) is clear on this: commit `Cargo.lock` for binaries and applications, ignore it for libraries. For binaries, the lock file ensures reproducible builds - everyone (and CI) builds with the exact same dependency versions. For libraries, your downstream users have their own lock file, and committing yours provides no value while causing noise in diffs.

**The `.env` pattern.** Never commit `.env` files. They contain secrets (API keys, database URLs, JWT secrets). Instead, commit a `.env.example` with placeholder values that documents which variables the project needs:

```bash
# .env.example (committed)
DATABASE_URL=sqlite:data/app.db
JWT_SECRET=change-me-in-production
RUST_LOG=info
```

## The complete workflow: start to push

Here's everything stitched together into a daily workflow.

### Project setup (once)

```bash
# Create the project
cargo new my-project
cd my-project

# Set up the gitignore (Cargo already created a basic one)
# Edit .gitignore to add the patterns from the section above

# Set up hooks directory
mkdir .githooks
```

Create `.githooks/pre-commit`:

```bash
#!/bin/sh
set -e

echo "==> Checking formatting..."
cargo fmt -- --check

echo "==> Running clippy..."
cargo clippy --all-targets --all-features -- -D warnings

echo "==> All checks passed."
```

```bash
chmod +x .githooks/pre-commit
git config core.hooksPath .githooks

# Initial commit
git add .
git commit -m "feat: initial project setup with pre-commit hooks"
git remote add origin git@github.com:you/my-project.git
git push -u origin main
```

### Daily development

```bash
# Start a feature
git checkout main
git pull origin main
git checkout -b feat/user-auth

# Work, compile, test...
cargo test
git add src/auth.rs src/main.rs
git commit -m "feat(auth): add JWT token generation and validation"

# More work...
git add src/middleware.rs
git commit -m "feat(auth): add authentication middleware"

# Done with the feature - rebase onto latest main
git fetch origin
git rebase origin/main

# If no conflicts, merge to main
git checkout main
git merge --ff-only feat/user-auth
git push origin main
git branch -d feat/user-auth
```

### Emergency fix while working on a feature

```bash
# Currently on feat/user-auth with uncommitted changes
git stash push -u -m "wip: auth middleware half-done"

# Fix the bug
git checkout main
git pull origin main
git checkout -b fix/parsing-panic

# Fix, test, commit
cargo test
git add src/parser.rs
git commit -m "fix(parser): handle empty input without panicking"

# Merge the fix
git checkout main
git merge --ff-only fix/parsing-panic
git push origin main
git branch -d fix/parsing-panic

# Resume feature work
git checkout feat/user-auth
git stash pop
# Rebase on updated main to include the fix
git rebase origin/main
```

### Finding a regression

```bash
# Something broke. Tests passed yesterday.
git log --oneline --since="2 days ago"
# Identify the last known-good commit

git bisect start
git bisect bad HEAD
git bisect good abc1234
git bisect run cargo test --test regression_test
# Git finds the culprit
git bisect reset
```

### Recovering from mistakes

```bash
# Accidentally reset too far
git reflog
git reset --hard HEAD@{2}

# Accidentally deleted a branch
git reflog | grep "feat/experiment"
git checkout -b feat/experiment-recovered abc1234

# Committed to the wrong branch
git log --oneline -1    # note the commit hash
git reset --soft HEAD~1  # undo commit, keep changes staged
git stash
git checkout correct-branch
git stash pop
git commit -m "feat: the thing I meant to commit here"
```

## What this workflow skips (and why that's fine)

**GitFlow's develop branch.** You don't need an integration branch when there's one developer. `main` is your integration branch.

**Release branches.** Tag `main` when you release. `git tag -a v1.2.0 -m "Release 1.2.0"` is sufficient. If you need changelogs generated from tags, [git-cliff](https://git-cliff.org/) reads your conventional commit messages and generates them automatically - I covered the setup in the [conventional commits post](/blog/conventional-commits-why-and-how-to-write-meaningful-commit-messages/).

**Pull request reviews.** Some solo developers still open PRs against their own repo for the CI checks and the diff view. That's fine - it's optional. The pre-commit hooks and `git diff main...` give you similar coverage without the ceremony.

**Signed commits.** Worth considering if you publish open source, but not essential for private projects. `git config commit.gpgsign true` with a GPG or SSH key if you want it.

The simplicity is the point. One long-lived branch, short-lived feature branches, rebasing for clean history, pre-commit hooks to catch mistakes early, and three rescue commands (`stash`, `bisect`, `reflog`) when things go sideways. Everything else is ceremony that solo developers pay for but rarely benefit from.

Keep `main` deployable. Keep your history linear. Let the tools catch what you miss.
