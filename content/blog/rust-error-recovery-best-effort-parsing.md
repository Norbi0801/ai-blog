+++
title = "Rust error recovery - partial results and best-effort parsing"
date = 2026-01-26
description = "Stop bailing on the first error. How to collect diagnostics, recover at synchronization points, and emit partial results in Rust the way rustc, clippy, and ruff do."
[taxonomies]
tags = ["rust", "parsing", "error-handling", "compilers"]
+++

Run `cargo check` on a broken file. You don't get one error, you get all of them. Twelve type mismatches, three missing imports, a borrow checker complaint - reported in a single pass. Now imagine if rustc bailed on the first one. You'd fix it, recompile, fix the next one, recompile, twelve cycles later you're done. That's not how good tools work.

Most error-handling advice in Rust centers on `?` and the fail-fast pattern. That's correct for 90% of code. The other 10% - parsers, linters, validators, batch processors - need the opposite. Errors are data. You collect them, keep going, and report everything at the end.

If you're not familiar with the basics of `thiserror`, `anyhow`, and miette, I covered them in [Error messages that help - designing user-facing errors in CLI tools](/blog/error-messages-that-help-designing-user-facing-errors-in-cli-tools/). This post picks up where that one stopped: when one error isn't enough.

<!-- more -->

## The fail-fast trap

The `?` operator is seductive because it makes happy-path code clean:

```rust
fn parse_config(path: &Path) -> Result<Config, ConfigError> {
    let raw = std::fs::read_to_string(path)?;
    let parsed: Config = toml::from_str(&raw)?;
    validate(&parsed)?;
    Ok(parsed)
}
```

This is fine for IO failures (you can't continue after the file isn't there). It's wrong for validation. If the user's config has three errors, you tell them about one, they fix it, you tell them about the second, repeat. Each round-trip costs minutes.

Real-world tools that get this right:

- **rustc** keeps parsing after a syntax error using panic-mode recovery, then runs the full type-checker over the salvaged AST so the user sees the full damage at once. Source: [rustc_parse/src/parser/diagnostics.rs](https://github.com/rust-lang/rust/blob/master/compiler/rustc_parse/src/parser/diagnostics.rs).
- **clippy** runs every lint over every file regardless of warnings already emitted. A single run gives you the full picture.
- **ruff** (Astral's Python linter, written in Rust, v0.13.x) collects diagnostics into a `Vec<Diagnostic>` per file and only reports them after the full check completes. The codebase is a great reference: [astral-sh/ruff](https://github.com/astral-sh/ruff).
- **biome** (formerly Rome, JS toolchain in Rust) uses the same model with their `Diagnostic` trait and a sink that accumulates results across files in parallel.

The pattern is identical: parse what you can, mark the broken parts, keep moving, report everything at the end. That's error recovery.

## The accumulator pattern

The simplest form is a `Vec<Error>` you push into instead of returning early:

```rust
use thiserror::Error;

#[derive(Error, Debug)]
pub enum ValidationError {
    #[error("field `{field}` is required")]
    Missing { field: String },

    #[error("field `{field}` must be at least {min}, got {actual}")]
    OutOfRange { field: String, min: i64, actual: i64 },

    #[error("field `{field}` must match `{pattern}`")]
    BadPattern { field: String, pattern: String },
}

pub struct Validated<T> {
    pub value: T,
    pub errors: Vec<ValidationError>,
}

pub fn validate(cfg: &RawConfig) -> Validated<Config> {
    let mut errors = Vec::new();

    let port = cfg.port.unwrap_or_else(|| {
        errors.push(ValidationError::Missing { field: "port".into() });
        0
    });

    if port != 0 && !(1..=65535).contains(&port) {
        errors.push(ValidationError::OutOfRange {
            field: "port".into(),
            min: 1,
            actual: port as i64,
        });
    }

    let name = cfg.name.clone().unwrap_or_else(|| {
        errors.push(ValidationError::Missing { field: "name".into() });
        String::new()
    });

    if !name.is_empty() && !name.chars().all(|c| c.is_ascii_alphanumeric() || c == '-') {
        errors.push(ValidationError::BadPattern {
            field: "name".into(),
            pattern: "[a-zA-Z0-9-]+".into(),
        });
    }

    Validated {
        value: Config { port, name },
        errors,
    }
}
```

The caller decides what to do:

```rust
let result = validate(&raw);
for err in &result.errors {
    eprintln!("{err}");
}
if !result.errors.is_empty() {
    std::process::exit(65);
}
```

You report every problem the user has, not just the one that happened to come first.

## Result<(T, Vec<Warning>)> for partial success

Sometimes errors split into two flavors: fatal (can't produce output) and non-fatal (output is degraded but usable). Compilers call these errors and warnings. The classic Rust signature for this is:

```rust
pub fn process<T>(input: &str) -> Result<(T, Vec<Warning>), Vec<Error>>;
```

`Ok((value, warnings))` means we got something usable, but pay attention to the warnings. `Err(errors)` means we couldn't recover. The `Vec` on the error side is intentional - even fatal failures usually come in batches.

A concrete example: parsing a CSV file where empty rows are skippable but malformed cells fail.

```rust
#[derive(Debug)]
pub struct Warning {
    pub line: usize,
    pub message: String,
}

#[derive(Debug, Error)]
#[error("line {line}: {message}")]
pub struct CsvError {
    pub line: usize,
    pub message: String,
}

pub fn parse_csv(src: &str) -> Result<(Vec<Row>, Vec<Warning>), Vec<CsvError>> {
    let mut rows = Vec::new();
    let mut warnings = Vec::new();
    let mut errors = Vec::new();

    for (i, line) in src.lines().enumerate() {
        let line_no = i + 1;
        let trimmed = line.trim();

        if trimmed.is_empty() {
            warnings.push(Warning {
                line: line_no,
                message: "empty line skipped".into(),
            });
            continue;
        }

        match parse_row(trimmed) {
            Ok(row) => rows.push(row),
            Err(msg) => errors.push(CsvError { line: line_no, message: msg }),
        }
    }

    if errors.is_empty() {
        Ok((rows, warnings))
    } else {
        Err(errors)
    }
}
```

Two things make this pattern work:

1. **Warnings travel with the value.** They're not logged from inside the function, they're returned. The caller decides whether to print them, suppress them under `-q`, or convert them to errors under `-Werror`.
2. **Errors are batched.** A 10,000-row CSV with two bad rows reports two errors, not one then exit.

Astral's ruff uses a fancier version of this. Their `check_path` returns `(Vec<Diagnostic>, ...)` and a `Diagnostic` carries its own severity level (Error, Warning, Info). One container holds everything.

## Multiple diagnostics with miette

miette (v7.6.0) supports related diagnostics natively. The `#[related]` attribute marks a field as a sequence of further diagnostics that should be rendered as part of the same report:

```rust
use miette::{Diagnostic, NamedSource, SourceSpan};
use thiserror::Error;

#[derive(Error, Debug, Diagnostic)]
#[error("config has {} error(s)", self.problems.len())]
pub struct ConfigReport {
    #[source_code]
    pub src: NamedSource<String>,
    #[related]
    pub problems: Vec<ConfigProblem>,
}

#[derive(Error, Debug, Diagnostic)]
pub enum ConfigProblem {
    #[error("unknown key `{key}`")]
    #[diagnostic(code(config::unknown_key), help("did you mean `{suggestion}`?"))]
    UnknownKey {
        key: String,
        suggestion: String,
        #[label("here")]
        span: SourceSpan,
    },

    #[error("value for `{key}` must be a positive integer")]
    #[diagnostic(code(config::bad_int))]
    BadInteger {
        key: String,
        #[label("got `{actual}`")]
        span: SourceSpan,
        actual: String,
    },
}
```

When you print a `ConfigReport`, miette renders the parent error and every `ConfigProblem` underneath it, each with its own labeled span pointing at the original source. The output looks like a multi-error rustc report:

```
  × config has 2 error(s)
  ├─▶ unknown key `tiomeout`
  │      ╭─[config.toml:3:1]
  │    3 │ tiomeout = 30
  │      · ────┬───
  │      ·     ╰── here
  │      ╰────
  │     help: did you mean `timeout`?
  │
  ╰─▶ value for `port` must be a positive integer
         ╭─[config.toml:5:8]
       5 │ port = "eighty"
         ·        ────┬────
         ·            ╰── got `"eighty"`
         ╰────
```

This is the same shape you'd see from the Rust compiler, generated by your code, in maybe 40 lines.

A practical wrinkle: `#[related]` accepts `Vec<T>` where `T: Diagnostic`. If your individual diagnostics are heterogeneous (different enums, different severities), you can box them - `Vec<Box<dyn Diagnostic + Send + Sync + 'static>>` works because miette's `Diagnostic` trait is dyn-safe.

## Parser recovery - synchronization points

For accumulating validation errors, you keep walking the input. Parsers are harder. Once a parser hits unexpected input, it's at an unknown state. Continuing naively gives you cascading errors - one missing semicolon causes 30 fake errors below it.

The classic technique is **panic-mode recovery**. When the parser detects an error, it:

1. Records the error.
2. Skips tokens until it finds a known *synchronization point* - usually a token that starts the next top-level construct.
3. Resumes parsing from there.

For a C-like grammar, sync points are usually `;`, `}`, or the start of a top-level keyword (`fn`, `struct`, `let`). Crafting Interpreters has the canonical [walkthrough](https://craftinginterpreters.com/parsing-expressions.html#synchronizing-a-recursive-descent-parser) and rustc's parser does the same thing under [`Parser::recover_*`](https://github.com/rust-lang/rust/blob/master/compiler/rustc_parse/src/parser/diagnostics.rs).

A miniature version, parsing a sequence of `let name = expr;` statements:

```rust
#[derive(Debug)]
pub enum Token {
    Let,
    Ident(String),
    Eq,
    Number(i64),
    Semi,
    Eof,
    Unknown(char),
}

#[derive(Debug)]
pub struct Stmt {
    pub name: String,
    pub value: i64,
}

pub struct Parser<'a> {
    tokens: &'a [Token],
    pos: usize,
    pub errors: Vec<String>,
}

impl<'a> Parser<'a> {
    fn peek(&self) -> &Token {
        self.tokens.get(self.pos).unwrap_or(&Token::Eof)
    }

    fn bump(&mut self) -> &Token {
        let t = &self.tokens[self.pos.min(self.tokens.len() - 1)];
        self.pos += 1;
        t
    }

    fn expect_ident(&mut self) -> Option<String> {
        match self.bump() {
            Token::Ident(s) => Some(s.clone()),
            other => {
                self.errors.push(format!("expected identifier, got {other:?}"));
                None
            }
        }
    }

    /// Skip tokens until we see a `;` or hit EOF. Resume from there.
    fn synchronize(&mut self) {
        while !matches!(self.peek(), Token::Semi | Token::Eof) {
            self.pos += 1;
        }
        if matches!(self.peek(), Token::Semi) {
            self.pos += 1;
        }
    }

    pub fn parse_program(&mut self) -> Vec<Stmt> {
        let mut out = Vec::new();
        while !matches!(self.peek(), Token::Eof) {
            match self.parse_stmt() {
                Some(s) => out.push(s),
                None => self.synchronize(),
            }
        }
        out
    }

    fn parse_stmt(&mut self) -> Option<Stmt> {
        if !matches!(self.bump(), Token::Let) {
            self.errors.push("expected `let`".into());
            return None;
        }
        let name = self.expect_ident()?;
        if !matches!(self.bump(), Token::Eq) {
            self.errors.push(format!("expected `=` after `{name}`"));
            return None;
        }
        let value = match self.bump() {
            Token::Number(n) => *n,
            other => {
                self.errors.push(format!("expected number, got {other:?}"));
                return None;
            }
        };
        if !matches!(self.bump(), Token::Semi) {
            self.errors.push(format!("expected `;` after value of `{name}`"));
            return None;
        }
        Some(Stmt { name, value })
    }
}
```

The whole strategy lives in two places: `parse_stmt` returns `Option` instead of bailing the whole parser, and `synchronize` walks forward to a known good point when something failed. The result is that `let a = ;\nlet b = 2;` produces one error (about `a`), not a cascade. `b` parses cleanly because we resynced at the semicolon.

Real parsers have richer sync sets. rustc tracks which tokens "could begin an expression" or "could begin an item" and uses those as sync targets depending on what context broke. The principle is the same.

A more advanced refinement is **error productions**: rules that match common mistakes deliberately, so you produce a high-quality error instead of a generic "unexpected token". rustc has dozens of these - things like "saw `=>` after `if` condition, you probably meant `{`". They're worth their weight in user goodwill.

## Severity and the linter use case

Linters are the cleanest example of best-effort processing because their *entire output* is errors. Each finding has a severity and a category, and you want them all in one batch.

A reusable shape:

```rust
use miette::{Diagnostic, SourceSpan};
use thiserror::Error;

#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum Severity {
    Error,
    Warning,
    Info,
}

#[derive(Error, Debug, Diagnostic, Clone)]
#[error("{message}")]
pub struct Finding {
    pub severity: Severity,
    pub rule: &'static str,
    pub message: String,
    #[label("{rule}")]
    pub span: SourceSpan,
}

pub struct LintResult {
    pub findings: Vec<Finding>,
}

impl LintResult {
    pub fn has_errors(&self) -> bool {
        self.findings.iter().any(|f| f.severity == Severity::Error)
    }

    pub fn exit_code(&self) -> i32 {
        if self.has_errors() { 65 } else { 0 }
    }
}
```

Now individual lint passes are independent functions that push into `findings`. They can run in any order, or in parallel using rayon, because they never short-circuit each other. That's exactly how clippy and ruff structure their internals.

The `-Werror` knob (treat warnings as errors) becomes a one-line transform on the result rather than a global flag threaded through every parser:

```rust
if cfg.warnings_as_errors {
    for f in &mut result.findings {
        if f.severity == Severity::Warning {
            f.severity = Severity::Error;
        }
    }
}
```

Bundling severity with the finding instead of branching on it keeps the producer code completely independent of caller policy. That separation of concerns is the whole point.

## When NOT to recover

Recovery isn't always right. Three cases where fail-fast wins:

**Security boundaries.** If a JWT signature doesn't verify, you don't keep parsing the payload "just in case the rest is valid". You drop it. Same for SQL, command parsing, anything where partially trusted input becomes a vulnerability. Recovery widens the attack surface because broken inputs reach more code paths.

**Transactional operations.** A migration that hits an error halfway through shouldn't keep applying remaining steps. Roll back. Recovery semantics get muddled when state is in flight.

**Cheap retries.** If retrying the operation costs nothing (single network call, idempotent), giving the user one error and a clean retry beats giving them a partial result they don't know how to act on.

The mental model: recovery is for human-driven workflows where the cost of a re-run is the human's time. Use it where the user benefits from seeing the full picture. Don't use it where partial state creates more confusion than it resolves.

## The pattern in summary

You have three knobs to turn when designing error flow:

1. **Accumulator vs. fail-fast.** Walk the whole input and collect, or stop on first error. Validators and linters should accumulate; security checks should fail fast.
2. **Partial output via `Result<(T, Vec<Warning>), Vec<Error>>`.** Expose warnings alongside the value when you can produce one. Errors batch the same way.
3. **Synchronization points for parsers.** Don't let one syntax error cascade. Define what "the start of the next thing" means for your grammar and resync there.

miette gives you the rendering for free once you've decided on shape. thiserror gives you the structured types. The harder design work is upstream: deciding what counts as recoverable, where to resync, how to label severities. Get those right and the rest is plumbing.

When your parser, validator, or linter reports six problems in one run instead of forcing six round-trips, users notice. They might not be able to articulate why your tool feels better than the one that bails on first error. They just keep using yours.
