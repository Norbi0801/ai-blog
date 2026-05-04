+++
title = "Understanding UTF-8 - why string handling is hard"
date = 2026-04-09
description = "UTF-8 is a variable-length encoding where one character can be 1 to 4 bytes - and what you think of as a character might not match what the computer thinks."

[taxonomies]
tags = ["rust", "strings", "unicode", "encoding"]
+++

ASCII mapped every character to one byte. 65 is `A`, 48 is `0`, 10 is newline. One byte, one character, always. You could index into a string with a number, get the nth character, reverse a string by reversing the bytes, compare strings by comparing bytes. String handling was trivial because the encoding was trivial.

Then the world needed more than 128 characters.

Unicode assigned a number (called a "code point") to every character from every writing system - Latin, Cyrillic, CJK, Arabic, emoji, mathematical symbols, musical notation, 154,998 characters and counting as of [Unicode 16.0](https://www.unicode.org/versions/Unicode16.0.0/). The question was: how do you represent those numbers in memory?

UTF-32 stores every code point as 4 bytes. Simple, but wasteful - English text uses 4x the memory it needs. UTF-16 uses 2 or 4 bytes per code point - Windows and Java went this route. UTF-8 uses 1 to 4 bytes per code point, and it's backwards compatible with ASCII. The web picked UTF-8 and never looked back. [Over 98% of web pages](https://w3techs.com/technologies/details/en-utf8) use it today.

UTF-8 won because it's efficient for the common case and compatible with existing ASCII infrastructure. But that efficiency comes from variable-length encoding, and variable-length encoding is what makes string handling hard.

If you already know [why Rust has so many string types](/blog/why-rust-has-so-many-string-types/) and how `String` is a `Vec<u8>` with a UTF-8 invariant, this post goes one layer deeper: into the encoding itself, the layers of abstraction between bytes and "characters", and the pitfalls that catch even experienced developers.

<!-- more -->

## How UTF-8 encodes code points

UTF-8 is a variable-length encoding. Each Unicode code point becomes 1, 2, 3, or 4 bytes depending on its value:

| Code point range | Bytes | Bit pattern | Example |
|---|---|---|---|
| U+0000 - U+007F | 1 | `0xxxxxxx` | `A` = 0x41 |
| U+0080 - U+07FF | 2 | `110xxxxx 10xxxxxx` | `ñ` = 0xC3 0xB1 |
| U+0800 - U+FFFF | 3 | `1110xxxx 10xxxxxx 10xxxxxx` | `€` = 0xE2 0x82 0xAC |
| U+10000 - U+10FFFF | 4 | `11110xxx 10xxxxxx 10xxxxxx 10xxxxxx` | `😀` = 0xF0 0x9F 0x98 0x80 |

The first byte tells the decoder how many bytes to read. If it starts with `0`, it's a single-byte ASCII character. If it starts with `110`, read 2 bytes. `1110` means 3 bytes. `11110` means 4. Continuation bytes always start with `10`, so a decoder can resynchronize after corruption by scanning for a byte that doesn't start with `10`.

This design is elegant, but it destroys a fundamental assumption: byte position no longer equals character position. The string `"hello"` is 5 bytes and 5 characters. The string `"héllo"` is 6 bytes but still 5 characters, because `é` (U+00E9) takes 2 bytes. The string `"😀😀😀"` is 12 bytes but 3 characters.

Let's verify in Rust:

```rust
fn main() {
    let ascii = "hello";
    let french = "héllo";
    let emoji = "😀😀😀";

    println!("{}: {} bytes, {} chars", ascii, ascii.len(), ascii.chars().count());
    println!("{}: {} bytes, {} chars", french, french.len(), french.chars().count());
    println!("{}: {} bytes, {} chars", emoji, emoji.len(), emoji.chars().count());

    // Inspect the actual bytes
    println!("bytes of 'é': {:02X?}", "é".as_bytes());
    // [C3, A9] - 2 bytes, matching the 110xxxxx 10xxxxxx pattern
    
    println!("bytes of '😀': {:02X?}", "😀".as_bytes());
    // [F0, 9F, 98, 80] - 4 bytes, matching 11110xxx 10xxxxxx 10xxxxxx 10xxxxxx
}
```

Output:

```
hello: 5 bytes, 5 chars
héllo: 6 bytes, 5 chars
😀😀😀: 12 bytes, 3 chars
bytes of 'é': [C3, A9]
bytes of '😀': [F0, 9F, 98, 80]
```

## Bytes vs code points vs grapheme clusters

Here's where it gets worse. "Character" isn't a single concept in Unicode. There are three layers, and they don't align:

**Bytes** - the raw `u8` values in memory. What `.len()` returns. What `&str` is indexed by.

**Code points (scalar values)** - the Unicode numbers. What `.chars()` iterates over. The `char` type in Rust represents one Unicode scalar value (all code points except surrogate pairs U+D800 through U+DFFF).

**Grapheme clusters** - what a human perceives as "one character". This is what you see on screen.

These three layers diverge in practice. Take the string `"café"`. You might expect 4 characters. But Unicode has two ways to represent `é`:

1. **Precomposed**: U+00E9 (LATIN SMALL LETTER E WITH ACUTE) - a single code point, 2 bytes in UTF-8
2. **Decomposed**: U+0065 (LATIN SMALL LETTER E) + U+0301 (COMBINING ACUTE ACCENT) - two code points, 3 bytes in UTF-8

Both render identically. Both are valid. But they have different byte lengths, different code point counts, and different results when you iterate with `.chars()`:

```rust
fn main() {
    let composed = "caf\u{00E9}";   // precomposed é
    let decomposed = "cafe\u{0301}"; // e + combining accent

    println!("composed:   '{}' = {} bytes, {} chars", composed, composed.len(), composed.chars().count());
    println!("decomposed: '{}' = {} bytes, {} chars", decomposed, decomposed.len(), decomposed.chars().count());
    
    // They look the same but...
    println!("equal? {}", composed == decomposed); // false!
    
    // The chars are different
    println!("composed chars:   {:?}", composed.chars().collect::<Vec<_>>());
    println!("decomposed chars: {:?}", decomposed.chars().collect::<Vec<_>>());
}
```

Output:

```
composed:   'café' = 5 bytes, 4 chars
decomposed: 'café' = 6 bytes, 5 chars
equal? false
composed chars:   ['c', 'a', 'f', 'é']
decomposed chars: ['c', 'a', 'f', 'e', '\u{301}']
```

Same visual string. Different bytes. Different code point count. Byte comparison says they're not equal. This is the fundamental source of string handling bugs across every programming language.

And emoji make it even more extreme. The "family" emoji 👨‍👩‍👧‍👦 is a single grapheme cluster made of seven code points glued together with zero-width joiners (U+200D):

```rust
fn main() {
    let family = "👨\u{200D}👩\u{200D}👧\u{200D}👦";

    println!("'{}': {} bytes, {} code points",
        family,
        family.len(),
        family.chars().count()
    );
    
    // Break it down
    for (i, ch) in family.chars().enumerate() {
        println!("  char {}: U+{:04X} ({} bytes in UTF-8)", i, ch as u32, ch.len_utf8());
    }
}
```

Output:

```
'👨‍👩‍👧‍👦': 25 bytes, 7 code points
  char 0: U+1F468 (4 bytes in UTF-8)
  char 1: U+200D (3 bytes in UTF-8)
  char 2: U+1F469 (4 bytes in UTF-8)
  char 3: U+200D (3 bytes in UTF-8)
  char 4: U+1F467 (4 bytes in UTF-8)
  char 5: U+200D (3 bytes in UTF-8)
  char 6: U+1F466 (4 bytes in UTF-8)
```

One visible character. 7 code points. 25 bytes. A function that reverses a string "character by character" using `.chars()` would tear this emoji apart, turning a family into a backwards sequence of individual people and invisible joiners.

Country flags are similar: 🇪🇺 is two regional indicator code points (U+1F1EA + U+1F1FA), 8 bytes total, rendered as a single flag glyph.

## Why &str[0..3] can panic

Rust's `str` type is indexed by bytes, not characters. When you write `&s[0..3]`, you're asking for the first 3 bytes. If byte 3 falls in the middle of a multi-byte character, Rust panics at runtime:

```rust
fn main() {
    let s = "héllo";
    // 'h' = 1 byte, 'é' = 2 bytes, 'l' = 1 byte ...
    // byte layout: [68, C3, A9, 6C, 6C, 6F]
    //               h   é(1) é(2) l    l    o

    let ok = &s[0..1];   // "h" - valid, byte 0-1 are both char boundaries
    println!("ok: {}", ok);

    let also_ok = &s[0..3]; // "hé" - valid, byte 3 is the start of 'l'
    println!("also_ok: {}", also_ok);

    // This panics:
    // let bad = &s[0..2]; // byte 2 is 0xA9, the second byte of 'é'
    // "byte index 2 is not a char boundary; it is inside 'é' (bytes 1..3)"
}
```

The panic message is clear: `byte index 2 is not a char boundary; it is inside 'é' (bytes 1..3)`. Rust won't silently give you broken UTF-8. It crashes instead.

This is a deliberate design choice. Rust guarantees that every `&str` is valid UTF-8. If slicing could produce invalid UTF-8, the guarantee would be worthless. So Rust checks at runtime and panics if you slice at a non-boundary.

The non-panicking alternative is `.get()`:

```rust
fn main() {
    let s = "héllo";

    match s.get(0..2) {
        Some(slice) => println!("got: {}", slice),
        None => println!("not a valid char boundary"),
    }

    // For safe character-aware iteration:
    for (byte_idx, ch) in s.char_indices() {
        println!("byte {}: '{}'", byte_idx, ch);
    }
}
```

Output:

```
not a valid char boundary
byte 0: 'h'
byte 1: 'é'
byte 3: 'l'
byte 4: 'l'
byte 5: 'o'
```

Notice the gap: byte index 2 doesn't appear because it's a continuation byte inside `é`. `.char_indices()` gives you the byte offset of each code point's start, which is the safe way to find slicing boundaries.

## String reversal: the classic gotcha

Reversing a string sounds trivial. In a byte-oriented language, reverse the bytes. In a code-point-oriented language, reverse the code points. Both are wrong.

```rust
fn main() {
    let s = "cafe\u{0301}"; // e + combining accent = é

    // Reverse by code points - WRONG
    let reversed_chars: String = s.chars().rev().collect();
    println!("original:  '{}'", s);
    println!("reversed:  '{}'", reversed_chars);
    // The combining accent now attaches to the previous character!
    // Expected: "éfac"
    // Got:      "´efac" (accent floats to 'e' on the other side, or attaches to nothing)
}
```

When you reverse the code points, the combining accent (U+0301) that was after `e` ends up before `e` in the reversed string - meaning it now applies to whatever character precedes it, or to nothing. The visual result depends on the rendering engine, but it's wrong either way.

Correct string reversal requires reversing grapheme clusters, not code points.

## Unicode normalization

If the same visible string can have multiple byte representations, how do you compare them? Normalization - converting strings to a canonical form before comparison.

[Unicode Standard Annex #15](https://unicode.org/reports/tr15/) defines four normalization forms:

**NFC (Canonical Decomposition, followed by Canonical Composition)** - decomposes characters, then recomposes them into precomposed form where possible. `e` + combining accent becomes `é` (U+00E9). This is what most systems use. Over 99% of text on the web is already in NFC.

**NFD (Canonical Decomposition)** - decomposes characters into base characters + combining marks. `é` (U+00E9) becomes `e` + combining accent. macOS normalizes filenames to NFD, which is a source of cross-platform bugs.

**NFKC (Compatibility Decomposition, followed by Canonical Composition)** - like NFC, but also replaces compatibility characters. The "fi" ligature (U+FB01) becomes two separate characters `f` and `i`. Useful for search, but lossy - you can't round-trip.

**NFKD (Compatibility Decomposition)** - like NFD, but with compatibility decomposition.

The Rust standard library doesn't include normalization. You need the [`unicode-normalization`](https://crates.io/crates/unicode-normalization) crate (v0.1.25, implementing Unicode 16.0 tables):

```rust
use unicode_normalization::UnicodeNormalization;

fn main() {
    let composed = "caf\u{00E9}";       // NFC: precomposed é
    let decomposed = "cafe\u{0301}";     // NFD: e + combining accent

    // Direct comparison fails
    assert_ne!(composed, decomposed);

    // Normalize both to NFC, then compare
    let a: String = composed.nfc().collect();
    let b: String = decomposed.nfc().collect();
    assert_eq!(a, b); // now equal

    // Or normalize both to NFD
    let c: String = composed.nfd().collect();
    let d: String = decomposed.nfd().collect();
    assert_eq!(c, d); // also equal

    println!("NFC bytes: {:02X?}", a.as_bytes());
    println!("NFD bytes: {:02X?}", c.as_bytes());
}
```

Output:

```
NFC bytes: [63, 61, 66, C3, A9]
NFD bytes: [63, 61, 66, 65, CC, 81]
```

The NFC form is 5 bytes (precomposed `é` is `C3 A9`). The NFD form is 6 bytes (separate `e` at `65` plus combining accent at `CC 81`). Same string, same visual, different bytes, different lengths.

### The macOS filename problem

macOS's HFS+ and APFS filesystems store filenames in NFD (decomposed) form. Linux filesystems store filenames as raw bytes - whatever you gave them. This means:

1. You create a file called `café.txt` on Linux. The filename is 9 bytes (NFC).
2. You copy it to macOS. macOS decomposes it to NFD. Now it's 10 bytes.
3. You copy it back to Linux. The filename is 10 bytes.
4. `"café.txt" != "café.txt"` at the byte level, even though they look identical.

This has caused real data loss. The Netatalk and Samba file servers [hit this exact bug](https://en.wikipedia.org/wiki/Unicode_equivalence#Errors_due_to_normalization_differences) - they normalized filenames differently, and files became unfindable.

The fix is to normalize before comparing. Always. If you're building anything that deals with user-provided filenames or identifiers across platforms, normalize to NFC first.

## String comparison pitfalls

Beyond normalization, Unicode has other comparison traps.

### Case folding is locale-dependent

In English, uppercase `I` maps to lowercase `i`. In Turkish, uppercase `I` maps to lowercase `ı` (dotless i), and uppercase `İ` (dotted I) maps to lowercase `i`. The `to_lowercase()` function in most languages uses a fixed mapping. This means a case-insensitive comparison of `"I"` gives different results depending on whether you're applying English or Turkish rules:

```rust
fn main() {
    let upper = "I";
    let lower_english = "i";
    let lower_turkish = "ı"; // U+0131, LATIN SMALL LETTER DOTLESS I

    // Rust's to_lowercase uses Unicode default (non-locale-specific)
    println!("'I'.to_lowercase() = '{}'", upper.to_lowercase());
    // Prints: 'I'.to_lowercase() = 'i'
    // This is correct for English, wrong for Turkish
    
    // For locale-aware case folding, you need the `icu` crate
    println!("'i' == 'ı'? {}", lower_english == lower_turkish); // false
}
```

### Characters that look identical but aren't

Unicode has multiple code points that render as visually identical characters:

```rust
fn main() {
    let latin_a = 'A';           // U+0041, LATIN CAPITAL LETTER A
    let cyrillic_a = 'А';       // U+0410, CYRILLIC CAPITAL LETTER A
    let greek_alpha = 'Α';      // U+0391, GREEK CAPITAL LETTER ALPHA

    // All three look like "A" but are different code points
    println!("Latin A:    U+{:04X}", latin_a as u32);
    println!("Cyrillic A: U+{:04X}", cyrillic_a as u32);
    println!("Greek A:    U+{:04X}", greek_alpha as u32);

    println!("Latin == Cyrillic? {}", latin_a == cyrillic_a);   // false
    println!("Latin == Greek?    {}", latin_a == greek_alpha);   // false

    // This enables homograph attacks:
    // "apple.com" vs "аpple.com" (Cyrillic 'а' in the second one)
    // They look identical but resolve to different domains
    let legit = "apple.com";
    let fake = "\u{0430}pple.com"; // starts with Cyrillic а
    println!("'{}' == '{}'? {}", legit, fake, legit == fake); // false
}
```

This is the basis of [homograph attacks](https://en.wikipedia.org/wiki/IDN_homograph_attack) - registering domain names using visually identical characters from different scripts. NFKC normalization doesn't catch this because they're legitimately different characters, not compatibility equivalents.

### Sorting is not byte sorting

Alphabetical ordering depends on the language. In German, `ä` sorts with `a`. In Swedish, `ä` is a separate letter after `z`. Byte-level sorting puts `ä` (0xC3 0xA4) after all ASCII characters, which is wrong for both languages. Proper locale-aware sorting needs the [Unicode Collation Algorithm](https://www.unicode.org/reports/tr10/), which the standard library doesn't implement. For Rust, the [`icu_collator`](https://crates.io/crates/icu_collator) crate from the ICU4X project handles this.

## How Rust handles this

Rust's approach to strings is conservative and correct. Some design decisions that fall directly out of the UTF-8 complexity:

**`str` is always valid UTF-8.** Every `&str` in safe Rust is guaranteed to contain valid UTF-8 bytes. This is enforced at creation time - `String::from_utf8()` returns `Result`, `str::from_utf8()` validates the bytes. You can trust that any `&str` you receive is well-formed.

**No indexing by integer.** `s[0]` doesn't compile. This forces you to think about what you want - byte? Code point? Grapheme cluster? - instead of getting the wrong thing silently.

```rust
fn main() {
    let s = String::from("héllo");
    
    // s[0]; // ERROR: the type `str` cannot be indexed by `usize`
    
    // You have to be explicit about what you want:
    let first_byte: u8 = s.as_bytes()[0];             // byte access
    let first_char: char = s.chars().next().unwrap();  // code point access
    // grapheme access needs the unicode-segmentation crate
    
    println!("first byte: 0x{:02X}", first_byte);  // 0x68
    println!("first char: '{}'", first_char);       // 'h'
}
```

**`.len()` returns bytes, `.chars().count()` returns code points.** Neither returns what the user thinks of as "characters" (grapheme clusters). The naming is honest - `len` is the byte length, which is O(1). Counting characters requires iterating, which is O(n).

**`char` is a Unicode scalar value, not a grapheme cluster.** A `char` in Rust is always 4 bytes (a `u32` internally) and represents exactly one Unicode scalar value. A single visible character like 👨‍👩‍👧‍👦 is 7 `char`s. This surprises people coming from languages where `char` means "one thing you see on screen."

## The unicode-segmentation crate

For grapheme cluster-aware string handling, you need the [`unicode-segmentation`](https://crates.io/crates/unicode-segmentation) crate. It implements [Unicode Standard Annex #29](https://unicode.org/reports/tr29/) for grapheme cluster boundaries:

```toml
[dependencies]
unicode-segmentation = "1.12"
```

```rust
use unicode_segmentation::UnicodeSegmentation;

fn main() {
    let family = "👨\u{200D}👩\u{200D}👧\u{200D}👦";

    println!("bytes:    {}", family.len());
    println!("chars:    {}", family.chars().count());
    println!("graphemes: {}", family.graphemes(true).count());

    // Correct string reversal
    let s = "cafe\u{0301}"; // e + combining accent
    let reversed: String = s.graphemes(true).rev().collect();
    println!("original: '{}'", s);
    println!("reversed: '{}'", reversed);

    // Count what humans think of as characters
    let text = "Hello 🇪🇺!";
    for g in text.graphemes(true) {
        println!("grapheme: '{}' ({} bytes)", g, g.len());
    }
}
```

Output:

```
bytes:    25
chars:    7
graphemes: 1
original: 'café'
reversed: 'éfac'
grapheme: 'H' (1 bytes)
grapheme: 'e' (1 bytes)
grapheme: 'l' (1 bytes)
grapheme: 'l' (1 bytes)
grapheme: 'o' (1 bytes)
grapheme: ' ' (1 bytes)
grapheme: '🇪🇺' (8 bytes)
grapheme: '!' (1 bytes)
```

The `true` argument to `.graphemes()` selects extended grapheme cluster boundaries (UAX#29), which is what you want for modern text. The alternative `false` gives legacy grapheme clusters, which mishandle some emoji sequences.

### Building correct string functions

With `unicode-segmentation`, you can build string functions that work on what humans perceive as characters:

```rust
use unicode_segmentation::UnicodeSegmentation;

fn grapheme_len(s: &str) -> usize {
    s.graphemes(true).count()
}

fn truncate_graphemes(s: &str, max: usize) -> &str {
    let mut end = 0;
    for (i, grapheme) in s.grapheme_indices(true).take(max) {
        end = i + grapheme.len();
    }
    &s[..end]
}

fn main() {
    let text = "Hello 👨\u{200D}👩\u{200D}👧\u{200D}👦 world";
    println!("grapheme length: {}", grapheme_len(text));
    println!("truncated to 8: '{}'", truncate_graphemes(text, 8));
}
```

Output:

```
grapheme length: 13
truncated to 8: 'Hello 👨‍👩‍👧‍👦 w'
```

Compare this to naive truncation with `.chars().take(8)`, which would cut the family emoji apart.

## Practical rules

After all this, here's what to keep in mind when working with strings in Rust:

**Use `.len()` for buffer sizes, not "string length."** When you need to allocate a buffer or check if a string fits in a fixed-size field, `.len()` gives you the byte count. That's what matters for memory.

**Use `.chars().count()` only when you actually need code point count.** This is O(n) and usually not what you want. It's wrong for user-visible length and wrong for byte length. It's correct for things like validating "no more than N Unicode scalar values."

**Use `unicode-segmentation` for user-facing length.** If you're implementing a text input with a max character limit, or truncating a string for display, or counting "characters" the way a user would - use grapheme clusters.

**Normalize before comparing user input.** If your system stores usernames, filenames, or any identifier that a user types, normalize to NFC before storage and comparison. Two users typing `café` on different keyboards might produce NFC or NFD, and you need them to match.

**Don't reverse strings unless you have a reason.** And if you do, reverse grapheme clusters. The function `s.chars().rev().collect::<String>()` is wrong for any string containing combining marks or emoji sequences.

**Prefer `.get()` over index ranges.** `&s[a..b]` panics if `a` or `b` aren't char boundaries. `s.get(a..b)` returns `None`. If there's any chance your indices come from external input or arithmetic, use `.get()`.

**Use `char_indices()` when you need byte positions.** If you're scanning a string for a pattern and need to slice it, `.char_indices()` gives you `(byte_offset, char)` pairs. The byte offsets are guaranteed to be valid slice boundaries.

String handling is hard because human text is hard. ASCII worked because it ignored most of the world. Unicode includes the whole world, and the price of inclusion is complexity. Rust doesn't hide that complexity behind convenient lies - it puts it in the type system and forces you to deal with it. That's uncomfortable, but it's honest. And it's why Rust programs tend to handle international text correctly where programs in other languages silently break.
