+++
title = "Why Rust has so many string types"
date = 2025-01-09
description = "String, &str, OsStr, CStr, Path - what each one actually is under the hood, when to reach for which, and how to convert between them without losing your mind."

[taxonomies]
tags = ["rust", "strings", "ffi", "memory"]
+++

Newcomers hit this wall fast. You want to pass a filename to a function and suddenly you're choosing between `String`, `&str`, `&Path`, `&OsStr`, and `PathBuf`. You try `.to_string()` on everything until it compiles. It works, but you've introduced unnecessary allocations, and the code reads like you fought the type system and won by exhaustion.

Rust doesn't have five string types because someone on the lang team was bored. Each type exists because it encodes a different invariant in the type system. `String` guarantees valid UTF-8. `OsStr` doesn't - because the OS doesn't. `CStr` guarantees a null terminator and no interior nulls - because C demands it. `Path` is a thin wrapper over `OsStr` that adds path-manipulation methods. Conflating these would mean either losing safety guarantees or silently corrupting data. Rust chose explicit types instead.

Let's look at what each one actually is under the hood, how they relate to each other, and when to use which.

<!-- more -->

## The owned/borrowed duality

Before anything else, notice the pattern. Every string type comes in pairs:

| Owned (heap, growable) | Borrowed (reference, read-only) |
|---|---|
| `String` | `&str` |
| `OsString` | `&OsStr` |
| `CString` | `&CStr` |
| `PathBuf` | `&Path` |

The relationship is always the same: the owned type contains the data (usually backed by a `Vec` of some kind), and the borrowed type is a reference into it. The owned type implements `Deref` to the borrowed type, so you can pass a `&String` where `&str` is expected, a `&PathBuf` where `&Path` is expected, etc.

This mirrors the `Vec<T>` / `&[T]` relationship. If you understand slices, you understand Rust strings.

## String and &str - the UTF-8 pair

`String` is Rust's primary owned string type. Under the hood, it's a newtype around `Vec<u8>`:

```rust
// From the standard library (library/alloc/src/string.rs)
pub struct String {
    vec: Vec<u8>,
}
```

That's it. A `String` is a `Vec<u8>` that maintains one extra invariant: the bytes are always valid [UTF-8](https://en.wikipedia.org/wiki/UTF-8). Every method on `String` that modifies the buffer - `push`, `push_str`, `insert`, `replace_range` - checks or enforces this invariant. You can't accidentally shove invalid bytes into a `String` through safe code.

Because it's backed by a `Vec<u8>`, `String` has the same memory layout as any `Vec` - three `usize` fields on the stack:

```
String on a 64-bit system (24 bytes on the stack):
+----------+----------+----------+
| pointer  |  length  | capacity |
| (8 bytes)| (8 bytes)| (8 bytes)|
+----------+----------+----------+
     |
     v
  [heap: UTF-8 bytes...]
```

If you already read the [flyweight pattern post](/blog/the-flyweight-pattern-sharing-data-efficiently-in-rust/), you've seen this layout. Capacity is separate from length because `String` can pre-allocate space for future growth, just like `Vec`.

`&str` - a string slice - is a fat pointer: a pointer to UTF-8 bytes and a length. No capacity, because you can't grow a slice.

```
&str (16 bytes on the stack):
+----------+----------+
| pointer  |  length  |
| (8 bytes)| (8 bytes)|
+----------+----------+
     |
     v
  [borrowed UTF-8 bytes...]
```

The borrowed bytes can live anywhere - on the heap (when slicing a `String`), in static memory (string literals), or even on the stack (from a fixed-size buffer). `&str` doesn't care where the data lives. It just needs a pointer and a length.

String literals are `&'static str` - they're baked into the binary's read-only data segment at compile time:

```rust
fn main() {
    let s: &'static str = "hello";
    // "hello" lives in the .rodata section of the binary
    // s is a fat pointer: (address_in_rodata, 5)
    
    let owned = String::from("hello");
    // "hello" bytes are copied to the heap
    // owned is: (heap_ptr, 5, 5)
    
    let borrowed: &str = &owned;
    // borrowed points into owned's heap buffer
    // borrowed is: (heap_ptr, 5)
}
```

You can verify this with `std::mem::size_of`:

```rust
use std::mem;

fn main() {
    assert_eq!(mem::size_of::<String>(), 24);
    assert_eq!(mem::size_of::<&str>(), 16);
    assert_eq!(mem::size_of::<&String>(), 8); // thin pointer - String is Sized
}
```

### The key invariant: UTF-8

Both `String` and `&str` guarantee valid UTF-8 at all times. This means:

- Indexing by byte position (`s[0]`) is not allowed (it might split a multi-byte character)
- `.len()` returns byte count, not character count
- `.chars()` iterates over Unicode scalar values, decoding UTF-8 on the fly

```rust
fn main() {
    let s = "cafe\u{0301}"; // "cafe" + combining acute accent = "cafe\u{0301}"
    println!("bytes: {}", s.len());         // 6 (e\u{0301} is 2 bytes for the accent)
    println!("chars: {}", s.chars().count()); // 5 (c, a, f, e, combining accent)

    // Grapheme clusters need the `unicode-segmentation` crate
    // "cafe\u{0301}" is 4 graphemes visually: c, a, f, e-with-accent
}
```

This UTF-8 guarantee is non-negotiable. That's why `String::from_utf8(bytes)` returns a `Result` - it validates the bytes. And that's exactly why the other string types exist: sometimes your data isn't UTF-8, and pretending otherwise would be a lie.

## OsStr and OsString - what the OS actually gives you

File names. Environment variables. Command-line arguments. These come from the operating system, and the OS doesn't speak UTF-8 - at least not on every platform.

On Linux and macOS, file paths are arbitrary byte sequences (with the only constraint being no null bytes and no `/` within a path component). Most files have UTF-8 names in practice, but nothing enforces it. You can create a file named `\xff\xfe` on Linux and it's perfectly legal. Try to store that in a `String` and you'll get an error.

On Windows, file paths are sequences of 16-bit code units - nominally UTF-16, but Windows allows unpaired surrogates, which aren't valid Unicode. A file name can contain the lone surrogate `0xD800` and Windows won't complain.

`OsStr` and `OsString` handle both cases. They're defined in [`std::ffi`](https://doc.rust-lang.org/std/ffi/struct.OsStr.html) and represent the platform's native string encoding:

- **On Unix**: `OsStr` is a wrapper around `[u8]`. Direct byte access is available through the `std::os::unix::ffi::OsStrExt` trait.
- **On Windows**: `OsStr` internally uses [WTF-8](https://simonsapin.github.io/wtf-8/) - an encoding that's a superset of UTF-8 and can represent unpaired surrogates. This lets Rust round-trip Windows strings losslessly without using UTF-16 internally.

The internal representation is intentionally opaque:

```rust
// Simplified from the standard library
pub struct OsString {
    inner: platform_specific::Buf, // you can't access this
}

pub struct OsStr {
    inner: platform_specific::Slice, // opaque
}
```

### When OsStr matters in practice

Any function that deals with the filesystem should accept `&OsStr` or `AsRef<OsStr>`, not `&str`. The standard library already does this - `std::fs::read_to_string` takes `AsRef<Path>`, which wraps `OsStr`.

But the moment you want to display, log, or serialize a path, you need a `String`. And that conversion can fail:

```rust
use std::ffi::OsStr;
use std::path::Path;

fn print_filename(path: &Path) {
    match path.file_name() {
        Some(name) => match name.to_str() {
            Some(utf8) => println!("File: {}", utf8),
            None => println!("File: {} (lossy)", name.to_string_lossy()),
        },
        None => println!("No filename"),
    }
}
```

`to_str()` returns `Option<&str>` - it's `None` when the OS string isn't valid UTF-8. `to_string_lossy()` returns `Cow<'_, str>` - it borrows if the data is valid UTF-8, or allocates a new `String` with replacement characters (`U+FFFD`) for invalid sequences.

```rust
use std::ffi::OsStr;

fn main() {
    let valid = OsStr::new("hello.txt");
    assert_eq!(valid.to_str(), Some("hello.txt")); // valid UTF-8

    // On Unix, you can create OsStr from arbitrary bytes:
    #[cfg(unix)]
    {
        use std::os::unix::ffi::OsStrExt;
        let invalid = OsStr::from_bytes(&[0xff, 0xfe, 0x2e, 0x74, 0x78, 0x74]);
        assert_eq!(invalid.to_str(), None); // not valid UTF-8
        // to_string_lossy replaces invalid bytes with U+FFFD
        println!("{}", invalid.to_string_lossy()); // "##.txt" (with replacement chars)
    }
}
```

### Platform-specific byte access

On Unix, you can get raw bytes:

```rust
#[cfg(unix)]
fn path_bytes(path: &std::path::Path) -> &[u8] {
    use std::os::unix::ffi::OsStrExt;
    path.as_os_str().as_bytes()
}
```

On Windows, you get UTF-16 code units:

```rust
#[cfg(windows)]
fn path_wide(path: &std::path::Path) -> Vec<u16> {
    use std::os::windows::ffi::OsStrExt;
    path.as_os_str().encode_wide().collect()
}
```

For cross-platform code, there's `as_encoded_bytes()` (stabilized in Rust 1.74), which gives you the raw internal bytes regardless of platform. But the encoding of those bytes is platform-specific, so you should only use this for passing data back into `OsStr::from_encoded_bytes_unchecked` (unsafe) or for byte-level pattern matching.

## CStr and CString - talking to C

C strings are fundamentally different from Rust strings. A C string is a sequence of non-null bytes terminated by a null byte (`\0`). There's no separate length field - you find the end by scanning for `\0`. There's no encoding guarantee - the bytes could be ASCII, Latin-1, UTF-8, or raw binary with no nulls.

[`CStr`](https://doc.rust-lang.org/core/ffi/struct.CStr.html) represents a borrowed C string. [`CString`](https://doc.rust-lang.org/std/ffi/struct.CString.html) is the owned version. They enforce the C string invariant in Rust's type system:

1. No interior null bytes (a null in the middle would look like the end of the string to C)
2. A null terminator at the end

```rust
use std::ffi::{CStr, CString};

fn main() {
    // Creating a CString - validates no interior nulls
    let c_string = CString::new("hello").expect("no interior nulls");

    // Includes the null terminator
    assert_eq!(c_string.as_bytes_with_nul(), b"hello\0");

    // Without the null terminator
    assert_eq!(c_string.as_bytes(), b"hello");

    // Get a *const c_char pointer for FFI
    let ptr = c_string.as_ptr();

    // CString::new fails if the input contains a null byte
    let result = CString::new("hello\0world");
    assert!(result.is_err()); // NulError
}
```

Since Rust 1.77, you can also use the `c""` literal syntax for `&CStr`:

```rust
use std::ffi::CStr;

fn main() {
    let greeting: &CStr = c"hello";
    assert_eq!(greeting.to_bytes(), b"hello");
    assert_eq!(greeting.to_bytes_with_nul(), b"hello\0");
}
```

### CStr in FFI calls

The primary use case is calling C libraries. Here's a realistic example calling libc's `getenv`:

```rust
use std::ffi::{CStr, CString};
use std::os::raw::c_char;

extern "C" {
    fn getenv(name: *const c_char) -> *const c_char;
}

fn get_env_var(name: &str) -> Option<String> {
    let c_name = CString::new(name).ok()?;
    unsafe {
        let ptr = getenv(c_name.as_ptr());
        if ptr.is_null() {
            None
        } else {
            // SAFETY: getenv returns a valid C string or null
            let c_str = CStr::from_ptr(ptr);
            Some(c_str.to_string_lossy().into_owned())
        }
    }
}
```

A critical footgun: dangling pointers. `CString::new("hello").unwrap().as_ptr()` creates a `CString`, calls `as_ptr()`, then drops the `CString` at the end of the expression. The pointer is now dangling. Always bind the `CString` to a variable first:

```rust
// WRONG - dangling pointer!
let ptr = CString::new("hello").unwrap().as_ptr();
// CString is dropped here, ptr points to freed memory

// CORRECT - CString lives long enough
let c_string = CString::new("hello").unwrap();
let ptr = c_string.as_ptr();
// use ptr while c_string is alive
```

The compiler won't catch this because `as_ptr()` returns a raw pointer with no lifetime information. This is documented as a known hazard in the [std::ffi::CString docs](https://doc.rust-lang.org/std/ffi/struct.CString.html#method.as_ptr).

### CStr to Rust string conversion

`CStr` bytes have no encoding guarantee, so converting to `&str` is fallible:

```rust
use std::ffi::CStr;

fn cstr_to_str(c: &CStr) -> Result<&str, std::str::Utf8Error> {
    c.to_str()
}

fn cstr_to_string_lossy(c: &CStr) -> String {
    c.to_string_lossy().into_owned()
}
```

## Path and PathBuf - OsStr with methods

[`Path`](https://doc.rust-lang.org/std/path/struct.Path.html) and [`PathBuf`](https://doc.rust-lang.org/std/path/struct.PathBuf.html) are thin wrappers around `OsStr` and `OsString`. Look at the actual [source code](https://github.com/rust-lang/rust/blob/main/library/std/src/path.rs):

```rust
// From library/std/src/path.rs
pub struct PathBuf {
    inner: OsString,
}

pub struct Path {
    inner: OsStr,
}
```

That's the entire struct definition. `Path` adds no data - it's a newtype over `OsStr` that provides path-specific methods: `extension()`, `file_name()`, `parent()`, `join()`, `components()`, etc.

Because `Path` wraps `OsStr`, a `Path` is NOT guaranteed to be valid UTF-8. This is correct - file paths aren't UTF-8 on most platforms.

```rust
use std::path::{Path, PathBuf};

fn main() {
    let path = Path::new("/home/user/documents/report.pdf");

    // Path decomposition
    println!("parent: {:?}", path.parent());           // Some("/home/user/documents")
    println!("file_name: {:?}", path.file_name());     // Some("report.pdf")
    println!("stem: {:?}", path.file_stem());           // Some("report")
    println!("extension: {:?}", path.extension());      // Some("pdf")

    // Building paths - join allocates a new PathBuf
    let mut buf = PathBuf::from("/home/user");
    buf.push("documents");
    buf.push("report.pdf");
    assert_eq!(buf.as_path(), path);

    // Path to string - fallible!
    match path.to_str() {
        Some(s) => println!("path as str: {}", s),
        None => println!("path contains non-UTF-8 bytes"),
    }
}
```

### Why Path instead of just OsStr?

You could pass `&OsStr` around for file paths. `Path` adds two things:

1. **Cross-platform path separator handling.** `Path::new("foo/bar")` and `Path::new("foo\\bar")` behave correctly on their respective platforms. `components()` splits on the right separator.

2. **Method discoverability.** When your function takes `&Path`, the type communicates intent: this is a filesystem path, not an arbitrary OS string. You get `extension()`, `is_absolute()`, `starts_with()`, and friends in autocomplete.

The cost is zero at runtime. `Path` is a `repr(transparent)` newtype. Passing `&Path` is the same as passing `&OsStr` in the compiled binary.

## The conversion map

Here's how everything connects. Arrows mean "can convert to" - solid lines are free (no allocation, no validation), dashed lines cost something:

```
                     Free conversions (Deref / AsRef)
                     ================================
  String  ----deref----> &str
  OsString ---deref----> &OsStr
  CString  ---deref----> &CStr
  PathBuf  ---deref----> &Path

                     Free via AsRef
                     ==============
  &str    ---AsRef<OsStr>---> &OsStr
  &str    ---AsRef<Path>----> &Path
  &OsStr  ---as_ref--------> &Path  (and vice versa)

                     Fallible / allocating conversions
                     =================================
  &OsStr  ---to_str()---------> Option<&str>        (free if valid UTF-8)
  &OsStr  ---to_string_lossy()-> Cow<str>           (free or allocates)
  &CStr   ---to_str()---------> Result<&str, Utf8Error>
  &str    ---CString::new()----> Result<CString, NulError>
  &Path   ---to_str()---------> Option<&str>
```

Some useful conversions in code:

```rust
use std::ffi::{CStr, CString, OsStr, OsString};
use std::path::{Path, PathBuf};

fn conversions() {
    let s: &str = "hello.txt";

    // &str -> String (allocates)
    let string: String = s.to_owned(); // or s.to_string(), or String::from(s)

    // String -> &str (free, via Deref)
    let back: &str = &string;

    // &str -> &OsStr (free)
    let os: &OsStr = OsStr::new(s);

    // &str -> &Path (free)
    let path: &Path = Path::new(s);

    // &OsStr -> Option<&str> (free if valid, None if not)
    let maybe_str: Option<&str> = os.to_str();

    // &Path -> &OsStr (free)
    let os_from_path: &OsStr = path.as_os_str();

    // PathBuf -> OsString (free, unwraps the inner OsString)
    let pathbuf = PathBuf::from("hello.txt");
    let os_string: OsString = pathbuf.into_os_string();

    // OsString -> Result<String, OsString> (free if valid UTF-8)
    let os_string2 = OsString::from("hello.txt");
    let string_result: Result<String, OsString> = os_string2.into_string();

    // &str -> CString (allocates, validates no interior nulls)
    let c_string = CString::new(s).unwrap();

    // &CStr -> &str (free if valid UTF-8)
    let c_ref: &CStr = c"hello.txt";
    let str_result: Result<&str, _> = c_ref.to_str();
}
```

## The .to_string() everywhere anti-pattern

New Rust developers learn that `.to_string()` "fixes" type errors. Function expects `String`? Slap `.to_string()` on it. Compiler stops complaining. Ship it.

The problem: every `.to_string()` call allocates. It copies the bytes to a fresh heap buffer. In a hot loop or a function called thousands of times, you're paying for allocations that might be avoidable.

```rust
// Anti-pattern: allocating everywhere
fn process(data: &[&str]) -> Vec<String> {
    data.iter()
        .filter(|s| s.len() > 3)
        .map(|s| s.to_string())  // heap allocation for every element
        .collect()
}

// Better: borrow when possible
fn process_borrowed<'a>(data: &[&'a str]) -> Vec<&'a str> {
    data.iter()
        .filter(|s| s.len() > 3)
        .copied()
        .collect()
}
```

The same applies to function signatures. If your function only reads the string, take `&str`, not `String`:

```rust
// Forces caller to allocate if they have a &str
fn bad_api(name: String) -> bool {
    name.len() > 10
}

// Borrows - zero allocation from the caller's side
fn good_api(name: &str) -> bool {
    name.len() > 10
}

// Even better for generic input - accepts String, &str, Cow<str>, etc.
fn flexible_api(name: impl AsRef<str>) -> bool {
    name.as_ref().len() > 10
}
```

For path-related APIs, use `impl AsRef<Path>` - it's what the standard library does:

```rust
use std::path::Path;
use std::io;

fn file_size(path: impl AsRef<Path>) -> io::Result<u64> {
    let metadata = std::fs::metadata(path)?;
    Ok(metadata.len())
}

fn main() -> io::Result<()> {
    // All of these work without conversion
    file_size("Cargo.toml")?;
    file_size(String::from("Cargo.toml"))?;
    file_size(std::path::PathBuf::from("Cargo.toml"))?;
    Ok(())
}
```

### When .to_string() is fine

Don't over-optimize. If you need to store a `String` in a struct that outlives the borrow, allocate. If you're building a `Vec<String>` to return from a function and the input is `&str`, you have to allocate. The problem isn't `.to_string()` itself - it's reaching for it reflexively without considering whether a borrow would work.

## Cow<str> - the best of both worlds

I covered `Cow<str>` in the [flyweight pattern post](/blog/the-flyweight-pattern-sharing-data-efficiently-in-rust/) for its role in avoiding allocations. Here's the angle specific to string type selection.

`Cow<'a, str>` is an enum:

```rust
pub enum Cow<'a, B: ?Sized + ToOwned> {
    Borrowed(&'a B),
    Owned(<B as ToOwned>::Owned),
}

// For str, that's:
// Cow::Borrowed(&'a str)
// Cow::Owned(String)
```

It's 32 bytes on the stack (discriminant + the larger variant, which is `String` at 24 bytes, plus padding). The key property: it can be either a borrow or an owned value, decided at runtime.

This is the right return type for functions that *sometimes* need to allocate:

```rust
use std::borrow::Cow;

fn ensure_ascii_lowercase(s: &str) -> Cow<'_, str> {
    if s.bytes().all(|b| !b.is_ascii_uppercase()) {
        Cow::Borrowed(s) // already lowercase, zero-cost return
    } else {
        Cow::Owned(s.to_ascii_lowercase()) // needs transformation, allocate
    }
}

fn main() {
    let a = ensure_ascii_lowercase("already_lowercase");
    let b = ensure_ascii_lowercase("NEEDS_WORK");

    // Both behave as &str via Deref
    println!("{}, {}", &*a, &*b);

    // Check which is which
    match &a {
        Cow::Borrowed(_) => println!("a: borrowed (no allocation)"),
        Cow::Owned(_) => println!("a: owned (allocated)"),
    }
}
```

`Cow<str>` shows up all over the standard library. `String::from_utf8_lossy` returns `Cow<str>` - if the input is valid UTF-8, it borrows the input bytes directly as a `&str` without allocating. `OsStr::to_string_lossy` does the same thing. `CStr::to_string_lossy` too. The pattern is everywhere because the alternative - always allocating a new `String` - wastes work in the common case where the data is already valid.

### When to use Cow<str> in your own APIs

Use it when:
- Your function returns `&str` most of the time but occasionally needs to allocate
- You're processing text that's usually passed through unchanged
- You want to accept both `&str` and `String` without overloading

Don't use it when:
- The data is always borrowed (just return `&str`)
- The data is always owned (just return `String`)
- The lifetime `'a` would propagate into places you don't want it (async functions, thread spawning, long-lived structs)

That last point is important. `Cow<'a, str>` carries a lifetime. If your struct needs to be `'static`, you'll fight the borrow checker until you call `.into_owned()`, which defeats the purpose.

## Summary: the decision table

Here's the cheat sheet. When you're picking a string type, ask what invariant you need:

| Type | Encoding guarantee | Null terminator | Use case |
|---|---|---|---|
| `String` / `&str` | Valid UTF-8 | No | Text processing, APIs, display, serialization |
| `OsString` / `&OsStr` | Platform-native | No | File paths, env vars, command args |
| `CString` / `&CStr` | None (raw bytes) | Yes | FFI with C libraries |
| `PathBuf` / `&Path` | Platform-native (wraps OsStr) | No | File path manipulation |
| `Cow<'_, str>` | Valid UTF-8 | No | Functions that sometimes allocate |

And the function signature guidelines:

- **Reading text?** Take `&str`
- **Reading a path?** Take `impl AsRef<Path>`
- **Storing text?** Field is `String`
- **Storing a path?** Field is `PathBuf`
- **FFI boundary?** Convert at the edge: `CString` going into C, `CStr::from_ptr` coming back out
- **Returning text that's sometimes borrowed?** Return `Cow<'_, str>`

The mental model is straightforward once you see it. `String`/`&str` are for text that your Rust code owns and manipulates. `OsStr`/`Path` are for data that came from or is going to the operating system. `CStr` is for data that came from or is going to C code. Each boundary - Rust-to-OS, Rust-to-C - has a string type that captures the constraints of both sides.

Don't fight the type system by converting everything to `String`. Use the type that matches where your data came from and where it's going. The conversions exist for the boundaries between these worlds, and Rust makes those boundaries explicit so you can handle the edge cases - invalid UTF-8, interior nulls, platform encoding differences - at the right place in your code.
