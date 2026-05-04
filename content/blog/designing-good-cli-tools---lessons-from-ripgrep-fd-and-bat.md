+++
title = "Designing good CLI tools - lessons from ripgrep, fd, and bat"
date = 2025-08-10
description = "What makes some CLI tools feel effortless while others fight you - dissecting the design decisions behind ripgrep, fd, and bat, then building those patterns in Rust."

[taxonomies]
tags = ["rust", "cli", "tools", "developer-experience"]
+++

Run `grep -rn "TODO" .` in a large project. Watch it crawl through `node_modules`, `.git`, and every binary file it encounters. Then run `rg TODO`. Same query, but the results appear before your fingers leave the keys.

The speed difference isn't just about algorithms (though those matter). It's about defaults. ripgrep skips `.gitignore`d files, ignores hidden directories, detects and avoids binary files, and colorizes output - all without a single flag. You didn't have to learn anything. It just worked.

That "it just worked" feeling is what separates great CLI tools from technically correct ones. And it's not accidental. It's designed.

<!-- more -->

## The holy trinity of modern CLI tools

Three tools have essentially redefined what developers expect from a command-line experience: [ripgrep](https://github.com/BurntSushi/ripgrep) (rg), [fd](https://github.com/sharkdp/fd), and [bat](https://github.com/sharkdp/bat). All three are written in Rust. All three are replacements for POSIX tools (grep, find, cat). And all three have accumulated massive adoption - ripgrep sits at ~61k GitHub stars, bat at ~57k, fd at ~42k.

They didn't get there by being slightly faster. They got there by being dramatically more pleasant to use.

Let's break down what they actually do differently and what you can steal for your own CLI tools.

## Lesson 1: Sensible defaults beat flexibility

The classic UNIX philosophy says tools should be simple and composable. That's great in theory, but `find . -type f -name "*.rs" -not -path "*/target/*"` is nobody's idea of simple. The POSIX tools were designed for a world without `.gitignore` files, without color terminals, without projects that have 50,000 files in `node_modules`.

ripgrep's defaults tell you everything about its design philosophy:

- **Recursive search** is on by default (grep needs `-r`)
- **`.gitignore` patterns are respected** (grep searches everything)
- **Hidden files are skipped** (you rarely want to search `.git/`)
- **Binary files are detected and skipped** (NUL byte heuristic on file content)
- **Smart case** - searches are case-insensitive when your pattern is all lowercase, case-sensitive the moment you use an uppercase letter
- **Colorized output** when writing to a terminal

fd does the same thing for file finding. Compare:

```bash
# find: you spell out every exclusion
find . -type f -name "*.rs" -not -path "*/target/*" -not -path "*/.git/*"

# fd: the obvious thing, with smart defaults
fd -e rs
```

fd skips `.gitignore`d files, hidden files, and colorizes output - all by default. It also uses regex patterns by default instead of globs, which is more powerful for ad-hoc searching.

The key principle: **optimize for the 90% case**. The person typing `rg TODO` at 2 AM doesn't want to think about flags. They want results. The 10% who need to search binary files or hidden dirs can pass `--no-ignore` or `-u` (and rg supports up to `-uuu` for progressively disabling each filter layer).

## Lesson 2: Respect the terminal

bat handles one of the most important UX decisions a CLI tool can make: detecting its context.

```bash
# In a terminal: full decorations, syntax highlighting, line numbers, paging
bat src/main.rs

# Piped to another command: plain text, no colors, no decorations
bat src/main.rs | head -20
```

bat uses `std::io::IsTerminal` (stable since Rust 1.70) to check if stdout is connected to a terminal or a pipe:

```rust
use std::io::{self, IsTerminal};

fn should_use_colors() -> bool {
    // Check NO_COLOR env var (https://no-color.org)
    if std::env::var("NO_COLOR").is_ok() {
        return false;
    }
    io::stdout().is_terminal()
}
```

Under the hood, `IsTerminal` calls `isatty()` on Unix and uses Windows-specific console detection on Windows. Before Rust 1.70, everyone used the `atty` crate for this - now it's in the standard library.

When piped, bat becomes a drop-in replacement for `cat`. When interactive, it becomes a syntax-highlighted pager. Same binary, completely different behavior based on context.

ripgrep does this too. Terminal output gets colors and grouped results. Piped output is plain, greppable text. This dual behavior is critical for composability - your tool should play nicely with `| head`, `| wc -l`, `| xargs` without vomiting ANSI escape codes into the pipe.

### The `NO_COLOR` standard

There's a convention at [no-color.org](https://no-color.org/) - if the `NO_COLOR` environment variable is set (to any value), your tool should suppress all color output. All three tools respect this. CI systems, logging pipelines, accessibility tools - there are real reasons people set this. If you're building a CLI tool in Rust, check for it.

## Lesson 3: Progressive disclosure

Good CLI tools don't dump 200 flags in your face. They layer information:

**Layer 1: Zero-config.** The tool works with no flags at all. `rg pattern`, `fd pattern`, `bat file`.

**Layer 2: `-h` (short help).** One-line descriptions of the most common flags. Fits on a screen.

**Layer 3: `--help` (long help).** Detailed descriptions with examples and caveats. ripgrep's `--help` output is a masterclass - flags are grouped by category (search, filter, output, logging), each with a paragraph of explanation.

**Layer 4: Man pages and docs.** The full reference, for when you need the edge cases.

[clap](https://docs.rs/clap/latest/clap/) (the standard Rust argument parser, currently at v4.6) supports this natively with its two-tier help system:

```rust
use clap::Parser;

#[derive(Parser)]
#[command(name = "mytool", about = "Search code fast")]
struct Cli {
    /// Pattern to search for
    pattern: String,

    /// Search hidden files and directories
    #[arg(short = '.', long,
        help = "Search hidden files",
        long_help = "Search hidden files and directories. By default, \
        hidden files are skipped. This is equivalent to providing \
        --no-ignore-hidden."
    )]
    hidden: bool,

    /// Number of threads to use
    #[arg(short = 'j', long, default_value_t = 0,
        help = "Number of threads (0 = auto)",
        long_help = "Number of threads to use for searching. Setting \
        this to zero (the default) causes the tool to choose the \
        thread count based on available CPU cores."
    )]
    threads: usize,
}
```

Running `mytool -h` shows the short `help` text. Running `mytool --help` shows the `long_help` with full context. This is exactly what ripgrep does. Users who know what they're looking for get a quick reference. Users who are exploring get thorough documentation.

## Lesson 4: What happens under the hood

The "it feels fast" quality of these tools isn't magic. Here's what's actually happening.

### ripgrep's parallel walker

ripgrep uses the [`ignore`](https://github.com/BurntSushi/ripgrep/tree/master/crates/ignore) crate (which it maintains) for recursive directory traversal. This crate provides `WalkParallel` - a parallel directory walker built on [crossbeam-deque](https://github.com/crossbeam-rs/crossbeam) work-stealing queues.

The architecture: each thread owns a FIFO/LIFO `Worker` deque. A global `Injector` queue seeds initial work. When a thread runs out of work in its own deque, it steals from other threads via their `Stealer` handles. This means no thread sits idle while others have directory entries queued up.

But the real cleverness is in how `.gitignore` matching works. The `ignore` crate builds a **persistent hierarchical data structure** - each directory has its own ignore matcher that references its parent's matcher. When you enter `src/`, its matcher knows about the root `.gitignore`, the root `.rgignore`, and `src/.gitignore`. This hierarchy is shared across threads without locking because it's immutable and append-only going down the tree.

The filter priority is layered: override patterns (`--glob`) take precedence over the ignore hierarchy (`.gitignore` > `.ignore` > `.rgignore`), which takes precedence over file type filters (`--type`). fd uses this same `ignore` crate, which is why both tools have identical `.gitignore` behavior.

If you want to look at the implementation, the parallel walker lives in [`crates/ignore/src/walk.rs`](https://github.com/BurntSushi/ripgrep/blob/master/crates/ignore/src/walk.rs) and the gitignore parser in [`crates/ignore/src/gitignore.rs`](https://github.com/BurntSushi/ripgrep/blob/master/crates/ignore/src/gitignore.rs).

### ripgrep's regex strategy

ripgrep ships two regex engines:

1. **Rust `regex` crate** (default) - finite automata with SIMD-accelerated literal optimizations. Guaranteed O(n) time complexity - no catastrophic backtracking, ever. The tradeoff: no lookahead, lookbehind, or backreferences.

2. **PCRE2** (via `-P` flag) - full Perl-compatible regex with JIT compilation. Supports look-around and backreferences, but can exhibit O(2^n) worst-case on pathological patterns.

For typical code searches, the Rust engine is faster because it aggressively extracts literal strings from patterns and uses SIMD to scan for them before engaging the full automaton. When you search for `fn\s+main`, it first scans for the literal `fn` at SIMD speeds, then only runs the regex engine on matching lines. Andrew Gallant wrote a [detailed analysis](https://blog.burntsushi.net/ripgrep/) of why this approach wins in practice.

### bat's syntax highlighting pipeline

bat uses [syntect](https://github.com/trishume/syntect), which loads Sublime Text `.sublime-syntax` definitions - giving it support for 170+ languages without maintaining its own grammar files. Since syntect 5.0, syntax definitions are lazy-loaded. bat only parses the grammar for the language it's actually rendering, which made small file display ~75% faster.

bat also sets a smart performance boundary: lines longer than 16,384 characters skip syntax highlighting entirely. This prevents pathological cases (minified JS, huge JSON blobs) from freezing your terminal.

For git integration, bat uses the `git2` crate (Rust bindings to libgit2) to compare the working directory against the git index, showing change markers (`+`, `~`, `-`) in the gutter alongside the syntax-highlighted code. This is conditionally compiled under the `git` feature flag - if you build bat without it, there's no libgit2 dependency at all.

### fd's parallel execution

fd doesn't just find files in parallel - it can execute commands on them in parallel too:

```bash
# Execute a command for each result, in parallel
fd -e rs -x rustfmt {}

# Execute a command once with all results as arguments
fd -e rs -X wc -l
```

The `-x` flag spawns threads (defaulting to available CPU count, capped at 64) that each consume entries from the search results and execute the command. `-X` collects all results and passes them as arguments to a single invocation. The thread cap at 64 prevents fd from overwhelming the system when running on high-core-count machines.

## Lesson 5: Exit codes matter

I covered error messages in detail in [Error messages that help - designing user-facing errors in CLI tools](/blog/error-messages-that-help-designing-user-facing-errors-in-cli-tools/), but exit codes deserve attention because they're the primary way your tool communicates with scripts and other programs.

ripgrep follows the grep convention:
- `0` - at least one match found
- `1` - no matches found (not an error - just no results)
- `2` - an actual error occurred

This three-way distinction is crucial. Scripts depend on it:

```bash
if rg -q "unsafe" src/; then
    echo "Found unsafe code, running extra checks..."
    cargo clippy -- -D warnings
fi
```

If "no results" returned exit code 0, this script wouldn't work. If it returned exit code 1 alongside actual errors, you couldn't distinguish between "the code is clean" and "the tool crashed."

In Rust, control this with `std::process::ExitCode` (stable since Rust 1.61):

```rust
use std::process::ExitCode;

fn main() -> ExitCode {
    let matches = run_search();

    match matches {
        Ok(count) if count > 0 => ExitCode::SUCCESS,
        Ok(_) => ExitCode::from(1),  // no matches found
        Err(e) => {
            eprintln!("codesearch: {e:#}");
            ExitCode::from(2)         // actual error
        }
    }
}
```

## Building these patterns in Rust

Let's put it all together. Here's a skeleton for a CLI tool that follows the patterns from ripgrep, fd, and bat - proper argument parsing, terminal detection, colored output, progress indicators, and correct piping behavior.

### Project setup

```toml
# Cargo.toml
[package]
name = "codesearch"
version = "0.1.0"
edition = "2024"

[dependencies]
clap = { version = "4.6", features = ["derive"] }
owo-colors = { version = "4.2", features = ["supports-colors"] }
indicatif = "0.17"
ignore = "0.4"
anyhow = "1"
```

A note on color crates. I recommend [`owo-colors`](https://github.com/owo-colors/owo-colors) over `colored`. It's zero-allocation (colors are applied through `Display` impl, no intermediate `String` created), supports `no_std`, and handles `NO_COLOR`/`FORCE_COLOR` out of the box. [`termcolor`](https://github.com/BurntSushi/termcolor) (by Andrew Gallant, same author as ripgrep) is another solid choice, especially if you need Windows console API support. `colored` works fine but allocates a new `String` for every colorized fragment.

### Terminal-aware output

```rust
use std::io::{self, IsTerminal, Write, BufWriter};
use owo_colors::OwoColorize;

struct OutputConfig {
    use_colors: bool,
    is_tty: bool,
}

impl OutputConfig {
    fn detect() -> Self {
        let is_tty = io::stdout().is_terminal();
        let use_colors = is_tty && std::env::var("NO_COLOR").is_err();
        Self { use_colors, is_tty }
    }
}

fn print_match(
    writer: &mut impl Write,
    path: &str,
    line_num: usize,
    line: &str,
    matched_range: std::ops::Range<usize>,
    config: &OutputConfig,
) -> io::Result<()> {
    if config.use_colors {
        write!(writer, "{}:", path.purple())?;
        write!(writer, "{}:", line_num.green())?;

        // Highlight only the matched portion
        let before = &line[..matched_range.start];
        let matched = &line[matched_range.clone()];
        let after = &line[matched_range.end..];
        writeln!(writer, "{}{}{}", before, matched.red().bold(), after)?;
    } else {
        // Plain output - standard grep format for piping
        writeln!(writer, "{}:{}:{}", path, line_num, line)?;
    }
    Ok(())
}
```

Notice how the plain output format (`path:line_num:line`) is the standard grep format that other tools expect. When piped, your output should be machine-parseable. When interactive, it should be human-readable. Two different audiences, same data.

Also notice `BufWriter`. Writing to stdout line-by-line without buffering is surprisingly slow. ripgrep uses a similar buffered writer internally - it's one of those details that doesn't show up in benchmarks until you remove it.

### Progress indication for long operations

When your tool processes thousands of files, silence is hostile. The user wonders: is it stuck? Is it working? Should I Ctrl-C?

```rust
use indicatif::{ProgressBar, ProgressStyle};
use std::time::Duration;

fn create_progress(config: &OutputConfig) -> ProgressBar {
    if !config.is_tty {
        // Don't pollute piped output with progress info
        return ProgressBar::hidden();
    }

    let pb = ProgressBar::new_spinner();
    pb.set_style(
        ProgressStyle::default_spinner()
            .template("{spinner:.cyan} {msg} [{elapsed_precise}]")
            .expect("valid template")
    );
    pb.enable_steady_tick(Duration::from_millis(120));
    pb
}
```

`ProgressBar::hidden()` is key here. indicatif has first-class support for suppressing progress bars in non-interactive contexts. You don't need conditional logic scattered through your code - create the progress bar once, and all the `pb.set_message()` and `pb.inc(1)` calls become no-ops when hidden.

### Using the ignore crate for .gitignore-aware walking

The `ignore` crate gives you everything ripgrep and fd use for directory traversal:

```rust
fn search(cli: &Cli, config: &OutputConfig) -> anyhow::Result<u64> {
    let pb = create_progress(config);
    let mut match_count = 0u64;

    let mut builder = ignore::WalkBuilder::new(&cli.path[0]);
    for path in &cli.path[1..] {
        builder.add(path);
    }
    builder
        .hidden(!cli.hidden)         // skip hidden files by default
        .git_ignore(!cli.no_ignore)  // respect .gitignore by default
        .threads(if cli.threads == 0 { 0 } else { cli.threads });

    let stdout = io::stdout();
    let mut writer = BufWriter::new(stdout.lock());

    for entry in builder.build() {
        let entry = entry?;
        if !entry.file_type().map_or(false, |ft| ft.is_file()) {
            continue;
        }

        let path = entry.path();
        pb.set_message(format!("{}", path.display()));

        let content = match std::fs::read_to_string(path) {
            Ok(c) => c,
            Err(_) => continue, // skip binary/unreadable files
        };

        for (line_num, line) in content.lines().enumerate() {
            if line.contains(&cli.pattern) {
                print_match(
                    &mut writer,
                    &path.display().to_string(),
                    line_num + 1,
                    line,
                    0..cli.pattern.len(),
                    config,
                )?;
                match_count += 1;
            }
        }
    }

    pb.finish_and_clear();
    Ok(match_count)
}
```

With `builder.build()` you get single-threaded walking. For parallel walking (what ripgrep actually uses), call `builder.build_parallel()` and provide a closure for each thread. The API handles the work-stealing, thread spawning, and synchronization.

### Shell completions for free

clap generates shell completions from your CLI definition. Use the newer runtime approach with `clap_complete::env` - no build script needed:

```bash
# User adds to their shell config:
source <(COMPLETE=bash codesearch)  # bash
source <(COMPLETE=zsh codesearch)   # zsh
COMPLETE=fish codesearch | source   # fish
```

Completions always stay in sync with the actual flags because they're generated from the same `Cli` struct that parses arguments. No stale completion files.

Or generate them at build time via `build.rs`:

```rust
// build.rs
use clap::CommandFactory;
use clap_complete::generate_to;
use clap_complete::shells::{Bash, Fish, Zsh};

include!("src/cli.rs");

fn main() {
    let out_dir = std::env::var("OUT_DIR").unwrap();
    let mut cmd = Cli::command();

    generate_to(Bash, &mut cmd, "codesearch", &out_dir).unwrap();
    generate_to(Zsh, &mut cmd, "codesearch", &out_dir).unwrap();
    generate_to(Fish, &mut cmd, "codesearch", &out_dir).unwrap();
}
```

## What ripgrep's architecture teaches about crate design

One thing that stands out about ripgrep's [codebase](https://github.com/BurntSushi/ripgrep) is its workspace structure. It's not one big binary crate - it's a collection of focused library crates: `grep-matcher` (trait abstraction for regex engines), `grep-regex` (Rust regex backend), `grep-pcre2` (PCRE2 backend), `grep-searcher` (line-oriented search), `grep-printer` (output formatting), `grep-cli` (argument handling), `ignore` (directory walking), and `globset` (glob matching).

The `grep-matcher` crate defines a trait that both `grep-regex` and `grep-pcre2` implement - the [strategy pattern](/blog/the-strategy-pattern-in-rust-polymorphism-done-right/) in action, letting ripgrep swap regex engines without touching any search logic.

The `ignore` crate is published independently on crates.io and used by fd, [tokei](https://github.com/XAMPPRocky/tokei), and dozens of other tools. Same with `globset`. This decomposition means ripgrep's innovations benefit the entire ecosystem, not just one binary.

If you're building a CLI tool that might grow, consider extracting your core logic into library crates from the start. The binary crate handles arguments, output formatting, and error presentation. The library crates handle the actual work. This separation makes testing easier (test the library directly without CLI invocation), enables embedding your tool as a library in other projects, and forces clean API boundaries.

## Config files: two schools of thought

bat follows the [XDG Base Directory Specification](https://specifications.freedesktop.org/basedir-spec/latest/) - its config lives at `~/.config/bat/config` (or `$BAT_CONFIG_PATH`). Each line is a flag, like `--theme="Dracula"` or `--style="numbers,changes,grid"`. Users can customize defaults without aliases.

ripgrep takes a deliberately different approach. There's no auto-loaded config file. Instead, you set `RIPGREP_CONFIG_PATH` to point at a file, one flag per line. Andrew Gallant's rationale ([issue #1719](https://github.com/BurntSushi/ripgrep/issues/1719)): auto-loading config from XDG paths creates implicit behavior that's hard to debug. If `rg` behaves differently on your machine versus CI, you need to know exactly where to look.

Both approaches have merit. The takeaway: if your tool supports config files, make it obvious where it reads from. `mytool --dump-config` or `mytool --config-path` saves users from hunting through dotfile directories.

For Rust, the [`dirs`](https://crates.io/crates/dirs) crate gives you platform-native config paths:

```rust
use std::path::PathBuf;

fn config_path() -> Option<PathBuf> {
    dirs::config_dir().map(|d| d.join("codesearch").join("config"))
}
// Linux:   ~/.config/codesearch/config
// macOS:   ~/Library/Application Support/codesearch/config
// Windows: C:\Users\<user>\AppData\Roaming\codesearch\config
```

## The patterns that matter

After studying these three tools, here's the distilled checklist:

**1. Default to what users expect, not what's easiest to implement.** ripgrep could just search every file like grep does. That's less code. But it would be a worse tool.

**2. Detect your context.** Terminal gets colors and paging. Pipe gets plain text in a machine-readable format. Check `IsTerminal` and `NO_COLOR`. This single behavior change determines whether your tool composes with the rest of the shell ecosystem.

**3. Two-tier help.** `-h` is a cheat sheet. `--help` is a reference manual. Group flags by purpose. Show defaults.

**4. Exit codes are an API.** `0` for success, non-zero for different failure modes. Scripts depend on this. Document your exit codes.

**5. Progress on stderr, data on stdout.** Progress bars, warnings, prompts - all stderr. Actual output - stdout. This lets `mytool | jq .` work while showing a spinner.

**6. Respect `.gitignore` if your tool walks directories.** The `ignore` crate gives you this for free. Users don't want results from `target/`, `node_modules/`, or `.git/`.

**7. Ship completions.** `clap_complete` generates them from your type definitions. Zero excuse for a modern CLI tool not to have shell completions.

## The bar has been raised

Ten years ago, a CLI tool that worked and had a `--help` flag was sufficient. Today, developers expect instant feedback with colors, smart defaults that don't require reading a man page, correct behavior when piped versus interactive, shell completions, and readable error messages.

ripgrep, fd, and bat didn't just happen to be good tools. They were deliberately designed around these principles. The Rust ecosystem - with clap, owo-colors, indicatif, the `ignore` crate, and `std::io::IsTerminal` - gives you every building block to match that bar.

The hard part isn't the implementation. It's the discipline to think about your defaults before you think about your features.
