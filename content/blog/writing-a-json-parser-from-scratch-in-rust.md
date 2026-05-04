+++
title = "Writing a JSON parser from scratch in Rust"
date = 2025-05-10
description = "Building a complete JSON parser in ~200 lines of Rust with recursive descent, unicode escape handling, and position-tracked error reporting."

[taxonomies]
tags = ["rust", "parsing", "json"]
+++

JSON's grammar fits on a napkin. Six types, a handful of structural characters, and a few escape sequences. That's it. No ambiguity, no context-dependent rules, no 17-step delimiter algorithm like Markdown's emphasis parsing. This simplicity makes JSON the perfect target for your first "real" parser - complex enough to teach you recursive descent, simple enough that you won't drown in edge cases.

A few months ago I [wrote a Markdown parser from scratch](/blog/writing-a-markdown-parser-in-rust/) and it took ~280 lines to handle a subset of the spec. The JSON parser we're building today handles the *entire* spec in about 200 lines. That ratio tells you something about the difference in grammar complexity.

We're not using [serde_json](https://crates.io/crates/serde_json). If you need to parse JSON in production, use serde_json - I covered this in the [build vs buy post](/blog/build-vs-buy-when-to-use-a-library-and-when-to-write-your-own/). The point here is to understand what happens between `serde_json::from_str` and the value you get back.

<!-- more -->

## The grammar

[RFC 8259](https://datatracker.ietf.org/doc/html/rfc8259) defines JSON's grammar in ABNF notation. Here's the core of it, condensed:

```
value = object / array / string / number / "true" / "false" / "null"

object = "{" [ member *( "," member ) ] "}"
member = string ":" value

array  = "[" [ value *( "," value ) ] "]"

string = '"' *char '"'
char   = unescaped / '\' ( '"' / '\' / '/' / 'b' / 'f' / 'n' / 'r' / 't' / 'u' 4HEXDIG )

number = [ "-" ] int [ "." 1*DIGIT ] [ ("e"/"E") ["+"/ "-"] 1*DIGIT ]
int    = "0" / ( digit1-9 *DIGIT )
```

That's the whole language. No keywords, no operator precedence, no statement terminators. `value` is recursive (objects and arrays contain values), which is where the structure comes from. Everything else is flat.

Compare this to Markdown where heading detection depends on line prefixes, emphasis depends on flanking rules, and blockquotes can nest recursively with different prefix semantics. JSON's grammar is context-free and unambiguous. Every valid JSON document has exactly one parse tree.

This means we don't need a separate tokenizer. In languages with keywords and operators (like Rust itself), a tokenizer simplifies parsing by pre-classifying character sequences into tokens. For JSON, the first byte of each value already tells you what it is: `"` means string, `{` means object, `[` means array, `t`/`f`/`n` means literal, digit or `-` means number. The parser can act as its own tokenizer.

## The AST

If you read the [Markdown parser post](/blog/writing-a-markdown-parser-in-rust/), you've already seen the pattern: use Rust enums as sum types to represent the parse tree. JSON's value type maps directly:

```rust
#[derive(Debug, Clone, PartialEq)]
pub enum JsonValue {
    Null,
    Bool(bool),
    Number(f64),
    Str(String),
    Array(Vec<JsonValue>),
    Object(Vec<(String, JsonValue)>),
}
```

Six variants, one for each JSON type. `Array` and `Object` contain `Vec<JsonValue>` - that's where the recursion lives. A JSON document can nest arbitrarily deep, and the type system models this directly.

Two design choices worth calling out. First, `Object` uses `Vec<(String, JsonValue)>` instead of `HashMap`. This preserves insertion order, which RFC 8259 doesn't require but most JSON tools expect. A `HashMap` would also work, but you'd lose the ability to round-trip a document without reordering keys. Second, `Number` uses `f64`. This handles most JSON numbers correctly, but it can't represent integers larger than 2^53 without precision loss. Production parsers like serde_json solve this with a `Number` type that can hold either an integer or a float internally.

You can check how much memory this enum uses:

```rust
println!("{} bytes", std::mem::size_of::<JsonValue>());
// 32 bytes on 64-bit
```

The `Vec` variants (Array, Object) are 24 bytes (pointer + length + capacity), plus 8 bytes for the discriminant and alignment. Every `JsonValue::Null` occupies 32 bytes even though it carries no data - that's the tradeoff with enums.

## Error reporting with position tracking

A parser that just says "invalid JSON" is useless. We need line and column numbers:

```rust
use std::fmt;

#[derive(Debug)]
pub struct ParseError {
    pub msg: String,
    pub line: usize,
    pub col: usize,
}

impl fmt::Display for ParseError {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "{}:{}: {}", self.line, self.col, self.msg)
    }
}
```

The parser struct tracks position as a byte offset. When an error occurs, we scan backwards to compute line and column:

```rust
struct Parser<'a> {
    input: &'a [u8],
    pos: usize,
}

impl<'a> Parser<'a> {
    fn new(input: &'a str) -> Self {
        Self { input: input.as_bytes(), pos: 0 }
    }

    fn line_col(&self) -> (usize, usize) {
        let before = &self.input[..self.pos];
        let line = before.iter().filter(|&&b| b == b'\n').count() + 1;
        let col = match before.iter().rposition(|&b| b == b'\n') {
            Some(nl) => self.pos - nl,
            None => self.pos + 1,
        };
        (line, col)
    }

    fn err(&self, msg: impl Into<String>) -> ParseError {
        let (line, col) = self.line_col();
        ParseError { msg: msg.into(), line, col }
    }
}
```

This approach - storing a single `pos: usize` and computing line/col on demand - is deliberate. The alternative is tracking line and column incrementally, updating them every time we advance. That's faster for error reporting but slower for everything else because every character advance now touches two extra fields. Since errors are rare (most input is valid), lazy computation wins.

The `rposition` call finds the last newline before the current position. If there's no newline, we're on line 1 and the column equals the position. This handles both Unix and Windows line endings correctly enough for error messages (Windows `\r\n` would report the column as off by one on lines that start with `\r`, but that's an edge case not worth complicating the code for).

## The parser primitives

These are the building blocks - the lowest-level operations every parse function uses:

```rust
impl<'a> Parser<'a> {
    fn peek(&self) -> Option<u8> {
        self.input.get(self.pos).copied()
    }

    fn next_byte(&mut self) -> Option<u8> {
        let b = self.input.get(self.pos).copied()?;
        self.pos += 1;
        Some(b)
    }

    fn eat(&mut self, expected: u8) -> Result<(), ParseError> {
        match self.next_byte() {
            Some(b) if b == expected => Ok(()),
            Some(b) => Err(self.err(format!(
                "expected '{}', found '{}'", expected as char, b as char
            ))),
            None => Err(self.err(format!(
                "expected '{}', found EOF", expected as char
            ))),
        }
    }

    fn skip_ws(&mut self) {
        while let Some(b' ' | b'\t' | b'\n' | b'\r') = self.peek() {
            self.pos += 1;
        }
    }
}
```

`peek` looks at the current byte without consuming it. `next_byte` consumes and advances. `eat` consumes and asserts a specific byte, returning an error with position context if it doesn't match. `skip_ws` burns through whitespace (RFC 8259 defines whitespace as space, tab, newline, and carriage return only - no form feeds, no vertical tabs, no Unicode spaces).

If you squint, these are parser combinators. Libraries like [nom](https://crates.io/crates/nom) and [winnow](https://crates.io/crates/winnow) abstract this pattern into generic types and traits. We're doing it by hand, which means less abstraction overhead but also less reusability. For a single format like JSON, hand-rolled is fine.

## Value dispatch

The core of recursive descent: look at the first byte and decide what to parse.

```rust
fn value(&mut self) -> Result<JsonValue, ParseError> {
    self.skip_ws();
    match self.peek() {
        Some(b'"') => self.string().map(JsonValue::Str),
        Some(b'{') => self.object(),
        Some(b'[') => self.array(),
        Some(b't' | b'f') => self.boolean(),
        Some(b'n') => self.null(),
        Some(b) if b == b'-' || b.is_ascii_digit() => self.number(),
        Some(b) => Err(self.err(format!("unexpected '{}'", b as char))),
        None => Err(self.err("unexpected end of input")),
    }
}
```

This is the function that makes JSON parseable without a tokenizer. Every JSON value starts with a distinct byte: `"`, `{`, `[`, `t`, `f`, `n`, `-`, or `0-9`. No ambiguity, no lookahead beyond one byte. Compare this to Markdown where `*` could start italic, bold, or a list item depending on context - that's why the [Markdown parser](/blog/writing-a-markdown-parser-in-rust/) needed ordering heuristics and multi-character lookahead.

## Parsing strings

Strings are the trickiest part of JSON. Not because of the basic case (`"hello"` is straightforward), but because of escape sequences and the UTF-8/UTF-16 boundary.

```rust
fn string(&mut self) -> Result<String, ParseError> {
    self.eat(b'"')?;
    let mut s = String::new();
    loop {
        // Fast path: scan for plain bytes
        let start = self.pos;
        while self.pos < self.input.len() {
            match self.input[self.pos] {
                b'"' | b'\\' => break,
                b if b < 0x20 => break,
                _ => self.pos += 1,
            }
        }
        if self.pos > start {
            // Safe: input is valid UTF-8, we only break at ASCII delimiters,
            // so this byte range is a valid UTF-8 subsequence
            s.push_str(
                std::str::from_utf8(&self.input[start..self.pos]).unwrap(),
            );
        }
        match self.next_byte() {
            Some(b'"') => return Ok(s),
            Some(b'\\') => match self.next_byte() {
                Some(b'"') => s.push('"'),
                Some(b'\\') => s.push('\\'),
                Some(b'/') => s.push('/'),
                Some(b'n') => s.push('\n'),
                Some(b't') => s.push('\t'),
                Some(b'r') => s.push('\r'),
                Some(b'b') => s.push('\u{0008}'),
                Some(b'f') => s.push('\u{000C}'),
                Some(b'u') => s.push(self.hex_escape()?),
                Some(c) => return Err(self.err(format!(
                    "invalid escape '\\{}'", c as char,
                ))),
                None => return Err(self.err("unterminated escape")),
            },
            Some(b) => return Err(self.err(format!(
                "control character U+{:04X} in string", b,
            ))),
            None => return Err(self.err("unterminated string")),
        }
    }
}
```

The fast path deserves attention. Instead of processing one byte at a time, we scan forward until we hit a quote, backslash, or control character, then copy the entire range into the output string. This works correctly with multi-byte UTF-8 because we only break at ASCII bytes (< 0x80), and UTF-8 continuation bytes are always >= 0x80. So a byte range between two ASCII positions in valid UTF-8 is always a valid UTF-8 subsequence. The `unwrap` cannot panic - it's enforced by the input being a `&str`.

This is the same technique serde_json uses internally. For strings without escape sequences (the common case in API responses), the fast path copies the entire string content in one `push_str` call - no per-byte branching.

RFC 8259 forbids unescaped control characters (bytes 0x00-0x1F) inside strings. That's the `Some(b) => Err(...)` arm - if we encounter a control character that isn't handled by an escape sequence, it's invalid. This catches things like literal tab characters or newlines inside JSON strings.

### Unicode escapes and surrogate pairs

The `\uXXXX` escape is where JSON's history as a JavaScript subset shows. JSON encodes Unicode using UTF-16 code units, not codepoints. Characters in the Basic Multilingual Plane (U+0000 to U+FFFF) use a single `\uXXXX`. Characters above U+FFFF - emoji, CJK extensions, historic scripts - use a *surrogate pair*: two `\uXXXX` escapes back to back.

The crab emoji (U+1F980) in JSON is `\uD83E\uDD80`. That's high surrogate D83E followed by low surrogate DD80. Our parser must detect this and decode it:

```rust
fn hex_escape(&mut self) -> Result<char, ParseError> {
    let hi = self.hex4()?;
    let cp = if (0xD800..=0xDBFF).contains(&hi) {
        // High surrogate - expect \uXXXX low surrogate
        self.eat(b'\\')?;
        self.eat(b'u')?;
        let lo = self.hex4()?;
        if !(0xDC00..=0xDFFF).contains(&lo) {
            return Err(self.err("expected low surrogate"));
        }
        0x10000 + ((hi - 0xD800) << 10) + (lo - 0xDC00)
    } else {
        hi
    };
    char::from_u32(cp).ok_or_else(|| self.err("invalid codepoint"))
}

fn hex4(&mut self) -> Result<u32, ParseError> {
    let mut n = 0u32;
    for _ in 0..4 {
        let d = match self.next_byte() {
            Some(b) => (b as char).to_digit(16).ok_or_else(|| {
                self.err(format!("bad hex digit '{}'", b as char))
            })?,
            None => return Err(self.err("unterminated \\u escape")),
        };
        n = n * 16 + d;
    }
    Ok(n)
}
```

The surrogate pair formula is: `codepoint = 0x10000 + (high - 0xD800) * 0x400 + (low - 0xDC00)`. This comes directly from the UTF-16 encoding scheme. The high surrogate range (0xD800-0xDBFF) and low surrogate range (0xDC00-0xDFFF) are reserved specifically for this purpose - they're not valid Unicode codepoints on their own.

`char::from_u32` is our final safety net. Even after surrogate pair decoding, we verify that the resulting u32 is a valid Unicode scalar value. Rust's `char` type guarantees this - it rejects surrogate codepoints and values above U+10FFFF.

## Parsing numbers

JSON numbers follow a specific grammar: optional minus, integer part (no leading zeros except for `0` itself), optional fractional part, optional exponent. Our parser validates the structure and delegates the actual conversion to Rust's `f64::parse`:

```rust
fn number(&mut self) -> Result<JsonValue, ParseError> {
    let start = self.pos;
    if self.peek() == Some(b'-') {
        self.pos += 1;
    }
    match self.peek() {
        Some(b'0') => self.pos += 1,
        Some(b) if b.is_ascii_digit() => self.digits()?,
        _ => return Err(self.err("expected digit")),
    }
    if self.peek() == Some(b'.') {
        self.pos += 1;
        self.digits()?;
    }
    if matches!(self.peek(), Some(b'e' | b'E')) {
        self.pos += 1;
        if matches!(self.peek(), Some(b'+' | b'-')) {
            self.pos += 1;
        }
        self.digits()?;
    }
    let s = std::str::from_utf8(&self.input[start..self.pos]).unwrap();
    s.parse::<f64>()
        .map(JsonValue::Number)
        .map_err(|_| self.err(format!("invalid number: {}", s)))
}

fn digits(&mut self) -> Result<(), ParseError> {
    if !matches!(self.peek(), Some(b) if b.is_ascii_digit()) {
        return Err(self.err("expected digit"));
    }
    while matches!(self.peek(), Some(b) if b.is_ascii_digit()) {
        self.pos += 1;
    }
    Ok(())
}
```

The integer part has a subtle rule: `0` must stand alone. `0.5` is valid, `01` is not. That's the `Some(b'0') => self.pos += 1` branch - if the first digit is zero, we don't call `digits()` to consume more. Any digit immediately after a leading `0` (without a dot or exponent) would be caught as trailing content.

We're also not parsing the number ourselves - we let `f64::from_str` handle the actual string-to-float conversion. Writing a correct float parser is surprisingly hard (see [Eisel-Lemire algorithm](https://nigeltao.github.io/blog/2020/eisel-lemire.html)), and Rust's standard library already implements it correctly. Our job is just to validate the JSON grammar and extract the right byte range.

## Parsing literals, arrays, and objects

Booleans and null are trivial - just match the byte sequence:

```rust
fn boolean(&mut self) -> Result<JsonValue, ParseError> {
    if self.input[self.pos..].starts_with(b"true") {
        self.pos += 4;
        Ok(JsonValue::Bool(true))
    } else if self.input[self.pos..].starts_with(b"false") {
        self.pos += 5;
        Ok(JsonValue::Bool(false))
    } else {
        Err(self.err("expected 'true' or 'false'"))
    }
}

fn null(&mut self) -> Result<JsonValue, ParseError> {
    if self.input[self.pos..].starts_with(b"null") {
        self.pos += 4;
        Ok(JsonValue::Null)
    } else {
        Err(self.err("expected 'null'"))
    }
}
```

Arrays and objects are where recursion kicks in. Both follow the same pattern: opening bracket, comma-separated elements, closing bracket. The trick is handling the empty case (no elements) before entering the loop:

```rust
fn array(&mut self) -> Result<JsonValue, ParseError> {
    self.eat(b'[')?;
    self.skip_ws();
    let mut items = Vec::new();
    if self.peek() == Some(b']') {
        self.pos += 1;
        return Ok(JsonValue::Array(items));
    }
    loop {
        items.push(self.value()?);
        self.skip_ws();
        match self.peek() {
            Some(b',') => self.pos += 1,
            Some(b']') => {
                self.pos += 1;
                return Ok(JsonValue::Array(items));
            }
            _ => return Err(self.err("expected ',' or ']'")),
        }
    }
}

fn object(&mut self) -> Result<JsonValue, ParseError> {
    self.eat(b'{')?;
    self.skip_ws();
    let mut entries = Vec::new();
    if self.peek() == Some(b'}') {
        self.pos += 1;
        return Ok(JsonValue::Object(entries));
    }
    loop {
        self.skip_ws();
        let key = self.string()?;
        self.skip_ws();
        self.eat(b':')?;
        let val = self.value()?;
        entries.push((key, val));
        self.skip_ws();
        match self.peek() {
            Some(b',') => self.pos += 1,
            Some(b'}') => {
                self.pos += 1;
                return Ok(JsonValue::Object(entries));
            }
            _ => return Err(self.err("expected ',' or '}'")),
        }
    }
}
```

Notice how `object` calls `self.string()` for keys, then `self.value()` for values. And `self.value()` can call `self.object()` again for nested objects. This mutual recursion is what lets `{"a": {"b": [1, 2]}}` parse correctly - each function handles its own structural level and delegates downward.

The empty collection check (`peek() == Some(b']')` before the loop) prevents trying to parse a value from `[]` or `{}`. Without it, the parser would enter the loop, call `value()`, see `]` or `}`, and fail with "unexpected character" instead of the more helpful "expected value".

One thing this parser does NOT handle: trailing commas. `[1, 2, 3,]` is invalid JSON per RFC 8259, and our parser rejects it correctly. After consuming the comma, it loops back to `self.value()`, which sees `]` and reports "unexpected ']'". serde_json has a feature flag for accepting trailing commas, but strict RFC compliance means rejecting them.

## The entry point

```rust
pub fn parse(input: &str) -> Result<JsonValue, ParseError> {
    let mut p = Parser::new(input);
    let val = p.value()?;
    p.skip_ws();
    if p.pos != p.input.len() {
        return Err(p.err("trailing content after JSON value"));
    }
    Ok(val)
}
```

The trailing content check is important. Without it, `{"a": 1} garbage` would parse successfully, returning the object and silently ignoring everything after it. RFC 8259 defines `JSON-text = ws value ws` - a single value with optional surrounding whitespace, nothing more.

## Running it

```rust
fn main() {
    let input = r#"{
  "name": "ferris",
  "age": 9,
  "mass_kg": null,
  "is_crab": true,
  "languages": ["Rust", "C"],
  "metadata": {
    "emoji": "\uD83E\uDD80",
    "website": "https://rustacean.net"
  }
}"#;

    match parse(input) {
        Ok(val) => println!("{:#?}", val),
        Err(e) => eprintln!("error: {}", e),
    }
}
```

Output:

```
Object([
    ("name", Str("ferris")),
    ("age", Number(9.0)),
    ("mass_kg", Null),
    ("is_crab", Bool(true)),
    ("languages", Array([Str("Rust"), Str("C")])),
    ("metadata", Object([
        ("emoji", Str("🦀")),
        ("website", Str("https://rustacean.net")),
    ])),
])
```

The surrogate pair `\uD83E\uDD80` decoded into the actual crab emoji. The parser consumed the entire input, validated every structural element, handled the nested object and array, and produced a typed Rust value we can pattern-match on.

## Error messages in practice

Good error messages are the difference between a parser that's useful and one that makes you want to `hexdump` the input. Here are some examples:

```rust
parse(r#"{"key": tru}"#);
// 1:9: expected 'true' or 'false'

parse(r#"{"key" "value"}"#);
// 1:8: expected ':', found '"'

parse(r#"{"name": "unterminated}"#);
// 1:24: unterminated string

parse("{\n  \"count\": 01\n}");
// 2:14: expected ',' or '}'
```

That last error - `01` being rejected - comes from the number parser not consuming the `1` after leading `0`, which then looks like trailing content inside the object. The error message isn't perfect (it says "expected ',' or '}'" rather than "leading zeros not allowed"), but it points to the right location. Improving error messages is an infinite rabbit hole - you can always make them better.

## What serde_json does differently

Our parser works, but production JSON parsing involves a lot more than structural correctness. Here's what [serde_json](https://github.com/serde-rs/json) brings to the table:

**Type-driven deserialization.** serde_json doesn't just parse into a generic value type - it can parse directly into your Rust structs. When you write `serde_json::from_str::<User>(input)`, the parser knows it's looking for a string field called "name" and an integer field called "age". This means it can skip values it doesn't need, validate types as it parses, and report errors in terms of your domain types ("expected string for field `email`" rather than just "expected string").

**Zero-copy string references.** Our parser allocates a new `String` for every JSON string. serde_json can borrow string values directly from the input when no escape processing is needed. If your JSON has `"hello"`, serde returns a `&str` pointing into the original input buffer - no allocation, no copy. This is behind the `#[serde(borrow)]` attribute and `Cow<'a, str>` types.

**Number precision.** We use `f64`, which can't represent integers above 2^53 exactly. serde_json's `Number` type preserves the original string representation and can be converted to `u64`, `i64`, or `f64` depending on what the caller needs. If you're parsing a Snowflake ID (`1285938573456384000`), f64 would silently corrupt it.

**Streaming parsing.** For large JSON files, loading everything into memory as a `JsonValue` tree isn't practical. serde_json provides `StreamDeserializer` that yields values one at a time from a byte stream. Our parser requires the entire input in memory.

**Performance.** serde_json uses `memchr` (which leverages SIMD on x86_64) to scan for string delimiters, processes strings in bulk rather than byte-by-byte in most code paths, and carefully avoids allocations. On typical JSON payloads, serde_json parses at several hundred megabytes per second. For even more speed, [simd-json](https://crates.io/crates/simd-json) implements the [simdjson](https://github.com/simdjson/simdjson) algorithm in Rust, processing 16-32 bytes at a time using SIMD instructions to classify character types in parallel. It can hit multiple gigabytes per second on modern CPUs.

## What you learn from building this

The whole parser is about 200 lines. If you've followed along, here's what you should take away:

**Recursive descent falls naturally out of recursive grammars.** JSON's grammar is recursive (`value` contains `object` which contains `value`), and our code mirrors that structure. Each grammar rule becomes a function. Each function returns `Result<T, ParseError>`. The composition is just function calls. This is the core of parser combinators without the abstraction layer.

**Operating on bytes is faster than operating on chars.** We work on `&[u8]` instead of iterating over `char`. JSON's structural characters are all ASCII, so byte comparison is safe and avoids the overhead of UTF-8 decoding on every character. We only deal with multi-byte UTF-8 in the string fast path, where we copy byte ranges directly.

**The grammar drives the code.** Look at the RFC's ABNF for `number`: `[ "-" ] int [ frac ] [ exp ]`. Now look at our `number()` function: check for minus, parse int part, optionally parse fraction, optionally parse exponent. The code is a transliteration of the grammar. This is the fundamental insight of recursive descent - if your grammar is clean, your parser writes itself.

**Error reporting costs almost nothing when done lazily.** We never compute line/column numbers during successful parsing. The `line_col()` function only runs when something goes wrong. This means the happy path (valid JSON) pays zero overhead for error reporting.

If you want to push this further, try adding a `Display` implementation that turns a `JsonValue` back into formatted JSON (pretty-printing with indentation). Or try parsing JSON from a `Read` stream instead of a `&str` - you'll discover why streaming parsers are harder (you can't peek ahead without buffering, and `line_col()` can't scan backwards through bytes you've already consumed). Or try writing a JSON Schema validator that walks the `JsonValue` tree - you'll appreciate having a proper AST instead of a stream of tokens.

The full parser fits in a single file. Grab it, run it against your API responses, throw malformed JSON at it and see what errors it produces. That's how you learn what `serde_json::from_str` actually does under the hood.
