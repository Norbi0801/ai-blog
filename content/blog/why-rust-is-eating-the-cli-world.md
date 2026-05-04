+++
title = "Why Rust is eating the CLI world"
date = 2026-04-26
description = "Ubuntu 26.04 is replacing coreutils with Rust. ripgrep, fd, bat, delta, zoxide already won. Here's what makes Rust the default language for modern CLI tools."

[taxonomies]
tags = ["rust", "cli", "tools", "linux"]
+++

Ubuntu 26.04 LTS will ship with [uutils/coreutils](https://github.com/uutils/coreutils) - a Rust rewrite of the GNU coreutils that have been the backbone of every Linux system since the early 90s. `ls`, `cp`, `mv`, `cat`, `sort`, `cut`, `head`, `tail` - all of them, rewritten in Rust. Sylvestre Ledru presented the results at [FOSDEM 2026](https://fosdem.org/2026/schedule/event/DTYYL9-rust-coreutils/): 96% GNU compatibility in uutils 0.6.0, with remaining differences treated as bugs, not features.

This isn't a hobby project or a conference demo. This is Canonical saying "we trust Rust enough to replace the foundation of our operating system." And the thing is, for anyone who's been paying attention, this was inevitable. The Rust CLI ecosystem didn't sneak up on anyone. It's been quietly replacing the UNIX toolbox, one command at a time, for years now.

<!-- more -->

## The replacement map

Here's what the modern Rust CLI toolkit looks like, mapped against the tools it replaces:

| Traditional | Rust replacement | GitHub stars |
|------------|-----------------|-------------|
| `grep` | [ripgrep](https://github.com/BurntSushi/ripgrep) (`rg`) | ~61k |
| `find` | [fd](https://github.com/sharkdp/fd) | ~42k |
| `cat` | [bat](https://github.com/sharkdp/bat) | ~57k |
| `ls` | [eza](https://github.com/eza-community/eza) | ~16k |
| `cd` (sort of) | [zoxide](https://github.com/ajaxenabled/zoxide) | ~27k |
| `du` | [dust](https://github.com/bootandy/dust) | ~10k |
| `git diff` | [delta](https://github.com/dandavison/delta) | ~27k |
| `wc` (for code) | [tokei](https://github.com/XAMPPRocky/tokei) | ~12k |
| `top`/`htop` | [bottom](https://github.com/ClementTsang/bottom) | ~11k |
| `ps` | [procs](https://github.com/dalance/procs) | ~5k |
| `time` | [hyperfine](https://github.com/sharkdp/hyperfine) | ~24k |
| shell prompt | [starship](https://github.com/starship/starship) | ~50k |
| `sed` (partial) | [sd](https://github.com/chmln/sd) | ~6k |
| `bash` | [nushell](https://github.com/nushell/nushell) | ~35k |

That's not a niche. That's a parallel universe of command-line tooling. And these aren't obscure crates with 12 stars - they collectively represent hundreds of thousands of GitHub stars and hundreds of millions of downloads on crates.io.

If you've written `rg` instead of `grep` even once, you already live in this universe.

## Why these tools won (it's not just speed)

The obvious answer is "Rust is fast." And it is. ripgrep is [up to 300x faster than grep](https://www.codeant.ai/blogs/ripgrep-vs-grep-performance) with default settings on modern codebases (that drops to about 10-20x when you give grep the same ignore rules). fd [benchmarks at 5-23x faster than find](https://github.com/sharkdp/fd) depending on the search pattern. These aren't marginal gains.

But raw speed isn't the whole story. `grep` was already fast enough for most things. If speed alone determined adoption, we'd all be using hand-tuned C programs for everything. The Rust CLI tools won on three fronts simultaneously: distribution, defaults, and correctness.

### Distribution: one binary, zero dependencies

You download a single file. You put it somewhere in your `$PATH`. It works.

No runtime to install. No `python3.11` vs `python3.12` mismatch. No `node_modules`. No `.so` files to chase down. No `apt install libfoo-dev` before compilation. No virtualenv. No nvm.

A stripped ripgrep binary is about 6 MB. fd is around 3.5 MB. bat is about 5.5 MB. These are statically-linked executables that contain everything they need.

Compare that to a Python CLI tool. Even a simple one pulls in the Python interpreter (~30 MB), plus its dependencies, plus whatever shim system (pipx, pip install --user, brew) you used to install it. And then there's the startup tax. Rust programs start in under a millisecond - [measured at 0.5ms on an Intel i5](https://github.com/bdrung/startup-time). Python 3.6 takes [25-30ms just to start the interpreter](https://github.com/bdrung/startup-time), before a single line of your code runs. On a Raspberry Pi, that gap widens to 4ms vs 200ms.

For interactive CLI tools, that startup time matters. When you alias `cat` to `bat` and run it 50 times during a debugging session, you don't want to feel the tool loading.

If you're interested in how to cross-compile these single binaries for every platform, I covered the full setup - rustup targets, musl static linking, cargo-zigbuild, cross-rs - in [Cross-compilation in Rust](/blog/cross-compilation-in-rust-building-for-linux-from-macos). And for Docker distribution, [Docker multi-stage builds for Rust](/blog/docker-multi-stage-builds-for-rust-from-2gb-to-20mb) shows how to go from a 2GB image to under 20MB.

### Defaults: the tools are opinionated in the right way

I wrote about this in detail in [Designing good CLI tools](/blog/designing-good-cli-tools-lessons-from-ripgrep-fd-and-bat), but it's worth restating here because it's the real adoption driver.

`find . -type f -name "*.rs" -not -path "*/target/*"` vs `fd .rs`. Same result. One of them respects your `.gitignore`, uses regex by default, colorizes output, and runs multithreaded out of the box.

ripgrep skips `.gitignore`d files, ignores hidden directories, avoids binary files, and colorizes matches - all without flags. bat adds syntax highlighting, line numbers, and Git integration to `cat`. eza shows Git status, file permissions in color, and tree views by default.

These tools made design decisions that match how developers actually work in 2026, not how POSIX committees designed shell utilities in 1988.

### Correctness: memory safety without a garbage collector

This is the argument that matters at the Ubuntu level. When you're shipping tools as part of an operating system - tools that process untrusted input, run as root, handle filenames with arbitrary bytes - memory safety isn't a nice-to-have. It's an attack surface.

GNU coreutils have had their share of CVEs. Buffer overflows in `sort`, integer overflows in `split`, use-after-free bugs discovered through fuzzing. These are the kinds of bugs that Rust's ownership system prevents at compile time. You literally cannot write a use-after-free in safe Rust. The compiler rejects it.

And Rust does this without a garbage collector. There's no GC pause when your `sort` is processing a 50GB file. No stop-the-world collection when your `cp` is in the middle of a large transfer. The memory is freed deterministically, at exactly the point where the compiler proves no one holds a reference to it anymore.

If you're not familiar with how ownership and zero-cost abstractions work under the hood, I covered the compiler mechanics - monomorphization, iterator fusion, LLVM optimization passes - in [Zero-cost abstractions in Rust](/blog/zero-cost-abstractions-in-rust-what-it-actually-means).

## Under the hood: what makes a Rust CLI binary different

Let's get concrete about what happens when you compile a Rust CLI tool.

### Static linking and musl

Most Rust CLI tools distributed as release binaries use musl libc instead of glibc. This means the binary is fully statically linked - it doesn't depend on *any* shared libraries on the target system. Not even libc.

```bash
$ file rg
rg: ELF 64-bit LSB executable, x86-64, statically linked,
    BuildID[sha1]=..., for GNU/Linux 3.2.0, stripped

$ ldd rg
        not a dynamic executable
```

That `not a dynamic executable` is the magic. This binary runs on any Linux kernel 3.2+ regardless of the distro, the glibc version, or what packages are installed. Alpine, Ubuntu, Amazon Linux, a bare Docker scratch image - it doesn't matter.

Compare with a typical Go binary:

```bash
$ file some-go-tool
some-go-tool: ELF 64-bit LSB executable, x86-64, statically linked,
              Go BuildID=..., stripped

$ ls -lh some-go-tool rg
-rwxr-xr-x 1 user user  12M Jun 24 10:00 some-go-tool
-rwxr-xr-x 1 user user 6.0M Jun 24 10:00 rg
```

Go produces static binaries too, and that's great. But Go binaries are typically larger because they embed their own runtime, goroutine scheduler, and garbage collector. Rust doesn't carry any of that. The binary is your code, the standard library functions you actually called (the linker strips the rest), and whatever crate code you pulled in. That's it.

### No runtime, for real

When people say "Rust has no runtime," they mean something specific. Here's what happens when a Rust binary starts:

```
_start (crt0)
  -> __libc_start_main (or musl equivalent)
    -> main()
      -> std::rt::lang_start()
        -> your main()
```

The Rust "runtime" is `lang_start`, which does three things: sets up a panic handler, initializes the thread-local storage for the main thread, and calls your `main()`. That's it. No JIT compilation. No bytecode interpreter. No heap pre-allocation. No class loading. No module resolution.

The kernel loads your binary, maps the segments, jumps to `_start`, and within microseconds you're running your code. This is why `hyperfine --warmup 3 'rg TODO'` gives you single-digit millisecond times for searching a small project.

### SIMD and the regex engine

ripgrep's speed isn't just "Rust is fast." Andrew Gallant wrote a custom regex engine ([regex](https://github.com/rust-lang/regex)) that uses SIMD instructions for literal string matching. When you search for a literal string like `TODO`, ripgrep doesn't use a regex automaton at all. It uses the Teddy algorithm - a SIMD-accelerated multi-pattern matcher that compares 16 or 32 bytes at a time using SSE2/AVX2 instructions.

Here's the rough idea. Instead of comparing one byte at a time:

```
input:   T h i s _ i s _ a _ T O D O _ l i n e
         ^
         check 'T'? yes -> check 'O'? no -> advance
```

The Teddy algorithm loads 16 bytes at once into an SSE register and checks all of them simultaneously using a fingerprint-based approach:

```
input:   [T h i s _ i s _ a _ T O D O _ l]  <- 16 bytes in one register
mask:    [fingerprint match on positions 0, 10]
verify:  position 10 -> "TODO" -> match!
```

This is why ripgrep searching for a literal string across a large codebase feels instantaneous. It's not doing character-by-character comparison. It's doing 16-or-32-bytes-at-once comparison, using the same SIMD instructions that video codecs and game engines use.

And the Rust type system makes this safe. The SIMD code uses `unsafe` internally (it has to - SIMD intrinsics are inherently unsafe), but it's wrapped in a safe API that the regex crate exposes. Users of ripgrep never touch unsafe code. The boundary is clear, audited, and fuzz-tested.

## The tools I actually use every day

I'm not going to list 20 tools and describe each in one sentence. Instead, here are the ones that actually changed my workflow, and specifically *why*.

### ripgrep (`rg`) over `grep`

The `.gitignore` awareness alone is worth the switch. In any Node.js, Rust, or Python project, `grep -r` without exclusion flags is unusable. `rg` just works. But the feature I use most is `-t` for type filtering:

```bash
# search only in Rust files
rg -t rust "pub fn" src/

# search only in TOML files
rg -t toml "version" .

# invert: search everything except test files
rg --glob '!*test*' "unwrap()"
```

And `rg --json` for piping structured output into other tools. grep's output is a string you have to parse. rg gives you JSON with file paths, line numbers, byte offsets, and match positions.

### fd over `find`

The command I type most often:

```bash
# find all Rust files modified in the last hour
fd -e rs --changed-within 1h

# find and delete all .DS_Store files
fd -H .DS_Store -x rm

# find executables
fd -t x
```

The `-x` flag (execute) replaces the nightmarish `find ... -exec {} \;` syntax. And fd is parallel by default - it uses multiple threads to walk the directory tree, which matters on NVMe drives where I/O isn't the bottleneck, CPU syscall processing is.

### bat over `cat`

I aliased `cat` to `bat --paging=never` two years ago and haven't looked back. Syntax highlighting in the terminal is genuinely useful when you're quickly checking a config file or reading a snippet of code. The Git integration shows modified lines in the gutter. And `bat --diff` gives you a syntax-highlighted side-by-side diff.

```bash
# quick look at a file with highlighting
bat src/main.rs

# show only lines 50-70
bat -r 50:70 src/main.rs

# use as a colorizing pager for other commands
kubectl logs mypod | bat -l log
```

### delta over `diff`

Configure it once in `.gitconfig`:

```ini
[core]
    pager = delta

[delta]
    navigate = true
    side-by-side = true
    line-numbers = true
```

Every `git diff`, `git log -p`, `git show` now has syntax highlighting, line numbers, and side-by-side view. The before/after is dramatic - default git diff is a wall of green and red text. delta turns it into something you can actually read.

### zoxide over `cd`

zoxide tracks which directories you visit and ranks them by frequency and recency. After a day of use:

```bash
# instead of: cd ~/projects/blog-drafts/src/features
z blog
# it just works - zoxide figured out which "blog" directory you mean

# fuzzy: jump to the most frecent match
z proj feat
# -> ~/projects/my-project/src/features
```

It hooks into your shell and overrides `cd` transparently. You don't learn new muscle memory. You just type shorter paths and it fills in the rest.

### hyperfine over `time`

The shell builtin `time` gives you one measurement, includes shell overhead, and formats the output differently across systems. hyperfine runs your command multiple times, calculates mean/median/stddev, does warmup runs, and can compare multiple commands:

```bash
$ hyperfine 'fd -e rs' 'find . -name "*.rs"'
Benchmark 1: fd -e rs
  Time (mean +- σ):      23.1 ms +-   1.2 ms
  Range (min ... max):    21.4 ms ...  26.8 ms

Benchmark 2: find . -name "*.rs"
  Time (mean +- σ):     338.7 ms +-  12.4 ms
  Range (min ... max):   321.2 ms ... 364.1 ms

Summary
  fd -e rs ran 14.67 +- 0.93 times faster than find . -name "*.rs"
```

This is the tool you use to back up your "it feels faster" claims with actual numbers.

## Why not Go?

Go also produces static binaries. Go also has a good CLI ecosystem (cobra, bubbletea). Go is also memory-safe (with a GC). So why did Rust win the CLI battle?

Three reasons:

**1. No GC pauses in hot paths.** When ripgrep is processing a 10GB log file, it's doing tight memory-mapped I/O with zero allocations in the hot loop. A Go implementation would need to manage the GC alongside the search, and while Go's GC is good, "no GC" beats "good GC" for this class of tool.

**2. Smaller binaries.** A minimal Rust CLI stripped and compiled with musl is around 300-500 KB. A minimal Go binary (which embeds the runtime and GC) starts at about 2 MB. For coreutils-scale tools that might be installed as system dependencies, that delta matters.

**3. The ecosystem built on itself.** ripgrep's regex engine is a crate. Other tools use it. ignore (the `.gitignore` matcher) is a crate. fd, bat, delta all use it. syntect (the syntax highlighter) is a crate. bat and delta both use it. The Rust CLI ecosystem has incredible component reuse - the same battle-tested libraries power dozens of tools. In Go, each tool tends to bring its own implementations.

That said, Go is still a great choice for CLI tools, especially when build speed matters or when your team already knows Go. The [bubbletea](https://github.com/charmbracelet/bubbletea) TUI framework is excellent. This isn't "Rust good Go bad" - it's "for the specific niche of high-performance system utilities that process untrusted input, Rust has structural advantages."

## Writing your own: getting started with clap

If you want to build a CLI tool in Rust, [clap](https://crates.io/crates/clap) is the standard argument parser. It's at version 4.5+ with 200M+ downloads on crates.io. The derive API makes argument parsing feel like defining a struct:

```rust
use clap::Parser;

/// Search for a pattern in files
#[derive(Parser)]
#[command(version, about)]
struct Cli {
    /// The pattern to search for
    pattern: String,

    /// The path to search in
    #[arg(default_value = ".")]
    path: String,

    /// Case-insensitive search
    #[arg(short, long)]
    ignore_case: bool,

    /// Maximum number of results
    #[arg(short, long, default_value_t = 100)]
    max_results: usize,

    /// File extensions to include
    #[arg(short = 't', long = "type", value_delimiter = ',')]
    file_types: Vec<String>,
}

fn main() {
    let cli = Cli::parse();

    println!("Searching for '{}' in {}", cli.pattern, cli.path);

    if cli.ignore_case {
        println!("  (case-insensitive)");
    }

    if !cli.file_types.is_empty() {
        println!("  file types: {}", cli.file_types.join(", "));
    }
}
```

That's it. You get `--help`, `--version`, short flags, long flags, default values, value validation, and shell completion generation - all from struct annotations. The generated help looks like this:

```
Search for a pattern in files

Usage: mysearch [OPTIONS] <PATTERN> [PATH]

Arguments:
  <PATTERN>  The pattern to search for
  [PATH]     The path to search in [default: .]

Options:
  -i, --ignore-case            Case-insensitive search
  -m, --max-results <MAX>      Maximum number of results [default: 100]
  -t, --type <FILE_TYPES>      File extensions to include
  -h, --help                   Print help
  -V, --version                Print version
```

### Adding color and error handling

Two crates complete the CLI starter kit:

[anyhow](https://crates.io/crates/anyhow) for error handling in application code (as opposed to library code):

```rust
use anyhow::{Context, Result};
use std::fs;

fn read_config(path: &str) -> Result<String> {
    fs::read_to_string(path)
        .with_context(|| format!("failed to read config from '{}'", path))
}

fn main() -> Result<()> {
    let config = read_config("config.toml")?;
    // ...
    Ok(())
}
```

And [colored](https://crates.io/crates/colored) (or [owo-colors](https://crates.io/crates/owo-colors) if you want zero-alloc) for terminal colors:

```rust
use colored::*;

fn print_match(path: &str, line_num: usize, line: &str, pattern: &str) {
    let highlighted = line.replace(
        pattern,
        &pattern.red().bold().to_string()
    );
    println!(
        "{}:{} {}",
        path.green(),
        line_num.to_string().yellow(),
        highlighted
    );
}
```

I covered error messages in CLI tools much more thoroughly - including exit codes, miette for fancy diagnostics, and user-facing error design - in [Error messages that help](/blog/error-messages-that-help-designing-user-facing-errors-in-cli-tools). And if you want to go beyond basic argument parsing into full terminal UIs, [Building a TUI app in Rust with ratatui](/blog/building-a-tui-app-in-rust-with-ratatui) walks through the event loop, layout system, and widgets.

### The Cargo.toml

Here's a realistic starting point for a CLI tool:

```toml
[package]
name = "mysearch"
version = "0.1.0"
edition = "2021"

[dependencies]
clap = { version = "4.5", features = ["derive"] }
anyhow = "1"
colored = "3"
regex = "1"
ignore = "0.4"        # .gitignore-aware directory walking (from ripgrep)
rayon = "1"            # parallel iterators

[profile.release]
opt-level = 3
lto = true             # link-time optimization
strip = true           # strip debug symbols
codegen-units = 1      # single codegen unit for better optimization
```

Those `[profile.release]` settings are what turn a 10 MB debug binary into a 2-3 MB release binary. `lto = true` enables cross-crate inlining (the linker can inline functions from dependencies into your code). `strip = true` removes debug symbols. `codegen-units = 1` sacrifices parallel compilation for better optimization - the compiler can see all your code at once and make smarter decisions.

### Directory walking with the ignore crate

The `ignore` crate is ripgrep's secret weapon, extracted as a reusable library. It gives you a parallel directory walker that respects `.gitignore`, `.ignore`, and `.fdignore` files:

```rust
use ignore::WalkBuilder;
use std::path::Path;

fn walk_files(root: &Path, file_types: &[String]) -> Vec<String> {
    let mut builder = WalkBuilder::new(root);
    builder.hidden(true)       // skip hidden files
           .git_ignore(true)   // respect .gitignore
           .git_global(true)   // respect global gitignore
           .git_exclude(true); // respect .git/info/exclude

    builder.build()
        .filter_map(|entry| entry.ok())
        .filter(|entry| entry.file_type().map_or(false, |ft| ft.is_file()))
        .filter(|entry| {
            if file_types.is_empty() {
                return true;
            }
            entry.path()
                .extension()
                .and_then(|ext| ext.to_str())
                .map_or(false, |ext| file_types.iter().any(|ft| ft == ext))
        })
        .map(|entry| entry.path().display().to_string())
        .collect()
}
```

This is the same walker that ripgrep and fd use internally. By pulling it into your own tool, you get the same `.gitignore` behavior that makes those tools feel magic. Your users don't have to think about it - the tool just skips the right files.

## What comes next

The Ubuntu decision isn't the end of this story. It's a checkpoint. uutils reaching [96% GNU compatibility](https://www.phoronix.com/news/Rust-Coreutils-FOSDEM-2026) and shipping in a major LTS release signals that Rust CLI tools have moved past the "cool alternative" stage into "production infrastructure."

The pattern is clear: wherever there's a C or C++ utility that processes untrusted input, somebody is writing a Rust replacement. And increasingly, that replacement is winning not because it's written in Rust, but because the author used that rewrite as an opportunity to rethink the defaults, modernize the UX, and build on a shared ecosystem of high-quality crates.

If you're building a CLI tool today and choosing between Python, Go, and Rust: pick whatever you're productive in. Seriously. A shipped tool in Python beats a perfect Rust tool that never gets finished. But if you're already comfortable with Rust, or if your tool needs to be distributed as a single binary, process large inputs efficiently, or run on systems where you can't install a runtime - Rust isn't just a good choice. It's becoming the obvious one.
