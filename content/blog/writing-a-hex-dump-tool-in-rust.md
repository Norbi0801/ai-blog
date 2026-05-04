+++
title = "Writing a hex dump tool in Rust"
date = 2026-05-01
description = "Building a small xxd/hexdump alternative in about 150 lines of Rust to learn binary I/O, the Read trait, ANSI colors, and how text encodings actually look on disk."

[taxonomies]
tags = ["rust", "cli", "binary", "beginner"]
+++

If you have ever debugged a corrupt PNG, peeked at a TLS handshake, or tried to figure out which UTF-8 abomination broke your CSV parser, you have used a hex dump tool. `xxd` ships with Vim, `hexdump` ships with util-linux, BSD has `od`, and modern alternatives like [`hexyl`](https://github.com/sharkdp/hexyl) make the output prettier. They all do the same thing: take a stream of bytes and show you the offset, the bytes in hex, and the printable ASCII for each row.

Writing one yourself is a near-perfect first Rust project. You touch slices of `u8`, the `Read` trait, file I/O, stdin piping, formatted output, ANSI escape codes, and command line argument parsing. You finish with a working tool you can drop into your `~/.local/bin` and actually use. No async, no lifetimes contortions, no ecosystem soup. Pure stdlib, around 150 lines.

We'll build a tool called `rxd` that supports:

- Reading from a file or stdin (so it works in pipelines)
- 16 bytes per row with offset on the left, ASCII on the right
- Configurable byte grouping (1, 2, 4, 8)
- Color coding for printable, control, whitespace, null, and high bytes
- Searching for a substring (ASCII or hex) and inverting the matches

<!-- more -->

## What a hex dump line looks like

Before writing any code, look at one row of `xxd` output:

```
00000010: 4865 6c6c 6f20 576f 726c 6421 0a00 deff  Hello World!....
```

There are three columns:

1. The **offset**, in hex, padded to 8 digits. This is the position in the file of the first byte on this row.
2. The **hex bytes**. Sixteen of them, two ASCII chars each, with spaces or grouping in between.
3. The **printable ASCII**, with `.` substituted for non-printable bytes.

That's the whole format. The width is fixed (so columns line up), missing bytes on the last partial row are padded with spaces, and the ASCII column is wrapped in pipes (`|...|`) so you can spot trailing whitespace.

A hex dump is a transformation: bytes go in, structured text comes out. That fits Unix philosophy and it fits Rust's `Read` and `Write` traits cleanly.

## Why hex at all

Computers store everything as bytes, which are 8-bit unsigned integers in the range `0..=255`. Hex is base 16, so each byte is exactly two hex digits (`00` to `ff`). That's the only reason hex won. Decimal would need three digits per byte and column alignment would be ugly. Binary would need eight. Octal makes sense if you grew up on PDP-11s but the rest of us moved on. Two hex digits per byte gives you a one-to-one correspondence between visual columns and the underlying memory.

A `u8` in Rust is exactly one byte. The compiler will not let you write `let x: u8 = 256;` because that overflows, and it will reject `let x: u8 = -1;` for the same reason. When you read a file, you get a `&[u8]` (a slice of bytes). That slice is the file. Nothing more, nothing less.

## Project setup

```toml
# Cargo.toml
[package]
name = "rxd"
version = "0.1.0"
edition = "2021"

[dependencies]
```

No dependencies. Everything we need is in `std`: `std::fs::File`, `std::io::Read`, `std::io::Write`, `std::env`, `std::process::ExitCode`. We won't even reach for `clap` for argument parsing, because the surface is small enough to handle by hand and a beginner project shouldn't hide what's actually happening.

## Reading bytes - the `Read` trait

The `Read` trait in `std::io` is the universal interface for "thing you can pull bytes out of." Its core method is:

```rust
fn read(&mut self, buf: &mut [u8]) -> io::Result<usize>;
```

You hand it a mutable byte buffer. It fills as much of the buffer as it can and returns how many bytes it actually wrote. A return of `Ok(0)` means EOF. The contract says nothing about filling the whole buffer in one call - sockets, pipes, and slow devices may give you partial reads.

`File` implements `Read`. So does `TcpStream`. So does `Stdin`. So does `&[u8]`. Anything that produces a stream of bytes. That's why most stdlib I/O code is generic over `R: Read`: you write the function once and it works for any source. This is the same design Go uses for `io.Reader` and Java uses for `InputStream`, but the Rust version is statically dispatched (zero-cost) when you use generics.

For our tool we just want everything in memory, so we use the convenience method `read_to_end`:

```rust
use std::io::Read;

fn slurp(mut r: impl Read) -> std::io::Result<Vec<u8>> {
    let mut v = Vec::new();
    r.read_to_end(&mut v)?;
    Ok(v)
}
```

`read_to_end` loops, calling `read` on the source until it returns `Ok(0)`, growing the `Vec<u8>` along the way. Yes, this means a 5 GB file will consume 5 GB of RAM. For a hex dump tool, that's almost always fine - if you're hex-dumping a 5 GB file the human reading the output is going to give up at megabyte one. If you cared, you'd `read` into a fixed 64 KiB buffer and emit lines as you go, which is what `hexyl` does. We'll keep it simple.

## Wiring in stdin

The reason `xxd file.bin` and `cat file.bin | xxd` both work is that `xxd` reads from stdin when no file argument is given. Rust makes this easy because `Stdin` and `File` both implement `Read`:

```rust
use std::fs::File;
use std::io::{self, Read};

let data: Vec<u8> = match cfg.path.as_deref() {
    Some(p) => {
        let mut f = File::open(p)?;
        let mut v = Vec::new();
        f.read_to_end(&mut v)?;
        v
    }
    None => {
        let mut v = Vec::new();
        io::stdin().lock().read_to_end(&mut v)?;
        v
    }
};
```

The `.lock()` call on `Stdin` is worth pausing on. `io::stdin()` returns a global handle protected by a mutex so multiple threads don't garble each other's reads. Every `read` call on the unlocked handle takes the lock, does the read, releases the lock. If you're reading a lot of data, that mutex churn matters. `.lock()` returns a `StdinLock` that holds the lock for its lifetime - one acquire, fast subsequent reads.

The same trick applies to stdout: if you `println!` a million times, each call locks. If you `let mut out = io::stdout().lock();` and `writeln!(out, ...)`, you lock once.

## Formatting hex - what `{:02x}` actually does

```rust
let b: u8 = 0x4a;
print!("{:02x}", b); // prints "4a"
```

`{:02x}` is `std::fmt`'s format spec for "hexadecimal, lowercase, padded to width 2 with leading zeros." For `u8` this is exactly what you want: every byte becomes two hex digits. Without the `02`, `0x05` would print as just `5` and your columns would shift. Without the `x`, you'd get decimal. With `X` you'd get uppercase.

Under the hood, this goes through `LowerHex` (for `x`) or `UpperHex` (for `X`). They are traits in `std::fmt`, implemented for every integer type. The compiler's inliner generally sees through the formatter machinery for simple cases, so this is not the bottleneck people sometimes assume it is. If you ever need to be paranoid, you can build the lookup table yourself - it's two static `&[u8; 256]` arrays of pre-rendered byte pairs, the same trick `hex` and `faster-hex` use.

For the offset on the left we use `{:08x}` - same idea, padded to 8 hex digits. That covers 4 GiB of input before the column wraps, which is enough.

## ASCII printable range

The right column shows printable ASCII. The "printable" range is `0x20` (space) through `0x7e` (`~`) inclusive. `0x7f` is DEL, which is technically a control character and not printable. Anything below `0x20` is a control character (tab, newline, bell, etc.). Anything above `0x7e` is non-ASCII - either part of a UTF-8 multibyte sequence or an unrelated encoding like Latin-1.

```rust
fn ascii(b: u8) -> u8 {
    if (0x20..0x7f).contains(&b) { b } else { b'.' }
}
```

Note we use `..0x7f` (exclusive on the right), not `..=0x7e`, but both express the same set. I prefer the exclusive form because the boundary is "below DEL" which is the actual rule.

## Color coding by category

ANSI escape codes are how terminals do color. They're literally bytes you write into stdout that the terminal interprets. The basic foreground color escape is `ESC [ N m` where `ESC` is `\x1b`, `[` is literal, `N` is a number from 30-37 or 90-97 (foreground colors), and `m` ends the sequence. To reset, you write `ESC [ 0 m`.

```rust
const RESET:  &[u8] = b"\x1b[0m";
const RED:    &[u8] = b"\x1b[31m";  // null byte
const GREEN:  &[u8] = b"\x1b[32m";  // printable ASCII
const YELLOW: &[u8] = b"\x1b[33m";  // whitespace
const BLUE:   &[u8] = b"\x1b[34m";  // high (>= 0x80)
const GREY:   &[u8] = b"\x1b[90m";  // other control bytes
const INV:    &[u8] = b"\x1b[7m";   // inverted (search hit)

fn color_of(b: u8) -> &'static [u8] {
    match b {
        0 => RED,
        b'\t' | b'\n' | b'\r' | b' ' => YELLOW,
        0x21..=0x7e => GREEN,
        0x80..=0xff => BLUE,
        _ => GREY, // 0x01..=0x1f and 0x7f, the control range
    }
}
```

That `match` is exhaustive over `u8` because Rust requires it. The compiler verifies you handled every variant. If you ever see a `_ => unreachable!()` arm in someone else's match on `u8`, that's a smell - you can almost always express the missing range explicitly.

We skip color when stdout isn't a TTY. Since Rust 1.70 you can check with `IsTerminal`:

```rust
use std::io::IsTerminal;
let color = io::stdout().is_terminal();
```

This avoids spraying escape codes into pipes and files, which is what every well-behaved CLI does.

## Grouping

`xxd` defaults to grouping pairs of bytes (groupsize 2). `hexdump` groups by 2 with `-C`. Some people want 4 (32-bit words), some want 8 (64-bit), some want 1 (every byte separated). It's a pure formatting decision: insert an extra space between groups.

```rust
for i in 0..16 {
    if i > 0 && i % cfg.group == 0 {
        out.write_all(b" ")?; // single space between groups
    }
    if i < line.len() {
        write_hex_byte(out, line[i], is_hit(chunk_start + i), cfg.color)?;
    } else {
        out.write_all(b"  ")?; // pad missing byte slot
    }
}
```

With `group = 1` we get `48 65 6c 6c 6f` (one space between each byte). With `group = 4` we get `48656c6c 6f2c2057` (four bytes joined, single space between groups). With `group = 8` the row collapses into two 16-char clumps. The total width of the hex column stays constant per row because missing-byte slots still emit two spaces, and the group separators are still printed for absent bytes too.

## Search

Searching binary data is just substring matching on `&[u8]`. We accept either an ASCII string (`-s GET`) or a hex pattern (`-x deadbeef`). The hex parser is a few lines:

```rust
fn parse_hex(s: &str) -> Result<Vec<u8>, String> {
    let s = s.trim_start_matches("0x");
    if s.len() % 2 != 0 { return Err("hex needs even length".into()); }
    s.as_bytes().chunks(2).map(|p| {
        let hi = hex_digit(p[0])?;
        let lo = hex_digit(p[1])?;
        Ok((hi << 4) | lo)
    }).collect()
}
```

Once you have the needle as `Vec<u8>`, finding all occurrences in the haystack is a naive scan. For 99% of files this is fine. If you wanted to dump the full Linux kernel and search it, you'd reach for `memchr::memmem` which uses SIMD and runs at memory-bandwidth speed.

```rust
fn find_all(haystack: &[u8], needle: &[u8]) -> Vec<usize> {
    if needle.is_empty() || haystack.len() < needle.len() { return vec![]; }
    let mut hits = Vec::new();
    let mut i = 0;
    while i + needle.len() <= haystack.len() {
        if &haystack[i..i + needle.len()] == needle {
            hits.push(i);
            i += needle.len();
        } else {
            i += 1;
        }
    }
    hits
}
```

The slice comparison `&haystack[i..i + needle.len()] == needle` compiles to `memcmp`. Slice equality on `[u8]` is a single intrinsic call; it doesn't loop in your code.

To highlight, we mark every byte position covered by a match and wrap those bytes in the inverse-video escape (`\x1b[7m`) when we render them.

## The full program

Putting it together:

```rust
use std::env;
use std::fs::File;
use std::io::{self, BufWriter, IsTerminal, Read, Write};
use std::process::ExitCode;

const RESET: &[u8]  = b"\x1b[0m";
const RED: &[u8]    = b"\x1b[31m";
const GREEN: &[u8]  = b"\x1b[32m";
const YELLOW: &[u8] = b"\x1b[33m";
const BLUE: &[u8]   = b"\x1b[34m";
const GREY: &[u8]   = b"\x1b[90m";
const INV: &[u8]    = b"\x1b[7m";

struct Cfg {
    path: Option<String>,
    group: usize,
    needle: Option<Vec<u8>>,
    color: bool,
}

fn die(msg: &str) -> ! {
    eprintln!("rxd: {msg}");
    std::process::exit(2);
}

fn parse_args() -> Cfg {
    let mut cfg = Cfg {
        path: None,
        group: 1,
        needle: None,
        color: io::stdout().is_terminal(),
    };
    let mut args = env::args().skip(1);
    while let Some(a) = args.next() {
        match a.as_str() {
            "-g" | "--group" => {
                let n: usize = args.next().and_then(|s| s.parse().ok()).unwrap_or(0);
                if ![1, 2, 4, 8].contains(&n) { die("-g must be 1, 2, 4, or 8"); }
                cfg.group = n;
            }
            "-s" | "--string" => {
                cfg.needle = Some(args.next().unwrap_or_else(|| die("-s needs arg")).into_bytes());
            }
            "-x" | "--hex" => {
                let s = args.next().unwrap_or_else(|| die("-x needs arg"));
                cfg.needle = Some(parse_hex(&s).unwrap_or_else(|e| die(&e)));
            }
            "--no-color" => cfg.color = false,
            "-h" | "--help" => { print_help(); std::process::exit(0); }
            _ if a.starts_with('-') => die(&format!("unknown flag: {a}")),
            _ => cfg.path = Some(a),
        }
    }
    cfg
}

fn hex_digit(b: u8) -> Result<u8, String> {
    match b {
        b'0'..=b'9' => Ok(b - b'0'),
        b'a'..=b'f' => Ok(b - b'a' + 10),
        b'A'..=b'F' => Ok(b - b'A' + 10),
        _ => Err(format!("invalid hex digit: {}", b as char)),
    }
}

fn parse_hex(s: &str) -> Result<Vec<u8>, String> {
    let s = s.trim_start_matches("0x");
    if s.len() % 2 != 0 { return Err("hex string needs even length".into()); }
    s.as_bytes().chunks(2).map(|p| {
        Ok((hex_digit(p[0])? << 4) | hex_digit(p[1])?)
    }).collect()
}

fn find_all(hay: &[u8], needle: &[u8]) -> Vec<usize> {
    if needle.is_empty() || hay.len() < needle.len() { return vec![]; }
    let mut hits = Vec::new();
    let mut i = 0;
    while i + needle.len() <= hay.len() {
        if &hay[i..i + needle.len()] == needle { hits.push(i); i += needle.len(); }
        else { i += 1; }
    }
    hits
}

fn color_of(b: u8) -> &'static [u8] {
    match b {
        0 => RED,
        b'\t' | b'\n' | b'\r' | b' ' => YELLOW,
        0x21..=0x7e => GREEN,
        0x80..=0xff => BLUE,
        _ => GREY,
    }
}

fn write_hex(out: &mut impl Write, b: u8, hit: bool, color: bool) -> io::Result<()> {
    if color {
        if hit { out.write_all(INV)?; }
        out.write_all(color_of(b))?;
        write!(out, "{:02x}", b)?;
        out.write_all(RESET)?;
    } else {
        write!(out, "{:02x}", b)?;
    }
    Ok(())
}

fn dump(data: &[u8], cfg: &Cfg, out: &mut impl Write) -> io::Result<()> {
    let hits = match &cfg.needle {
        Some(n) => find_all(data, n),
        None => Vec::new(),
    };
    let nlen = cfg.needle.as_ref().map_or(0, |n| n.len());
    let is_hit = |abs: usize| hits.iter().any(|&s| abs >= s && abs < s + nlen);

    let mut off = 0;
    while off < data.len() {
        let end = (off + 16).min(data.len());
        let line = &data[off..end];

        write!(out, "{:08x}  ", off)?;
        for i in 0..16 {
            if i > 0 && i % cfg.group == 0 { out.write_all(b" ")?; }
            if i < line.len() {
                write_hex(out, line[i], is_hit(off + i), cfg.color)?;
            } else {
                out.write_all(b"  ")?;
            }
        }
        out.write_all(b"  |")?;
        for (i, &b) in line.iter().enumerate() {
            let c = if (0x20..0x7f).contains(&b) { b } else { b'.' };
            if cfg.color {
                if is_hit(off + i) { out.write_all(INV)?; }
                out.write_all(color_of(b))?;
                out.write_all(&[c])?;
                out.write_all(RESET)?;
            } else {
                out.write_all(&[c])?;
            }
        }
        out.write_all(b"|\n")?;
        off += 16;
    }
    Ok(())
}

fn print_help() {
    println!("rxd - tiny hex dump tool");
    println!("usage: rxd [FILE] [OPTIONS]");
    println!("  -g, --group N      bytes per group (1, 2, 4, 8)");
    println!("  -s, --string S     highlight ASCII string S");
    println!("  -x, --hex H        highlight hex bytes H (e.g. deadbeef)");
    println!("      --no-color     disable ANSI colors");
    println!("  -h, --help         show this help");
}

fn main() -> ExitCode {
    let cfg = parse_args();
    let data = match &cfg.path {
        Some(p) => {
            let mut v = Vec::new();
            match File::open(p).and_then(|mut f| f.read_to_end(&mut v).map(|_| ())) {
                Ok(()) => v,
                Err(e) => { eprintln!("rxd: {p}: {e}"); return ExitCode::from(1); }
            }
        }
        None => {
            let mut v = Vec::new();
            if let Err(e) = io::stdin().lock().read_to_end(&mut v) {
                eprintln!("rxd: stdin: {e}"); return ExitCode::from(1);
            }
            v
        }
    };
    let stdout = io::stdout().lock();
    let mut out = BufWriter::new(stdout);
    if let Err(e) = dump(&data, &cfg, &mut out) {
        if e.kind() != io::ErrorKind::BrokenPipe {
            eprintln!("rxd: {e}");
            return ExitCode::from(1);
        }
    }
    ExitCode::SUCCESS
}
```

That's right around 150 lines including the help text and config struct. `cargo build --release` produces a 400-500 KB binary on Linux x86_64. Strip it with `strip target/release/rxd` and you're under 350 KB.

## Trying it

```
$ echo -n "Hello, World!" | cargo run --quiet
00000000  48 65 6c 6c 6f 2c 20 57 6f 72 6c 64 21           |Hello, World!|

$ cargo run --quiet -- /bin/ls -g 4 -x 7f454c46 | head -3
00000000  7f454c46 02010100 00000000 00000000  |.ELF............|
00000010  03003e00 01000000 c0660000 00000000  |..>......f......|
00000020  40000000 00000000 88830200 00000000  |@...............|
```

That second example is searching for `7f 45 4c 46`, which is the ELF magic number. `7f` is DEL, `45` is `E`, `4c` is `L`, `46` is `F`. Every Linux executable starts with those four bytes. The first row shows them highlighted.

## A note on `BrokenPipe`

The reason for the `BrokenPipe` check at the end is that `rxd big-file | head` will close the pipe after `head` has its 10 lines, and any further `write_all` from `rxd` will return `BrokenPipe`. Treating that as a hard error means every pipe-consumer combo prints "rxd: Broken pipe" to stderr, which is wrong. The convention is: on `BrokenPipe`, exit silently. `cat` does this. `grep` does this. Yours should too.

## Where to go next

A few worthwhile extensions, each of which teaches you something:

- Stream instead of buffering. Wrap input in `BufReader` and emit lines as 16-byte chunks fill up. You'll learn the difference between `read` and `read_exact`, and you'll handle partial reads properly.
- Add a write mode. `xxd -r` reverses a hex dump back into binary. The parser is small.
- Replace the naive search with `memchr::memmem` and benchmark it on a 100 MB file. The difference is 5x to 20x.
- Add `-n N` to limit output to the first N bytes, and `-S OFFSET` to seek into the file with `File::seek` before dumping.
- Add a JSON output mode. Once you have the data structured, emitting it as JSON is trivial and handy for scripts.

Hex dump is one of those tools that looks trivial until you start writing one. By the time you've handled grouping, partial rows, color, search highlights, broken pipes, and the printable ASCII boundary, you have a solid feel for `&[u8]`, the `Read` and `Write` traits, and how Unix CLI plumbing actually works. That foundation pays off immediately when you move on to harder binary formats - parsing PNG chunks, decoding TLS records, walking ELF section headers - because all of that is just hex dumping with structure on top.
