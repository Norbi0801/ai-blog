+++
title = "MCP Audit - building and publishing a security scanner in Rust"
date = 2026-04-01
description = "How I built mcp-audit, an open-source security scanner for MCP server configurations, and published it on crates.io."

[taxonomies]
tags = ["rust", "mcp", "security", "open-source"]
+++

I've been working with MCP (Model Context Protocol) servers a lot recently - building tools, integrating them with Claude, configuring them for different projects. At some point I realized how easy it is to misconfigure these things. One wrong path and an AI agent has access to your entire filesystem. One hardcoded API key in your config and it's exposed to anyone who sees your dotfiles.

So I built [mcp-audit](https://github.com/Norbi0801/mcp-audit) - a CLI tool that scans MCP configurations for security issues.

<!-- more -->

## What it does

You run it against your `claude_desktop_config.json` or `.mcp.json`:

```bash
cargo install mcp-audit
mcp-audit scan
```

It checks for 19 different vulnerability patterns - hardcoded API keys (`sk-`, `ghp_`, `AKIA`), shell injection vectors, excessive filesystem permissions, prompt injection in tool descriptions, unpinned dependencies, missing auth, and more.

I scanned the 50 most popular public MCP servers on GitHub. 30% had high or critical issues. 22% had hardcoded credentials in their example configs. Pretty much every single one was missing timeout and response size limits.

## Stuff that went wrong

**Naming.** I originally called it `mcp-scanner`. Built the whole thing, went to publish on crates.io, and the name was already taken. 49 downloads, barely maintained, but the name was gone. Had to rename everything to `mcp-audit`. Check crates.io *before* you start building.

**OWASP licensing.** I wanted to base the rules on the OWASP MCP Top 10. Turns out it's licensed CC BY-NC-SA 4.0 - the NonCommercial part means you can't copy their text into anything commercial. The security concepts themselves aren't copyrightable though, so I wrote my own descriptions. You can say "checks for risks identified in the OWASP MCP Top 10" but you can't say "OWASP certified" or copy their rule text.

**JSON output bug.** When using `--format json`, the tool was printing valid JSON and then appending a plain text summary after it. Broke every script that tried to parse the output. Simple fix - only show the summary for table format - but I only caught it when I tried piping the output to `jq`.

**CI/CD.** The GitHub Action (`action.yml`) needs to handle a lot of edge cases. What if there's no pre-built binary for this platform? Fall back to `cargo install`. What if there are no config files to scan? Exit cleanly. What if SARIF upload requires permissions the workflow doesn't have? Skip it. Each edge case was a separate debugging session.

## How to publish on crates.io

If you haven't done it before, it's simpler than you'd think:

Your `Cargo.toml` needs these fields:

```toml
[package]
name = "mcp-audit"
version = "0.1.0"
license = "AGPL-3.0-or-later"
description = "Security scanner for MCP server configurations"
repository = "https://github.com/Norbi0801/mcp-audit"
keywords = ["mcp", "security", "scanner", "audit"]
categories = ["command-line-utilities", "development-tools"]
```

Then:

```bash
# Get your token from crates.io/settings/tokens
cargo login

# Test first
cargo publish --dry-run

# Ship it
cargo publish
```

After that, `cargo install mcp-audit` works for anyone. The README renders automatically on crates.io so make sure it looks good.

One thing to know - you can't unpublish a version. You can yank it (which hides it from new installs) but it stays in the registry forever. So do the dry-run first.

## What's next

v0.1 does static analysis of config files. Planning to add runtime scanning (actually connecting to servers and probing them), custom rule definitions, and a VS Code extension.

The project is on [GitHub](https://github.com/Norbi0801/mcp-audit) and [crates.io](https://crates.io/crates/mcp-audit). Try it on your configs.
