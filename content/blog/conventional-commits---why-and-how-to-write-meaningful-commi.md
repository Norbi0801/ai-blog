+++
title = "Conventional commits - why and how to write meaningful commit messages"
date = 2026-04-06
description = "A practical guide to the Conventional Commits spec, the tooling that builds on it, and how structured commit messages unlock automated changelogs, semantic releases, and a git history you can actually read."

[taxonomies]
tags = ["git", "devops", "tools", "developer-experience"]
+++

Open any long-lived project you haven't touched in six months. Run `git log --oneline`. What do you see?

```
a1b2c3d fixed stuff
e4f5g6h updates
h7i8j9k wip
l0m1n2o more fixes
p3q4r5s asdf
```

Congratulations, your git history is a write-only data structure. Nobody - including future you - will ever extract useful information from it.

Now compare that with a project that uses [Conventional Commits](https://www.conventionalcommits.org/en/v1.0.0/):

```
a1b2c3d feat(auth): add OAuth2 PKCE flow for mobile clients
e4f5g6h fix(parser): handle escaped quotes in string literals
h7i8j9k docs: update API reference for v3 endpoints
l0m1n2o refactor(db): extract connection pooling into shared module
p3q4r5s feat!: replace session tokens with JWT
```

Every line tells you *what changed*, *where it changed*, and *why it matters*. The exclamation mark on the last one screams "breaking change" before you even read the description. That's the point of Conventional Commits - turning your commit history from noise into signal.

<!-- more -->

## The spec in five minutes

Conventional Commits v1.0.0 defines a lightweight structure on top of your commit messages:

```
<type>[optional scope]: <description>

[optional body]

[optional footer(s)]
```

That's it. Three sections, only the first line is mandatory. Let's break each piece down.

### Type

The type tells you the *category* of change. The spec itself only mandates two: `feat` (new functionality) and `fix` (bug patch). Everything else is a convention your team agrees on. The most common set, inherited from the [Angular commit convention](https://github.com/angular/angular/blob/main/CONTRIBUTING.md#-commit-message-format), looks like this:

| Type | What it means |
|---|---|
| `feat` | A new feature visible to users |
| `fix` | A bug fix |
| `docs` | Documentation only |
| `style` | Formatting, semicolons, whitespace - no logic change |
| `refactor` | Code change that neither fixes a bug nor adds a feature |
| `perf` | Performance improvement |
| `test` | Adding or correcting tests |
| `build` | Changes to the build system or dependencies |
| `ci` | CI configuration files and scripts |
| `chore` | Maintenance tasks that don't modify src or test files |
| `revert` | Reverts a previous commit |

A common mistake is overthinking the type. If you added a new endpoint, it's `feat`. If you fixed a null pointer, it's `fix`. If you moved code around without changing behavior, it's `refactor`. Don't spend five minutes deciding between `chore` and `build` - pick one and move on. Consistency within your project matters more than picking the "perfect" type.

### Scope

The scope is optional but powerful. It narrows down *where* the change lives:

```
feat(auth): add refresh token rotation
fix(parser): handle UTF-8 BOM in input files
docs(api): add rate limiting section
refactor(db): split migrations into versioned modules
```

Scopes should map to something meaningful in your project - a module, a service, a feature area. For a Rust project, you might use crate names or module paths. For a monorepo, package names. The key rule: keep scopes stable. If you rename `auth` to `authentication` halfway through, your changelog grouping breaks.

### Description

The first line after the type and scope. Write it in imperative mood ("add feature" not "added feature"), keep it under 72 characters, no period at the end. This matches git's own conventions for merge commits and tags.

Bad:
```
feat(auth): Added the new OAuth flow for handling mobile client authentication.
```

Good:
```
feat(auth): add OAuth2 PKCE flow for mobile clients
```

### Body

One blank line after the description, then as much detail as you need. This is where you explain *why* the change was necessary, not *what* the diff shows (the diff already shows that):

```
feat(cache): switch from LRU to LFU eviction policy

The LRU cache was thrashing under our read-heavy workload because
frequently accessed keys were getting evicted by one-time bulk reads.
LFU tracks access frequency, keeping hot keys alive regardless of
recent bulk operations.

Benchmarks show 40% fewer cache misses under production traffic
patterns. See internal doc #1234 for the full analysis.
```

### Footer

Footers carry metadata. The most important one is `BREAKING CHANGE`:

```
feat(api)!: return structured error responses

BREAKING CHANGE: Error responses now return JSON objects with `code`
and `message` fields instead of plain text strings. All API clients
need to update their error handling.

Reviewed-by: Alice
Refs: #456
```

Notice the `!` after the scope - that's a shorthand for flagging breaking changes right in the first line. You can use either the `!` or the `BREAKING CHANGE:` footer, or both. The `!` makes it visible in `git log --oneline`, the footer provides space for a detailed explanation.

Other common footers include `Refs:` for issue references, `Reviewed-by:`, and `Co-authored-by:`.

## Why bother? The real payoff

Structured commits are mildly useful for readability alone. They become *extremely* useful when you plug them into automated tooling. Here's the chain reaction:

### 1. Automatic changelog generation

If every commit declares its type and scope, a tool can group them automatically:

```markdown
## [1.5.0] - 2026-04-09

### Features
- **auth**: add OAuth2 PKCE flow for mobile clients (a1b2c3d)
- **cache**: switch from LRU to LFU eviction policy (f6g7h8i)

### Bug Fixes
- **parser**: handle escaped quotes in string literals (e4f5g6h)

### Breaking Changes
- **api**: return structured error responses (j9k0l1m)
```

No more maintaining a changelog by hand. No more "wait, what did we ship in this release?" debates.

### 2. Semantic versioning on autopilot

The Conventional Commits spec was designed to map directly onto [Semantic Versioning](https://semver.org/):

| Commit type | SemVer bump |
|---|---|
| `fix` | PATCH (1.0.0 -> 1.0.1) |
| `feat` | MINOR (1.0.0 -> 1.1.0) |
| `BREAKING CHANGE` / `!` | MAJOR (1.0.0 -> 2.0.0) |

Everything else (`docs`, `refactor`, `test`, `chore`, etc.) typically triggers no version bump at all - they don't affect the public API.

This mapping means your CI pipeline can compute the next version number from the commits since the last release tag. No human decision needed. No "should this be 2.0 or 1.5?" discussions.

### 3. Searchable history

Need to find all breaking changes in the last year?

```bash
git log --oneline --grep="^feat\!:\|^fix\!:\|BREAKING CHANGE" --since="1 year ago"
```

All auth-related changes?

```bash
git log --oneline --grep="(auth)"
```

All performance improvements?

```bash
git log --oneline --grep="^perf"
```

Try doing that with "fixed stuff" and "updates" commits. Good luck.

### 4. Better code review

When every commit has a clear type, reviewers can calibrate their attention. A `refactor` commit should have no behavior change - so the reviewer focuses on structural quality, not correctness. A `feat` commit introduces new behavior - so the reviewer looks for edge cases, tests, documentation. A `fix` commit should reference the bug and include a regression test. The type sets expectations before the reviewer reads a single line of code.

## The tooling ecosystem

The specification alone is just a convention. The real power comes from the tools that parse and enforce it.

### commitlint

[commitlint](https://commitlint.js.org/) is the most popular commit message linter in the JavaScript ecosystem, pulling around 4.8 million weekly downloads on npm for its `@commitlint/config-conventional` package. It validates commit messages against the Conventional Commits spec (or a custom config) and rejects anything that doesn't match.

Basic setup with [Husky](https://typicode.github.io/husky/) for git hooks:

```bash
npm install --save-dev @commitlint/cli @commitlint/config-conventional husky

# Enable git hooks
npx husky init

# Add the commit-msg hook
echo 'npx --no -- commitlint --edit $1' > .husky/commit-msg
```

Create a `commitlint.config.js`:

```js
export default {
  extends: ['@commitlint/config-conventional'],
  rules: {
    // Enforce lowercase types
    'type-case': [2, 'always', 'lower-case'],
    // Limit header length
    'header-max-length': [2, 'always', 100],
    // Only allow these types
    'type-enum': [2, 'always', [
      'feat', 'fix', 'docs', 'style', 'refactor',
      'perf', 'test', 'build', 'ci', 'chore', 'revert'
    ]],
  },
};
```

Now every `git commit` runs through the linter. Try committing "fixed stuff" and you'll get a clear error explaining what's wrong. This is the single most effective way to enforce the convention - make the wrong thing impossible rather than relying on code review.

### semantic-release

[semantic-release](https://github.com/semantic-release/semantic-release) is the full automation pipeline. It analyzes your commits since the last release, determines the version bump, generates release notes, publishes the package, and creates a GitHub release. All from CI, zero manual steps.

A minimal `.releaserc.json`:

```json
{
  "branches": ["main"],
  "plugins": [
    "@semantic-release/commit-analyzer",
    "@semantic-release/release-notes-generator",
    "@semantic-release/changelog",
    "@semantic-release/npm",
    "@semantic-release/github"
  ]
}
```

The `commit-analyzer` plugin reads your conventional commits and decides: did we get any `feat`? That's a MINOR bump. Only `fix` and `perf`? PATCH. Any `BREAKING CHANGE`? MAJOR. The `release-notes-generator` turns those same commits into formatted release notes. The rest handles publishing.

For a GitHub Actions workflow:

```yaml
name: Release
on:
  push:
    branches: [main]

jobs:
  release:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
      - uses: actions/setup-node@v4
        with:
          node-version: 22
      - run: npm ci
      - run: npx semantic-release
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          NPM_TOKEN: ${{ secrets.NPM_TOKEN }}
```

Every push to main triggers a release cycle. If the commits warrant a version bump, it happens automatically. If they don't (say, only `docs` and `chore` commits), nothing happens. No wasted releases.

### git-cliff - for Rust developers

If you're working in the Rust ecosystem, [git-cliff](https://git-cliff.org/) is worth knowing. Written in Rust by [Orhun Parmaksiz](https://github.com/orhun), it generates changelogs from your git history with native Conventional Commits support - and it's fast. We're talking 120ms for 10,000 commits.

```bash
cargo install git-cliff
```

Initialize a config:

```bash
git-cliff --init
```

This creates a `cliff.toml` with customizable templates. Generate a changelog:

```bash
# Full changelog
git-cliff -o CHANGELOG.md

# Only unreleased changes
git-cliff --unreleased

# Changes between tags
git-cliff v1.0.0..v2.0.0
```

The output groups commits by type, includes scope information, and links to commits. You can customize the template to match your project's style - it uses [Tera](https://keats.github.io/tera/) templates, so you have full control over the output format.

A `cliff.toml` snippet showing grouping:

```toml
[changelog]
header = "# Changelog\n"
body = """
{% for group, commits in commits | group_by(attribute="group") %}
### {{ group | upper_first }}
{% for commit in commits %}
  - {% if commit.scope %}**{{ commit.scope }}**: {% endif %}{{ commit.message | upper_first }}\
{% endfor %}
{% endfor %}
"""

[git]
conventional_commits = true
commit_parsers = [
  { message = "^feat", group = "Features" },
  { message = "^fix", group = "Bug Fixes" },
  { message = "^doc", group = "Documentation" },
  { message = "^perf", group = "Performance" },
  { message = "^refactor", group = "Refactoring" },
  { message = "^style", group = "Style" },
  { message = "^test", group = "Testing" },
  { message = "^chore", group = "Miscellaneous" },
]
```

If you want the full Conventional Commits lifecycle in Rust - not just changelogs but also version bumping and commit validation - check out [Cocogitto](https://docs.cocogitto.io/). Its `cog` CLI replaces `git commit` with subcommands that produce spec-compliant messages (`cog feat "add login"`, `cog fix "null pointer in parser"`), verifies your entire commit history with `cog check`, and handles version bumps with `cog bump`.

```bash
cargo install cocogitto

# Create a conventional commit
cog feat "add user registration" --scope auth

# Check all commits comply
cog check

# Bump version based on commit history
cog bump --auto
```

## Conventional Commits vs. other conventions

Conventional Commits isn't the only game in town. Here's how it stacks up against the alternatives.

### Angular commit convention

Conventional Commits grew out of the [Angular commit message format](https://github.com/angular/angular/blob/main/CONTRIBUTING.md#-commit-message-format). The Angular format is nearly identical in structure - same `type(scope): description` pattern, same body and footer sections. The main difference: Angular's spec is tightly coupled to Angular's development process (it includes Angular-specific types and rules about what goes in `CHANGELOG.md`), while Conventional Commits is framework-agnostic. If you follow Conventional Commits, you're following a superset that works everywhere, not just in Angular projects.

### Gitmoji

[Gitmoji](https://gitmoji.dev/) replaces text types with emojis: `:sparkles:` for features, `:bug:` for fixes, `:memo:` for docs.

```
:sparkles: add user registration
:bug: fix null pointer in parser
:memo: update API docs
:recycle: extract shared validation logic
```

The appeal is visual - scanning a log full of emojis is quick if you've memorized the mapping. The downsides are real though:

- **Terminal support varies.** Not every terminal renders emojis correctly. Some show empty boxes or question marks. SSH sessions, older macOS Terminal versions, and certain CI log viewers all have issues.
- **Tooling gap.** The ecosystem of tools that parse Gitmoji commits is much smaller than Conventional Commits. You won't find mature changelog generators or release automation that natively understand `:sparkles:`.
- **Memorization cost.** There are 70+ gitmojis. Compare that with the 11 conventional types most teams use.
- **Machine parsing.** Regex for `^feat:` is trivial. Regex for `:sparkles:` or the actual unicode sparkle character (or both) is fragile.

Some teams combine both: `feat: :sparkles: add login`. That gives you machine-readable types and visual flair. Personally, I'd pick one and stick with it. The hybrid approach doubles the convention burden without doubling the value.

### Freeform with keywords

Some projects use a looser convention: "always start with Fix, Add, Update, Remove, or Refactor." No formal spec, no scopes, no footers.

```
Add user registration
Fix null pointer in parser
Update API docs
```

This is better than nothing but has no tooling support. You can't reliably auto-generate changelogs or compute version bumps because there's no formal grammar to parse. It also drifts quickly - someone writes "Added" instead of "Add", someone else writes "Fixes" instead of "Fix", and now your `grep` patterns break.

### The verdict

If you want tooling support, go Conventional Commits. The ecosystem is enormous, the spec is stable (v1.0.0, unchanged since formalization), and every major CI platform has plugins for it. Gitmoji is fine for small teams who value aesthetics over automation. Freeform keywords are what you do when you know you should have a convention but haven't picked one yet.

## How to actually start

You've read the spec, you know the tools exist. Here's how to go from "fixed stuff" to `fix(parser): handle escaped quotes` without making it a months-long process.

### Step 1: Just start typing the prefix

The simplest possible change. Next commit, instead of:

```
git commit -m "fixed the login bug"
```

Write:

```
git commit -m "fix(auth): prevent session fixation on token refresh"
```

That's it. No tooling, no config, no meetings about naming conventions. Just a prefix. You'll internalize it within a week.

### Step 2: Add a commit-msg hook

Once you're comfortable with the format, enforce it. A basic shell hook in `.git/hooks/commit-msg` (no npm required):

```bash
#!/usr/bin/env bash

commit_msg=$(cat "$1")
pattern="^(feat|fix|docs|style|refactor|perf|test|build|ci|chore|revert)(\(.+\))?(!)?: .+"

if ! echo "$commit_msg" | grep -qE "$pattern"; then
    echo "ERROR: Commit message does not follow Conventional Commits format."
    echo ""
    echo "Expected: <type>[optional scope]: <description>"
    echo "Example:  feat(auth): add OAuth2 PKCE flow"
    echo ""
    echo "Valid types: feat, fix, docs, style, refactor, perf, test, build, ci, chore, revert"
    exit 1
fi
```

Make it executable with `chmod +x .git/hooks/commit-msg`. This catches mistakes before they hit the remote. The downside is `.git/hooks/` isn't committed to the repo - that's where Husky or [pre-commit](https://pre-commit.com/) comes in to share hooks across the team.

### Step 3: Set up shared hooks

For team-wide enforcement, use a git hooks manager. With [Husky](https://typicode.github.io/husky/) (Node.js projects):

```bash
npx husky init
npm install --save-dev @commitlint/cli @commitlint/config-conventional
echo 'npx --no -- commitlint --edit $1' > .husky/commit-msg
```

For Rust projects, you can use Cocogitto's git hook:

```bash
cog install-hook --all
```

Or the framework-agnostic [pre-commit](https://pre-commit.com/) with a `.pre-commit-config.yaml`:

```yaml
repos:
  - repo: https://github.com/compilerla/conventional-pre-commit
    rev: v3.6.0
    hooks:
      - id: conventional-pre-commit
        stages: [commit-msg]
```

### Step 4: Add changelog generation to CI

Pick a tool based on your ecosystem:

- **Rust**: `git-cliff` or `cocogitto`
- **Node.js**: `semantic-release` or `conventional-changelog-cli`
- **Language-agnostic**: `git-cliff` (it's a standalone binary, works anywhere)

Add it to your release workflow. For GitHub Actions with git-cliff:

```yaml
- name: Generate changelog
  uses: orhun/git-cliff-action@v4
  with:
    config: cliff.toml
    args: --latest --strip header
  env:
    OUTPUT: CHANGELOG.md
```

### Step 5: Consider CI-level enforcement

Git hooks run locally and can be bypassed (`--no-verify`). For teams where compliance matters, add a CI check:

```yaml
name: Lint Commits
on:
  pull_request:
    branches: [main]

jobs:
  commitlint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
      - uses: wagoid/commitlint-github-action@v6
```

This checks every commit in a PR against the Conventional Commits spec. If any commit fails, the PR can't merge. It's the backstop for when someone forgets to install hooks or bypasses them.

## Common mistakes and how to avoid them

After using this convention across multiple projects, a few recurring patterns stand out.

**Squashing away useful history.** If you squash-merge PRs, make sure the squash commit message follows the convention. GitHub lets you set a default squash message format - use it. Otherwise your beautiful conventional commits inside the PR become a single "Merge pull request #123" message.

**Scope creep in commits.** One commit should do one thing. If you find yourself writing `feat(auth,parser,db): restructure everything`, that's three separate changes crammed into one commit. Split them. Each commit should be independently revertable and understandable.

**Using `chore` as a catch-all.** "I don't know what type this is, so... `chore`." Be more specific. Updating dependencies? That's `build`. Tweaking CI config? That's `ci`. Reformatting code? That's `style`. Reserve `chore` for things that genuinely don't fit anywhere else - like updating `.gitignore` or reorganizing the project structure.

**Writing the `what` instead of the `why` in the body.** The diff already shows what changed. The body should explain *why* you made the change, what alternative approaches you considered, and what impact this has. A body that says "Changed line 42 from X to Y" is wasting space.

**Inconsistent scopes.** One dev writes `fix(auth)`, another writes `fix(authentication)`, a third writes `fix(login)`. Now your changelog has three groups for the same area. Document your scopes in a `CONTRIBUTING.md` or commit convention doc. Even a simple list is enough:

```markdown
## Commit Scopes
- `auth` - Authentication and authorization
- `api` - REST API endpoints
- `db` - Database layer and migrations
- `ui` - Frontend components
- `ci` - CI/CD pipelines
```

## The cost-benefit analysis

Let's be honest about the tradeoffs.

**Costs:**
- Slightly more effort per commit (maybe 10 extra seconds to type `feat(auth):` instead of just typing a message)
- Initial setup time for tooling (30 minutes to an hour)
- Team alignment (one discussion about types and scopes)
- Occasionally you'll debate whether something is `refactor` or `fix` (set a timer, pick one, move on)

**Benefits:**
- Automated changelogs that actually reflect what shipped
- Semantic versioning without human guesswork
- Git history you can search, filter, and reason about
- Better code reviews through type-level expectations
- Onboarding: new team members can scan the log and understand the project's evolution

The cost is paid once per commit. The benefits compound over the entire lifetime of the project. For any project that lives longer than a few weeks, it's a clear win.

## Getting your team on board

The hardest part isn't the convention itself - it's adoption. Some practical tips:

1. **Don't introduce it with a 50-slide presentation.** Send a one-page doc with 5 examples and a link to the spec.
2. **Set up the tooling first.** If the commit hook rejects bad messages with a helpful error, people learn the format naturally.
3. **Be lenient at first.** Don't reject PRs over `fix` vs `refactor` debates in the first month. Focus on getting the prefix habit established.
4. **Show the output.** The first time you run `git-cliff` and generate a changelog from the team's commits, people see the value immediately.
5. **Lead by example.** If your commits follow the convention and everyone else's don't, that inconsistency becomes visible - and peer pressure does the rest.

Start with the prefix. Add a hook when it feels natural. Wire up changelog generation when you need it. Each step is independently useful - you don't have to buy the whole package on day one.

The best commit convention is the one your team actually follows. Conventional Commits works because the barrier to entry is low (just a prefix), the spec is small enough to fit on a single page, and the tooling payoff is real. Stop writing "fixed stuff." Future you will be grateful.
