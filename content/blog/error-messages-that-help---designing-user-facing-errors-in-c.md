+++
title = "Error messages that help - designing user-facing errors in CLI tools"
date = 2025-09-05
description = "Good error messages tell users what went wrong, why, and how to fix it - here's how to build them in Rust with miette, anyhow, colored output, and proper exit codes."

[taxonomies]
tags = ["rust", "cli", "error-handling", "developer-experience"]
+++

Run a CLI tool. It prints `Error: file not found`. That's it. No path. No suggestion. You stare at the terminal and start guessing.

Now imagine this instead:

```
error: config.toml not found in /home/user/myproject

  help: Run `mytool init` to generate a default config file.
        Or specify a path with --config <path>
```

Same error. Completely different experience. The second one tells you what happened, where it happened, and what to do about it. The first one wastes your time.

This isn't about aesthetics. It's about whether your tool gets adopted or abandoned. Every bad error message is a support ticket waiting to happen, a GitHub issue someone will open, or - most likely - a user who silently switches to a different tool.

<!-- more -->

## The anatomy of a good error message

Every useful error message answers three questions:

1. **What went wrong?** - the immediate problem, stated clearly
2. **Why did it go wrong?** - enough context to understand the cause
3. **How do I fix it?** - a concrete next step

Most tools nail the first one and skip the other two. Let's look at real examples from tools that get this right.

**rustc** is the gold standard. When you write `let x: u32 = "hello";`, you don't get `type mismatch`. You get the exact location, the expected type, the found type, and often a suggestion for how to fix it. Rust's compiler team explicitly drew inspiration from [Elm's error message philosophy](https://elm-lang.org/news/compiler-errors-for-humans) - the idea that compilers (and by extension, all dev tools) should be assistants, not adversaries.

**clap** handles argument errors well. Pass an invalid flag and you see:

```
error: unexpected argument '--colour' found

  tip: a similar argument exists: '--color'

Usage: mytool [OPTIONS] <input>

For more information, try '--help'.
```

It didn't just reject the input. It guessed what you meant, showed the correct flag, and reminded you how to get help.

**git** often gets criticism for bad errors, but its recent versions improved significantly. `git switch nonexistent` now suggests similar branch names instead of just failing.

The pattern is consistent: good tools treat errors as a conversation, not a wall.

## The Rust error crate ecosystem

Rust has a layered approach to error handling. If you've been following along, I covered trait bounds and derive macros in earlier posts - that background helps here since the error ecosystem is built on top of both.

### thiserror - structured error types

[thiserror](https://crates.io/crates/thiserror) (v2.0.18) generates `std::error::Error` implementations via derive. You define an enum of error variants, each with a format string, and thiserror writes the `Display` and `Error` impls for you.

```rust
use thiserror::Error;

#[derive(Error, Debug)]
pub enum ConfigError {
    #[error("config file `{path}` not found")]
    NotFound {
        path: String,
        #[source]
        cause: std::io::Error,
    },

    #[error("failed to parse config at line {line}")]
    ParseError {
        line: usize,
        #[source]
        cause: toml::de::Error,
    },

    #[error("missing required field `{field}` in config")]
    MissingField { field: String },
}
```

The `#[source]` attribute chains errors together - `Error::source()` returns the underlying cause. The `#[from]` attribute generates a `From<T>` impl for automatic conversion with `?`. This is your library-facing error type: callers can `match` on variants and handle each case differently.

### anyhow - context chaining for applications

[anyhow](https://crates.io/crates/anyhow) (v1.0.102) takes the opposite approach. Instead of typed enums, it wraps errors in a type-erased `anyhow::Error` and lets you chain context messages on top.

```rust
use anyhow::{Context, Result};
use std::path::Path;

fn load_config(path: &Path) -> Result<Config> {
    let content = std::fs::read_to_string(path)
        .with_context(|| format!("failed to read config from {}", path.display()))?;

    let config: Config = toml::from_str(&content)
        .context("failed to parse config file")?;

    validate(&config)
        .context("config validation failed")?;

    Ok(config)
}
```

When the innermost error fires, you get a chain:

```
Error: config validation failed

Caused by:
    0: failed to parse config file
    1: expected `=`, found newline at line 7 column 1
```

Each `.context()` call adds a layer. The user sees the high-level problem first, then can drill down into the root cause. This is the pattern you want in application code - binary crates, CLI tools, servers.

**When to use which:** thiserror for libraries (callers need to match on error variants), anyhow for applications (you want to add context and print a useful message). They compose well together - define errors with thiserror, propagate them with anyhow's `.context()`.

## miette - rich diagnostics for humans

[miette](https://crates.io/crates/miette) (v7.6.0) is where error reporting gets serious. It extends `std::error::Error` with structured metadata - error codes, help text, source code snippets with labeled spans, severity levels, and related diagnostics. Think rustc-style error output for your own tool.

```rust
use miette::{Diagnostic, NamedSource, SourceSpan};
use thiserror::Error;

#[derive(Error, Debug, Diagnostic)]
#[error("invalid value for field `{field}`")]
#[diagnostic(
    code(config::invalid_value),
    help("expected {expected}, got `{actual}`. Check the docs at https://mytool.dev/config")
)]
pub struct InvalidConfigValue {
    field: String,
    expected: String,
    actual: String,
    #[source_code]
    src: NamedSource<String>,
    #[label("this value")]
    span: SourceSpan,
}
```

With the `fancy` feature enabled, this renders as:

```
  × invalid value for field `timeout`
   ╭─[config.toml:3:1]
 2 │ name = "myapp"
 3 │ timeout = "not_a_number"
   ·           ──────────────── this value
 4 │ retries = 3
   ╰────
  help: expected integer (seconds), got `not_a_number`. Check the docs
        at https://mytool.dev/config
```

That's a massive improvement over `Error: invalid config`. The user sees exactly where the problem is, what was expected, and where to find more information.

### Building diagnostics at runtime

You don't always have source code to point at. For file-system errors, network failures, or validation issues, miette's `miette!` macro and `MietteDiagnostic` builder work without spans:

```rust
use miette::{miette, Result, LabeledSpan};

fn resolve_path(input: &str) -> Result<PathBuf> {
    let path = PathBuf::from(input);
    if !path.exists() {
        return Err(miette!(
            help = "check that the path is correct, or use an absolute path",
            code = "io::not_found",
            "path `{}` does not exist",
            path.display()
        ));
    }
    Ok(path)
}
```

### Report handlers

miette ships three report handlers, and you can pick the right one for the context:

- `GraphicalReportHandler` - the fancy Unicode box-drawing output shown above. This is the default when `fancy` is enabled.
- `NarratableReportHandler` - plain text, screen-reader friendly. No box-drawing characters, no colors.
- `JSONReportHandler` - machine-readable output for piping into other tools or log aggregation.

You can swap the handler at startup:

```rust
miette::set_hook(Box::new(|_| {
    if std::env::var("MYTOOL_JSON_ERRORS").is_ok() {
        Box::new(miette::JSONReportHandler::new())
    } else {
        Box::new(miette::GraphicalReportHandler::new())
    }
}))
.expect("failed to set miette hook");
```

This is a real pattern. CI environments and structured logging pipelines want JSON. Humans want the graphical output. Let the environment decide.

## Color and formatting

Color isn't decoration - it's a signaling system. When a user scans terminal output, color instantly tells them what needs attention.

The convention most tools follow:

- **Red** + bold = error (something failed, action needed)
- **Yellow** = warning (something might be wrong, but execution continues)
- **Cyan** or blue = help/hint (suggestion for the user)
- **Green** = success (operation completed)
- **Dim/gray** = metadata (timestamps, paths, secondary info)

### owo-colors

[owo-colors](https://crates.io/crates/owo-colors) (v4.2.3) is zero-allocation and `no_std`-compatible. It implements coloring through the `OwoColorize` trait, which you can call on any type that implements `Display`:

```rust
use owo_colors::OwoColorize;

fn print_error(msg: &str, help: Option<&str>) {
    eprintln!("{}{} {}", "error".red().bold(), ":".bold(), msg);
    if let Some(h) = help {
        eprintln!("  {} {}", "help:".cyan().bold(), h);
    }
}
```

The zero-allocation part matters. `"error".red()` doesn't create a new `String` - it returns a wrapper that writes ANSI escape codes during `Display::fmt`. This is different from the [colored](https://crates.io/crates/colored) crate (v3.1.1) which allocates a `ColoredString`. For hot paths or libraries, owo-colors is the better choice. For simple CLIs, colored's API is slightly more ergonomic - pick whichever fits.

### Respecting NO_COLOR

The [NO_COLOR](https://no-color.org/) convention is simple: if the `NO_COLOR` environment variable is set (to any value), don't output ANSI color codes. This matters for piping, CI logs, accessibility, and users who just prefer plain text.

owo-colors supports this via its `supports-colors` feature and the `if_supports_color` method:

```rust
use owo_colors::{OwoColorize, Stream};

eprintln!(
    "{} {}",
    "error:".if_supports_color(Stream::Stderr, |t| t.red().bold()),
    msg
);
```

colored respects `NO_COLOR`, `CLICOLOR`, and `CLICOLOR_FORCE` out of the box. If you're using miette with the `fancy` feature, it handles this automatically.

Don't be the tool that dumps raw ANSI escape codes into a log file. Check the stream.

## Exit codes that mean something

Every process returns an exit code. Most Rust CLIs return 0 for success and 1 for everything else. That's a missed opportunity.

### The sysexits convention

BSD's `sysexits.h` (adopted in POSIX) defines a set of exit codes that other tools can act on:

| Code | Name | Meaning |
|------|------|---------|
| 0 | `EX_OK` | Success |
| 1 | - | Generic failure |
| 2 | `EX_USAGE` | Invalid arguments (clap uses this) |
| 64 | `EX_USAGE` | Command line usage error |
| 65 | `EX_DATAERR` | Input data format error |
| 66 | `EX_NOINPUT` | Input file not found/unreadable |
| 73 | `EX_CANTCREAT` | Can't create output file |
| 74 | `EX_IOERR` | I/O error |
| 78 | `EX_CONFIG` | Configuration error |

Why does this matter? Because scripts and CI pipelines can branch on exit codes. If your linter returns 65 for "found lint errors" vs 78 for "config file is broken," a CI pipeline can distinguish between "code needs fixing" and "the CI config itself is wrong."

### Exit codes in Rust

Since Rust 1.61, `std::process::ExitCode` implements the `Termination` trait, so you can return it directly from `main()`:

```rust
use std::process::ExitCode;

fn main() -> ExitCode {
    match run() {
        Ok(()) => ExitCode::SUCCESS,
        Err(e) => {
            print_error(&e);
            match e.classify() {
                ErrorClass::Usage => ExitCode::from(2),
                ErrorClass::Config => ExitCode::from(78),
                ErrorClass::Io => ExitCode::from(74),
                ErrorClass::Internal => ExitCode::from(70),
            }
        }
    }
}
```

The [sysexits](https://crates.io/crates/sysexits) crate (v0.13.0) wraps these codes in a proper enum that also implements `Termination`:

```rust
use sysexits::ExitCode;

fn main() -> ExitCode {
    match run() {
        Ok(()) => ExitCode::Ok,
        Err(ref e) if e.kind() == io::ErrorKind::NotFound => ExitCode::NoInput,
        Err(ref e) if e.kind() == io::ErrorKind::PermissionDenied => ExitCode::NoPerm,
        Err(_) => ExitCode::Software,
    }
}
```

One thing to watch: Rust's default panic handler exits with code 101. That's fine - it signals an unexpected crash, distinct from any of your application-level codes. If you use [human-panic](https://crates.io/crates/human-panic) (v2.0.7), it replaces the raw backtrace with a user-friendly message and a crash report file path, while keeping the non-zero exit code.

## Putting it all together

Here's how I structure error handling in a CLI tool, combining everything discussed:

```rust
use std::process::ExitCode;
use miette::{Diagnostic, Result as MietteResult};
use thiserror::Error;
use owo_colors::OwoColorize;

#[derive(Error, Debug, Diagnostic)]
pub enum AppError {
    #[error("config file `{path}` not found")]
    #[diagnostic(
        code(app::config::not_found),
        help("run `{bin} init` to create a default config, or pass --config <path>")
    )]
    ConfigNotFound { path: String, bin: String },

    #[error("invalid config: {reason}")]
    #[diagnostic(code(app::config::invalid), help("{suggestion}"))]
    ConfigInvalid {
        reason: String,
        suggestion: String,
        #[source_code]
        src: miette::NamedSource<String>,
        #[label("here")]
        span: miette::SourceSpan,
    },

    #[error("network request to `{url}` failed")]
    #[diagnostic(
        code(app::network::failed),
        help("check your internet connection and try again")
    )]
    NetworkError {
        url: String,
        #[source]
        cause: reqwest::Error,
    },
}

impl AppError {
    fn exit_code(&self) -> ExitCode {
        match self {
            AppError::ConfigNotFound { .. } => ExitCode::from(66),
            AppError::ConfigInvalid { .. } => ExitCode::from(78),
            AppError::NetworkError { .. } => ExitCode::from(69),
        }
    }
}

fn main() -> ExitCode {
    // Install miette's fancy handler for graphical output.
    // In CI (when NO_COLOR is set), miette automatically
    // falls back to plain text.
    miette::set_hook(Box::new(|_| {
        Box::new(miette::GraphicalReportHandler::new())
    }))
    .ok();

    match run() {
        Ok(()) => ExitCode::SUCCESS,
        Err(e) => {
            let code = e.exit_code();
            // miette handles the formatting:
            // source spans, help text, error codes - all included
            eprintln!("{:?}", miette::Report::new(e));
            code
        }
    }
}
```

The layers work together:

1. **thiserror** defines the error variants with structured fields
2. **miette's Diagnostic** derive adds help text, error codes, and source spans
3. **owo-colors** or miette's built-in formatter handles coloring
4. **Exit codes** map each error class to a meaningful code
5. **NO_COLOR** and output format are handled automatically

## Common mistakes

**Swallowing context.** The worst pattern is `.map_err(|_| "something went wrong")`. You just threw away the actual error. Always chain with `.context()` or `.with_context()` - the original error is preserved in the "Caused by" chain.

**Mixing stderr and stdout.** Errors go to `stderr`. Always. If your tool's output can be piped (and it should be), error messages in stdout corrupt the pipeline. Use `eprintln!`, never `println!` for errors.

**Forgetting the machine audience.** Humans read your error messages. Scripts read your exit codes. JSON consumers parse your structured output. A tool that only serves one audience fails the others. Consider `--format json` for error output in automation-heavy tools.

**Leaking internal paths.** `Error: /home/ci-runner/.cache/mytool/v2/tmp/proc_7841.lock: permission denied` tells the user nothing actionable and leaks your internal directory structure. Show the logical path (`cache lock file`), not the physical one, unless `--verbose` is set.

**Not testing error paths.** If you wrote a nice error message, write a test that triggers it. miette's `NarratableReportHandler` produces stable plain-text output that's easy to snapshot-test:

```rust
#[test]
fn test_config_not_found_message() {
    let err = AppError::ConfigNotFound {
        path: "config.toml".into(),
        bin: "mytool".into(),
    };
    let report = miette::Report::new(err);
    let mut output = String::new();
    miette::NarratableReportHandler::new()
        .render_report(&mut output, report.as_ref())
        .unwrap();

    assert!(output.contains("config.toml"));
    assert!(output.contains("mytool init"));
}
```

## The effort is worth it

Writing good error messages takes time. You need to think about what the user was trying to do when things broke, not just what the code was doing. You need to imagine someone reading your output at 2am during an incident and ask: does this help them, or does it waste their time?

The Rust ecosystem gives you the tools. thiserror and anyhow handle the plumbing. miette handles the presentation. owo-colors handles the formatting. sysexits handles the exit codes. The hard part isn't the code - it's the empathy. Sit on the other side of the terminal and think about what you'd want to see.

The tools that win are the ones that respect their users' time. A good error message is the cheapest feature you can build that directly reduces support burden and increases adoption.
