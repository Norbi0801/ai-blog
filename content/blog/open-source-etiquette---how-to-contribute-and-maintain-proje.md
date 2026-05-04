+++
title = "Open source etiquette - how to contribute and maintain projects"
date = 2025-11-30
description = "A practical guide to contributing to open source without wasting maintainers' time, maintaining projects without burning out, and the invisible infrastructure that holds it all together."

[taxonomies]
tags = ["open-source", "git", "devops", "developer-experience"]
+++

Open source runs on goodwill. There's no contract, no SLA, no customer support. Someone wrote code, published it, and now millions of builds depend on it. According to GitHub's [Octoverse 2025](https://octoverse.github.com/) report, the platform hosts 395 million public repositories with 1.12 billion contributions across them. 518.7 million pull requests were merged in a single year. A new developer joins GitHub every second.

Behind those numbers are maintainers. Real people reviewing PRs at 11 PM, triaging issues on weekends, answering the same question for the fifteenth time. The [2024 Tidelift maintainer survey](https://blog.tidelift.com/maintainer-burnout-is-real) found that 60% of open source maintainers are unpaid, and 44% cite burnout as their reason for walking away. Some don't just walk away - they burn the bridge on the way out.

This post is about both sides of the table. How to contribute without being the person maintainers dread, and how to maintain a project without destroying yourself in the process.

<!-- more -->

## The contributor side

### Before you touch code: read the room

Every project has norms. They might be written down, or they might be implicit in the commit history and PR reviews. Your first job as a contributor is to figure out what they are.

Start here:

1. **README.md** - Does the project accept contributions? Some explicitly say they don't. Respect that.
2. **CONTRIBUTING.md** - If it exists, read all of it. Not just the "how to submit a PR" section. The setup instructions, the testing requirements, the coding style, the branch naming convention. This document is the maintainer telling you exactly what they need. Ignoring it is like showing up to a job interview and asking "so what does the company do?"
3. **Open issues and PRs** - Browse 10-15 of the most recent ones. How do people communicate? What gets merged? What gets rejected? What feedback do reviewers give? This tells you more about the project's culture than any document.
4. **CI configuration** - Look at the GitHub Actions workflows, the `.cargo/config.toml`, the `rustfmt.toml`, the `clippy.toml`. These are automated style guides. If the project runs `cargo fmt --check` in CI, your PR needs to pass `cargo fmt` before you submit it.

I've seen contributors open a PR, get a "please run `cargo fmt`" comment, and never come back. That's wasted time for everyone. Five minutes of reading the CI config would have prevented it.

### Finding something to work on

The worst way to start contributing is to clone a repo, skim the code for ten minutes, and open a PR that refactors something you think looks messy. Maintainers don't need surprise refactors. They need help with the work they've already identified.

**The "good first issue" label.** GitHub research shows that [45% of issues labeled "good first issue"](https://dl.acm.org/doi/10.1145/3510003.3510196) are solved by true newcomers. These are issues the maintainer has specifically scoped and tagged as approachable. They're your entry point.

But there's a catch: only about 0.1% of GitHub projects actively label issues this way. If the project you care about doesn't have labeled issues, look for:

- Bug reports with clear reproduction steps but no fix yet
- Documentation gaps (missing examples, outdated instructions)
- Failing tests on specific platforms
- Typos and dead links (yes, really - it's a legitimate first contribution that gets you familiar with the contribution workflow)

**Ask before building.** If you want to add a feature or make a significant change, open an issue first. Describe what you want to do and why. Wait for a response. Maintainers know things you don't - maybe the feature was already attempted and abandoned, maybe it conflicts with the roadmap, maybe there's a reason the code looks the way it does. Building something for a week and then having the PR rejected because it doesn't align with the project's direction is painful for both sides.

### Writing a PR that gets reviewed

Maintainers are volunteers with limited time. The easier you make their job, the faster your PR gets attention.

**Keep it focused.** One PR, one change. Don't fix a bug, refactor the module, update a dependency, and add a feature all in one PR. Each of those is a separate review with separate risk. Mixed PRs take longer to review, are harder to bisect if something goes wrong, and often get stuck because one part is ready but another isn't.

**Write a useful description.** The PR description should answer:

- What does this change?
- Why is this change needed? (link the issue)
- How did you test it?

A real example of a good PR description:

```markdown
## What

Fixes the panic in `parse_header` when the input contains
non-UTF-8 bytes. Returns `Err(ParseError::InvalidEncoding)`
instead of unwrapping.

Closes #247

## How

Replaced `str::from_utf8().unwrap()` with a match on the
Result, mapping the error to our existing ParseError type.

## Testing

- Added test case with binary input (`tests/parser.rs`)
- Verified existing tests still pass
- Tested manually with the malformed file from issue #247
```

Compare that to "fixed bug" with no description. Which one would you rather review?

**Match the existing style.** If the project uses `snake_case` for module names, don't introduce `kebab-case`. If functions return `Result<T, AppError>`, don't return `anyhow::Result`. If tests are in a `tests/` directory, don't put inline `#[cfg(test)]` modules. If you've read my post on [conventional commits](/blog/conventional-commits-why-and-how-to-write-meaningful-commit-messages), apply that format only if the project already uses it. Imposing your preferred workflow on someone else's project is not a contribution.

**Write tests.** If you're fixing a bug, write a test that fails without your fix and passes with it. If you're adding a feature, cover the happy path and at least one edge case. A PR with tests tells the maintainer "I've thought about whether this actually works." A PR without tests tells them "I hope it works, good luck."

Here's a pattern that works well for bug fix PRs in Rust:

```rust
#[test]
fn parse_header_with_invalid_utf8_returns_error() {
    // This is the exact input from issue #247
    let input = b"\xff\xfe\x00\x01";
    let result = parse_header(input);
    assert!(result.is_err());
    assert!(matches!(
        result.unwrap_err(),
        ParseError::InvalidEncoding { .. }
    ));
}
```

The test name describes the scenario. The input is documented. The assertion is specific. A maintainer can read this and understand the fix without looking at the diff.

### Responding to feedback

Your PR will get feedback. Maybe the maintainer asks you to rename a variable, restructure a function, add a test case, or rewrite the approach entirely. This is normal. It's not personal. It's the review process working as designed.

What not to do:

- Argue about style preferences that the project has already settled
- Ghost after receiving feedback (if you can't finish the PR, say so - someone else can pick it up)
- Push back on every comment (choose your battles - if it's a genuine design disagreement, explain your reasoning once and accept the maintainer's decision)

What to do:

- Respond to each comment, even if it's just "done" or "good point, fixed"
- Push updates as new commits (don't force-push during review unless asked - reviewers lose context when the history changes under them)
- Thank the reviewer. They're spending their free time on your code.

## The maintainer side

### Making your project approachable

If you want contributions, you have to make contributing possible. That sounds obvious, but a shocking number of projects have no setup instructions, no test suite, and a README that says "a tool for doing X" with zero examples.

**README structure that works:**

```markdown
# project-name

One sentence: what it does.

## Quick start

How to install/use it in under 60 seconds.

## Example

A complete, working code snippet showing the most
common use case.

## Building from source

Prerequisites, clone, build, run tests.

## Contributing

Link to CONTRIBUTING.md or inline instructions.

## License

MIT / Apache-2.0 / whatever.
```

If you've read my post on [writing documentation that developers actually read](/blog/writing-documentation-that-developers-actually-read), the same principles apply here. Answer "what is this," "how do I use it," and "show me" - in that order.

**CONTRIBUTING.md that saves time:**

A good CONTRIBUTING.md prevents bad PRs. It should cover:

- How to set up the development environment (exact commands, not "install the dependencies")
- How to run the test suite
- Code style expectations (or point to the `rustfmt.toml`)
- The PR review process and expected response time
- What kinds of contributions you're looking for (and what you're not)

That last point is important. If your project isn't accepting new features right now, say so. If you only want bug fixes, say so. It's better to set clear boundaries than to have contributors spend hours on work you'll reject.

**Issue templates.** GitHub supports issue templates that prompt reporters for the right information:

```yaml
# .github/ISSUE_TEMPLATE/bug_report.yml
name: Bug Report
description: Report a bug
body:
  - type: textarea
    id: description
    attributes:
      label: What happened?
      description: A clear description of the bug
    validations:
      required: true
  - type: textarea
    id: reproduction
    attributes:
      label: Steps to reproduce
      description: Minimal steps to reproduce the behavior
    validations:
      required: true
  - type: input
    id: version
    attributes:
      label: Version
      description: "Output of `your-tool --version`"
    validations:
      required: true
  - type: dropdown
    id: os
    attributes:
      label: Operating System
      options:
        - Linux
        - macOS
        - Windows
```

This turns "it doesn't work" issues into actionable bug reports. It saves you from playing 20 questions in the comments.

### CI is your first reviewer

Every PR should be automatically checked before you even look at it. At minimum, your CI pipeline should run:

```yaml
# .github/workflows/ci.yml
name: CI
on: [push, pull_request]

jobs:
  check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: dtolnay/rust-toolchain@stable
        with:
          components: rustfmt, clippy
      - uses: actions/cache@v4
        with:
          path: |
            ~/.cargo/registry
            ~/.cargo/git
            target
          key: ${{ runner.os }}-cargo-${{ hashFiles('**/Cargo.lock') }}
      - run: cargo fmt --check
      - run: cargo clippy -- -D warnings
      - run: cargo test
```

This catches formatting issues, lint warnings, and test failures automatically. When a PR shows up with a green checkmark, you know the basics are handled. When it shows up with a red X, you can point the contributor to the CI log instead of manually explaining what's wrong.

One security note from GitHub's [2026 Actions security roadmap](https://github.blog/news-insights/product-news/whats-coming-to-our-github-actions-2026-security-roadmap/): pin your actions to full commit SHAs in production workflows, not just version tags. Tags are mutable - a compromised action could push malicious code under an existing tag. For open source CI that runs on untrusted PRs from forks, this matters.

### Versioning and changelogs

If you've read my post on [semantic versioning in Rust](/blog/semantic-versioning-what-breaking-changes-actually-means-in-rust), you know how much damage a wrong version bump can cause. For maintainers, the discipline is straightforward:

- Follow [semver 2.0.0](https://semver.org/) strictly
- Document every user-facing change in a CHANGELOG.md
- Use the [Keep a Changelog](https://keepachangelog.com/) format

A CHANGELOG entry should look like this:

```markdown
## [0.5.0] - 2026-05-01

### Added
- `parse_header` now accepts `&[u8]` in addition to `&str`

### Fixed
- Panic on non-UTF-8 input in `parse_header` (#247)

### Changed
- BREAKING: `ParseError` now has an `InvalidEncoding` variant
```

The "BREAKING" prefix in the Changed section is critical. Users scan changelogs for breaking changes before upgrading. Make them impossible to miss.

### Responding to PRs

This is where most maintainers lose contributors. According to research by GitHub, PRs that don't receive a response within 7 days have a dramatically lower chance of ever being completed. Contributors interpret silence as rejection.

You don't need to do a full review immediately. A quick comment goes a long way:

- "Thanks for this! I'll review it this weekend."
- "This is on my radar - I'm finishing a release this week and will look at it after."
- "I see the CI is failing on the formatting check. Could you run `cargo fmt` and push?"

If you can't maintain responsiveness, say so in your README or CONTRIBUTING.md. "I review PRs on weekends" or "expect 1-2 week turnaround" sets the right expectations. No response at all is worse than a slow response.

## Licenses: the legal infrastructure

The license file is the most important file in your repository. It defines what anyone can legally do with your code. No license means no permission - not "do whatever you want" but "you have no rights to use this."

Here's the practical breakdown of the licenses you'll actually encounter:

**MIT** - The most popular license on GitHub (found in [92% of audited open source](https://www.synopsys.com/software-integrity/resources/analyst-reports/open-source-security-risk-analysis.html) according to the 2025 OSSRA report). Do whatever you want, keep the copyright notice. No patent protection.

**Apache 2.0** - Like MIT but with an explicit patent grant. If a contributor owns patents that cover code they contribute, those patents are licensed to all users. This is why corporate-backed projects (Kubernetes, TensorFlow, Rust itself) prefer Apache 2.0 - it protects against patent trolling.

**GPL (v2, v3)** - Copyleft. If you use GPL code in your project, your project must also be GPL. This is why the Linux kernel is GPL - it ensures that all derivatives remain open. The v3 adds patent protection and anti-tivoization clauses.

**Dual licensing (MIT OR Apache-2.0)** - The Rust ecosystem convention. Every official Rust project and most crates use this. It gives users the choice between MIT's simplicity and Apache's patent protection.

If you're starting a new project and don't have strong opinions, `MIT OR Apache-2.0` is the safe default for the Rust ecosystem. Add both license files:

```
LICENSE-MIT
LICENSE-APACHE
```

And in your `Cargo.toml`:

```toml
[package]
license = "MIT OR Apache-2.0"
```

One thing people get wrong: changing a license after the fact requires consent from every contributor. Their contributions were made under the original license terms. If you start with GPL and want to switch to MIT, you need every contributor to agree. Choose wisely up front.

## Code of conduct: why it matters more than you think

"We don't need a code of conduct, we're all adults." I've heard this from maintainers of projects that later had their issue tracker turn into a war zone. A code of conduct isn't about policing - it's about setting expectations before problems arise.

The [Contributor Covenant](https://www.contributor-covenant.org/) is the standard. Version 3.0, released in July 2025, is adopted by 9 of the 10 largest open source projects in the world, including the Linux kernel, Rust, and Django. It covers the basics:

- Be respectful
- No harassment, discrimination, or personal attacks
- How to report violations
- What happens when violations occur

The 3.0 version reframes enforcement as "addressing and repairing harm" - inspired by restorative justice principles rather than punitive ones. It's designed for communities, not courtrooms.

Adding it is simple:

```bash
# Download the latest version
curl -o CODE_OF_CONDUCT.md \
  https://www.contributor-covenant.org/version/3/0/code_of_conduct/code_of_conduct.md
```

Then actually enforce it. A code of conduct that's never referenced when someone is being abusive is worse than not having one - it signals that the rules are decoration.

## Burnout: the real threat to open source

This isn't a soft topic. Maintainer burnout is an existential risk to the software supply chain. The examples are stark:

**xz Utils (2024).** A multi-year social engineering attack that exploited maintainer burnout. The original xz maintainer, overwhelmed and understaffed, gradually ceded co-maintainer access to a bad actor ("Jia Tan") who spent two years building trust before injecting a backdoor into versions 5.6.0 and 5.6.1. It was [discovered by accident](https://en.wikipedia.org/wiki/XZ_Utils_backdoor) when a PostgreSQL developer noticed a performance anomaly. This wasn't a sophisticated zero-day - it was someone patiently exploiting the fact that a critical infrastructure maintainer was drowning. CISA's post-incident analysis called it a [structural sustainability problem](https://www.cisa.gov/news-events/news/lessons-xz-utils-achieving-more-sustainable-open-source-ecosystem).

**colors.js (2022).** The sole maintainer deliberately sabotaged his own library - 3.3 billion lifetime downloads, 19,000 dependents - by introducing an infinite loop. His stated reason: corporations profiting from his unpaid labor.

**event-stream (2018).** A maintainer who had moved on transferred ownership to a volunteer who turned out to be malicious, injecting cryptocurrency-stealing code into a package with millions of downloads.

Three incidents, one pattern: a single overwhelmed person maintaining critical infrastructure with no support, no funding, and no succession plan.

### Protecting yourself as a maintainer

**Set boundaries publicly.** Put your availability in the README. "I work on this project on weekends" or "I have a day job and a family - response times may be slow." People's expectations adjust to the information they have.

**Say no.** Not every feature request is worth implementing. Not every PR is worth merging. "This doesn't align with the project's goals" is a complete sentence. You don't owe anyone a ten-paragraph justification.

**Delegate.** If someone consistently submits good PRs, give them triage permissions. Then review permissions. Building a team of two or three trusted contributors is the difference between a sustainable project and a single point of failure.

**Accept funding.** GitHub Sponsors, Open Collective, Tidelift - if people use your code in production, it's reasonable to accept support for maintaining it. The "I do this for fun, not money" mindset is noble right up until it's 2 AM and you're fixing a CVE in code that runs in half the Fortune 500.

**Archive when done.** If you've lost interest and nobody's stepping up to maintain the project, archive the repository. A clearly archived project is infinitely better than an abandoned one that people keep depending on because it's not marked as unmaintained.

### Being a good consumer of open source

On the flip side, if you're using open source in production:

- **Report bugs with reproduction steps**, not "it doesn't work"
- **Don't demand timelines** on free software. "When will this be fixed?" on an unpaid volunteer's issue tracker is corrosive
- **Sponsor** the maintainers of your critical dependencies. Check `cargo tree` - if a crate is in your dependency chain and you ship production software, the maintainer's work has monetary value to you
- **Contribute back** when you find bugs. A PR with a fix is worth a hundred issues

## Putting it together

Open source works because of social contracts, not legal ones. The license gives you permission to use the code. The code of conduct sets the behavioral baseline. The CONTRIBUTING.md tells you how to participate. The maintainer's time and energy are the scarce resource that makes all of it function.

Whether you're opening your first PR or maintaining a crate with a million downloads, the etiquette is the same: respect the humans on the other side of the screen. Read what they've written. Follow the process they've established. Communicate clearly. And when the balance between giving and receiving tips too far in one direction, have the awareness to correct it.

The best open source contributors I've seen aren't the ones who write the most code. They're the ones who make the maintainer's life easier - by reading the docs before asking, by writing clean PRs with tests and descriptions, by responding to feedback gracefully, and by knowing when to contribute and when to step back.

And the best maintainers aren't the ones who merge the most PRs. They're the ones who set clear expectations, build contributor pipelines, protect their own time, and know when to hand the project off rather than let it rot.

Open source is a commons. Take care of it.
