+++
title = "From and Into traits - Rust's conversion magic"
date = 2025-04-01
description = "How impl From<X> for Y gives you Into, error conversion with ?, clean API boundaries, and why From impls replace constructors in idiomatic Rust."

[taxonomies]
tags = ["rust", "traits", "type-system", "api-design"]
+++

Every Rust codebase has a moment where you write `.into()` and the compiler just figures it out. You pass a `&str` where a `String` is expected, return an error type that isn't the function's return type, or construct a newtype from raw data - and it works. No explicit cast, no manual conversion function, no ceremony. Behind this is a pair of traits - `From` and `Into` - that form the backbone of Rust's type conversion system.

What makes them interesting isn't the trait definitions. Those are trivial. What's interesting is the ecosystem they create: a single `impl From<X> for Y` gives you six things for free, the `?` operator is built on top of `From`, and the pattern of accepting `impl Into<T>` in function signatures produces APIs that feel effortless to call.

<!-- more -->

## The traits themselves

The definitions in [`core::convert`](https://github.com/rust-lang/rust/blob/main/library/core/src/convert/mod.rs) are almost disappointingly simple:

```rust
pub trait From<T>: Sized {
    fn from(value: T) -> Self;
}

pub trait Into<T>: Sized {
    fn into(self) -> T;
}
```

`From<T>` converts a `T` into `Self`. `Into<T>` converts `Self` into a `T`. They're mirror images. But the relationship between them is what matters.

## The blanket impl that changes everything

The standard library contains this [blanket implementation](https://github.com/rust-lang/rust/blob/main/library/core/src/convert/mod.rs):

```rust
impl<T, U> Into<U> for T
where
    U: From<T>,
{
    fn into(self) -> U {
        U::from(self)
    }
}
```

If you covered [blanket implementations](/blog/rust-trait-bounds-where-clauses-associated-types-and-the-rest-of-the-iceberg) before, this is a textbook example. For every pair `(T, U)` where `U: From<T>`, the compiler automatically provides `Into<U> for T`. You implement one trait, you get both directions.

This is why the Rust convention is **always implement `From`, never `Into` directly**. Writing `impl Into<Y> for X` only gives you the `Into` direction. Writing `impl From<X> for Y` gives you both `From` and `Into`.

There's also a reflexive impl - every type can convert from itself:

```rust
impl<T> From<T> for T {
    #[inline(always)]
    fn from(t: T) -> T {
        t
    }
}
```

This might seem useless, but it matters when you write generic code that accepts `impl Into<T>` - callers can pass a `T` directly without conversion.

## What one From impl actually gives you

Let's write a concrete example. Say you have a domain type:

```rust
struct EmailAddress(String);

impl From<String> for EmailAddress {
    fn from(s: String) -> Self {
        EmailAddress(s)
    }
}
```

That single impl gives you:

```rust
// 1. Explicit From conversion
let email = EmailAddress::from("alice@example.com".to_string());

// 2. Into conversion (from blanket impl)
let email: EmailAddress = "alice@example.com".to_string().into();

// 3. TryFrom with Error = Infallible (from blanket impl)
let email = EmailAddress::try_from("alice@example.com".to_string()).unwrap();

// 4. TryInto with Error = Infallible (from blanket impl)
let email: EmailAddress = "alice@example.com".to_string().try_into().unwrap();

// 5. Works as a function argument with impl Into<EmailAddress>
fn send_to(addr: impl Into<EmailAddress>) {
    let addr = addr.into();
    // ...
}
send_to("alice@example.com".to_string());

// 6. Error conversion with ? (when used in Result chains)
```

The `TryFrom` one might surprise you. There's another blanket impl in the standard library:

```rust
impl<T, U> TryFrom<U> for T
where
    U: Into<T>,
{
    type Error = Infallible;

    fn try_from(value: U) -> Result<Self, Self::Error> {
        Ok(U::into(value))
    }
}
```

Every infallible `From` impl automatically becomes a `TryFrom` impl with `Error = Infallible`. The compiler knows it can never fail.

## The `impl Into<T>` pattern for clean APIs

This is where `From`/`Into` transforms API design. Compare these two constructors:

```rust
// Rigid - caller must provide exact types
impl Config {
    fn new(host: String, port: u16, db_name: String) -> Self {
        Config { host, port, db_name }
    }
}

// Caller has to write:
let config = Config::new("localhost".to_string(), 5432, "mydb".to_string());
```

Now with `impl Into<String>`:

```rust
impl Config {
    fn new(host: impl Into<String>, port: u16, db_name: impl Into<String>) -> Self {
        Config {
            host: host.into(),
            port,
            db_name: db_name.into(),
        }
    }
}

// Caller can write:
let config = Config::new("localhost", 5432, "mydb");
```

The `.to_string()` noise disappears. The function accepts anything that can convert into a `String` - a `&str`, a `String`, a `Cow<str>`, a `Box<str>`. The conversion happens inside the function, and the caller's code reads naturally.

This pattern is all over the Rust ecosystem. Look at `std::fs::File::open`:

```rust
pub fn open<P: AsRef<Path>>(path: P) -> io::Result<File>
```

It uses `AsRef<Path>` instead of `Into<PathBuf>` because it only needs to borrow the path, not own it. That's a deliberate choice - more on `AsRef` later.

If you worked through the [string types post](/blog/why-rust-has-so-many-string-types), you already know why Rust has `String` vs `&str`. The `impl Into<String>` pattern is how you bridge that gap at API boundaries without forcing callers to think about it.

## Error conversion and the `?` operator

This is the most impactful use of `From` in practice. When you write `?` on a `Result`, the compiler uses `From` to convert the error type. Here's what actually happens.

Consider this code:

```rust
use std::io;
use std::num::ParseIntError;

#[derive(Debug)]
enum AppError {
    Io(io::Error),
    Parse(ParseIntError),
}

impl From<io::Error> for AppError {
    fn from(e: io::Error) -> Self {
        AppError::Io(e)
    }
}

impl From<ParseIntError> for AppError {
    fn from(e: ParseIntError) -> Self {
        AppError::Parse(e)
    }
}

fn read_config(path: &str) -> Result<u32, AppError> {
    let content = std::fs::read_to_string(path)?;  // io::Error -> AppError
    let port: u32 = content.trim().parse()?;        // ParseIntError -> AppError
    Ok(port)
}
```

Both `?` calls work, even though `read_to_string` returns `Result<_, io::Error>` and `parse` returns `Result<_, ParseIntError>`. The function returns `Result<_, AppError>`. The `?` operator bridges the gap.

Under the hood, the `?` operator desugars through the [`Try`](https://doc.rust-lang.org/nightly/src/core/ops/try_trait.rs.html) and `FromResidual` traits ([RFC 3058](https://rust-lang.github.io/rfcs/3058-try-trait-v2.html)):

```rust
// What `content = std::fs::read_to_string(path)?` becomes (simplified):
let content = match std::fs::read_to_string(path) {
    Ok(val) => val,
    Err(e) => return Err(From::from(e)),  // <-- From conversion
};
```

The key line is `From::from(e)`. The compiler sees that you need to convert `io::Error` into `AppError`, looks for `impl From<io::Error> for AppError`, finds it, and calls it. No manual mapping, no `.map_err()`.

The actual mechanism lives in `Result`'s `FromResidual` implementation:

```rust
impl<T, E, F: From<E>> FromResidual<Result<!, E>> for Result<T, F> {
    fn from_residual(x: Result<!, E>) -> Self {
        match x {
            Err(e) => Err(From::from(e)),
        }
    }
}
```

Notice the bound: `F: From<E>`. That's the constraint that makes `?` work. Your return type's error (`F`) must implement `From` over the expression's error (`E`).

This is why [thiserror](https://github.com/dtolnay/thiserror) exists. Instead of writing those `From` impls by hand, you derive them:

```rust
use thiserror::Error;

#[derive(Error, Debug)]
enum AppError {
    #[error("io error: {0}")]
    Io(#[from] io::Error),           // generates impl From<io::Error> for AppError

    #[error("parse error: {0}")]
    Parse(#[from] ParseIntError),    // generates impl From<ParseIntError> for AppError
}
```

The `#[from]` attribute generates the same `From` impls we wrote manually. One annotation per variant, and `?` works everywhere.

### The axum pattern

Web frameworks push this further. Here's a pattern from [axum](https://github.com/tokio-rs/axum/blob/main/examples/anyhow-error-response/src/main.rs) for handling errors in request handlers:

```rust
struct AppError(anyhow::Error);

impl IntoResponse for AppError {
    fn into_response(self) -> Response {
        (
            StatusCode::INTERNAL_SERVER_ERROR,
            format!("Something went wrong: {}", self.0),
        )
            .into_response()
    }
}

// This blanket From impl is the magic:
impl<E> From<E> for AppError
where
    E: Into<anyhow::Error>,
{
    fn from(err: E) -> Self {
        Self(err.into())
    }
}
```

Now any handler returning `Result<impl IntoResponse, AppError>` can use `?` on any error type that converts into `anyhow::Error` - which is basically everything that implements `std::error::Error`. A single `From` impl makes every error type in the ecosystem work with your error handler.

## From impls as constructors

A pattern that shows up in well-designed Rust libraries: use `From` impls instead of named constructors when the conversion is obvious and unambiguous.

```rust
struct Milliseconds(u64);
struct Seconds(u64);

impl From<Seconds> for Milliseconds {
    fn from(s: Seconds) -> Self {
        Milliseconds(s.0 * 1000)
    }
}
```

Now you can write `Milliseconds::from(Seconds(5))` or `let ms: Milliseconds = Seconds(5).into()`. The conversion is self-documenting from the type names alone.

The [derive_more](https://crates.io/crates/derive_more) crate takes this further with `#[derive(From)]`:

```rust
use derive_more::From;

#[derive(From)]
struct Meters(f64);

// Generates:
// impl From<f64> for Meters {
//     fn from(value: f64) -> Self { Meters(value) }
// }

let m: Meters = 42.0.into();
```

For newtype wrappers - types covered in the [API design post](/blog/api-design-in-rust-making-impossible-states-unrepresentable) - this eliminates the boilerplate of writing `fn new(value: f64) -> Self` when `From` communicates the intent just as clearly.

But there's a judgment call here. `From` should be used when the conversion is:
- **Obvious** from the types involved
- **Infallible** - it always succeeds
- **Not lossy** - no data is thrown away
- **Cheap enough** to call without warning the caller

If any of those don't hold, use a named method instead. `fn parse(s: &str) -> Result<Self, ParseError>` is clearer than `TryFrom<&str>` when the parsing rules are complex. `fn from_raw_parts(ptr, len, cap)` is clearer than `From<(*mut u8, usize, usize)>` because the conversion is unsafe and the parameter names carry meaning.

## TryFrom and TryInto - when conversion can fail

Not every conversion is infallible. Parsing a string as an integer can fail. Converting a `u64` to a `u16` can overflow. These need `TryFrom`:

```rust
pub trait TryFrom<T>: Sized {
    type Error;
    fn try_from(value: T) -> Result<Self, Self::Error>;
}
```

The associated `Error` type lets each implementation define its own error. And just like `From`/`Into`, there's a blanket impl that gives you `TryInto` for free when you implement `TryFrom`.

Here's a real use case - validated domain types:

```rust
use std::fmt;

#[derive(Debug, Clone)]
struct Port(u16);

#[derive(Debug)]
struct PortError(u32);

impl fmt::Display for PortError {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "invalid port number: {} (must be 1-65535)", self.0)
    }
}

impl std::error::Error for PortError {}

impl TryFrom<u32> for Port {
    type Error = PortError;

    fn try_from(value: u32) -> Result<Self, Self::Error> {
        if value == 0 || value > 65535 {
            Err(PortError(value))
        } else {
            Ok(Port(value as u16))
        }
    }
}

// Usage:
let port = Port::try_from(8080).unwrap();   // Ok(Port(8080))
let bad = Port::try_from(70000);            // Err(PortError(70000))

// Or with .try_into():
let port: Port = 8080u32.try_into().unwrap();
```

The `TryFrom`/`TryInto` traits were stabilized in [Rust 1.34](https://github.com/rust-lang/rfcs/blob/master/text/1542-try-from.md). Before that, people used `From` for fallible conversions by panicking on invalid input - a bad pattern that still shows up in older code.

The standard library uses `TryFrom` extensively for numeric narrowing:

```rust
let x: u8 = u8::try_from(256u16).unwrap_err();  // TryFromIntError
let y: u8 = u8::try_from(255u16).unwrap();       // 255
```

Note that `From<u8> for u16` exists (widening is infallible), but `From<u16> for u8` does not (narrowing can lose data). This is a deliberate design choice - `From` guarantees lossless conversion.

### TryFrom with serde

A powerful pattern: use `TryFrom` for validated deserialization. Serde supports `#[serde(try_from = "Type")]`:

```rust
use serde::Deserialize;

#[derive(Deserialize)]
#[serde(try_from = "String")]
struct Email(String);

impl TryFrom<String> for Email {
    type Error = String;

    fn try_from(value: String) -> Result<Self, Self::Error> {
        if value.contains('@') && value.contains('.') {
            Ok(Email(value))
        } else {
            Err(format!("invalid email: {}", value))
        }
    }
}
```

Now any JSON with an invalid email field fails at deserialization time, not somewhere deep in your business logic. The validation lives with the type, not scattered across handlers.

## AsRef and AsMut - borrowing conversions

`From`/`Into` consume the value. Sometimes you just want to borrow it differently. That's `AsRef` and `AsMut`:

```rust
pub trait AsRef<T: ?Sized> {
    fn as_ref(&self) -> &T;
}

pub trait AsMut<T: ?Sized> {
    fn as_mut(&mut self) -> &mut T;
}
```

The key difference: `AsRef`/`AsMut` return references. No allocation, no ownership transfer. They're used when you need to "view" a value as a different type.

The standard library is full of `AsRef` impls:

```rust
// String implements AsRef<str>, AsRef<[u8]>, AsRef<Path>
let s = String::from("hello");
let bytes: &[u8] = s.as_ref();
let str_ref: &str = s.as_ref();

// Vec<T> implements AsRef<[T]>
let v = vec![1, 2, 3];
let slice: &[i32] = v.as_ref();

// PathBuf implements AsRef<Path>
let p = std::path::PathBuf::from("/tmp");
let path: &std::path::Path = p.as_ref();
```

### When to use AsRef vs Into as function parameters

This is a common question. The rule is about ownership:

```rust
// Use AsRef when you only need to READ the value
fn print_path(path: impl AsRef<std::path::Path>) {
    println!("{}", path.as_ref().display());
}
// Accepts: &str, String, PathBuf, &Path, &PathBuf, ...

// Use Into when you need to STORE/OWN the value
fn set_name(&mut self, name: impl Into<String>) {
    self.name = name.into();  // Takes ownership
}
// Accepts: &str (allocates), String (moves), Cow<str>, ...
```

`AsRef` is cheaper at the call site - no allocation needed. `Into` may allocate (e.g., `&str` into `String`) but gives you ownership. Choose based on what the function does with the value.

Here's a decision matrix:

| You need to... | Use | Why |
|---|---|---|
| Read the value | `impl AsRef<T>` | Borrows, zero-cost |
| Store the value in a struct | `impl Into<T>` | Takes ownership |
| Validate and possibly reject | `impl TryInto<T>` | Returns Result |
| Read and maybe modify | `impl AsMut<T>` | Mutable borrow |

## The orphan rule constraint

There's a practical limitation you'll hit. Rust's orphan rule says: you can only implement a trait for a type if you define either the trait or the type (or both) in your crate.

Since `From` is defined in `std`, you can only implement it when your type is the target:

```rust
// Works - MyType is local
impl From<String> for MyType {
    fn from(s: String) -> Self { MyType(s) }
}

// Works - MyType is local (source position is fine too, for local types)
impl From<MyType> for String {
    fn from(m: MyType) -> Self { m.0 }
}

// Does NOT work - both types are foreign
impl From<serde_json::Value> for reqwest::Body {
    // Error: neither trait nor types are local
}
```

The workaround is the [newtype pattern](/blog/api-design-in-rust-making-impossible-states-unrepresentable). Wrap the foreign type in your own struct, and now you can implement `From` on it:

```rust
struct JsonBody(serde_json::Value);

impl From<JsonBody> for reqwest::Body {
    fn from(j: JsonBody) -> Self {
        reqwest::Body::from(j.0.to_string())
    }
}
```

This is exactly why newtypes exist in Rust's ecosystem - they create a local type that you can hang trait impls on.

## Zero-cost? Let's check

A reasonable question: does all this trait indirection add overhead? The answer: no. `From`/`Into` are monomorphized - the compiler generates specialized code for each concrete type pair. There's no vtable, no dynamic dispatch.

The blanket `Into` impl is marked `#[inline]`, and the reflexive `From<T> for T` is marked `#[inline(always)]`. LLVM inlines these and optimizes them away completely.

You can verify this on [Compiler Explorer](https://rust.godbolt.org/). Take this code:

```rust
struct Meters(f64);

impl From<f64> for Meters {
    fn from(v: f64) -> Self {
        Meters(v)
    }
}

pub fn direct(v: f64) -> Meters {
    Meters(v)
}

pub fn via_from(v: f64) -> Meters {
    Meters::from(v)
}

pub fn via_into(v: f64) -> Meters {
    v.into()
}
```

With `opt-level=2`, all three functions compile to identical assembly. The `From` and `Into` calls are fully erased. The abstraction is genuinely zero-cost.

## Chaining conversions

One thing `From`/`Into` does not do: transitive conversion. If `A: Into<B>` and `B: Into<C>`, you don't automatically get `A: Into<C>`. Each conversion step is explicit:

```rust
struct Raw(Vec<u8>);
struct Parsed(String);
struct Validated(String);

impl From<Raw> for Parsed {
    fn from(r: Raw) -> Self {
        Parsed(String::from_utf8_lossy(&r.0).into_owned())
    }
}

impl From<Parsed> for Validated {
    fn from(p: Parsed) -> Self {
        Validated(p.0.trim().to_string())
    }
}

// This does NOT work:
// let v: Validated = Raw(vec![72, 101, 108, 108, 111]).into();

// You must chain explicitly:
let raw = Raw(vec![72, 101, 108, 108, 111]);
let parsed: Parsed = raw.into();
let validated: Validated = parsed.into();

// Or in one expression:
let validated = Validated::from(Parsed::from(Raw(vec![72, 101, 108, 108, 111])));
```

This is by design. Transitive conversions would make the trait system ambiguous - if there are multiple paths from `A` to `C`, which one should the compiler choose? Explicit chains keep things predictable.

## Common mistakes

**Implementing `From` for lossy conversions.** `From` has an implicit contract: the conversion is lossless and infallible. If you implement `From<f64> for i32` with truncation, you'll surprise every caller who expects `From` to preserve data. Use `TryFrom`, or a named method like `fn truncate(v: f64) -> i32`.

**Implementing `Into` directly.** Unless you're working around the orphan rule (you need the conversion but can't implement `From` because neither the source nor the target type is local - which is rare since you typically control at least one side), implement `From` instead. You lose the reverse direction otherwise.

**Overusing `impl Into<String>` in public APIs.** This creates a generic function boundary that's monomorphized for every call site with a different type. For hot paths in tight loops, this is fine - it optimizes away. But for public library APIs with many call sites, it can increase binary size. Measure before worrying, but be aware.

**Implementing `From` for types with multiple reasonable conversions.** If converting from `(f64, f64)` to `Point` could mean `(x, y)` or `(lat, lng)`, neither is obvious. Use named constructors instead: `Point::from_xy(x, y)` and `Point::from_lat_lng(lat, lng)`. `From` works best when there's exactly one sensible conversion.

## Putting it all together

Here's a complete example combining everything - a small HTTP client wrapper with clean error handling and ergonomic API:

```rust
use std::collections::HashMap;
use std::fmt;

// Domain types with From impls
#[derive(Debug, Clone)]
struct Url(String);

impl From<&str> for Url {
    fn from(s: &str) -> Self {
        Url(s.to_string())
    }
}

impl From<String> for Url {
    fn from(s: String) -> Self {
        Url(s)
    }
}

// Header with Into-based constructor
#[derive(Debug, Clone)]
struct Header {
    name: String,
    value: String,
}

impl Header {
    fn new(name: impl Into<String>, value: impl Into<String>) -> Self {
        Header {
            name: name.into(),
            value: value.into(),
        }
    }
}

// Error type with From impls for ? operator
#[derive(Debug)]
enum ClientError {
    Io(std::io::Error),
    InvalidUrl(String),
    Timeout,
}

impl From<std::io::Error> for ClientError {
    fn from(e: std::io::Error) -> Self {
        ClientError::Io(e)
    }
}

impl fmt::Display for ClientError {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        match self {
            ClientError::Io(e) => write!(f, "io error: {}", e),
            ClientError::InvalidUrl(u) => write!(f, "invalid url: {}", u),
            ClientError::Timeout => write!(f, "request timed out"),
        }
    }
}

impl std::error::Error for ClientError {}

// Request builder using Into for ergonomic construction
struct Request {
    url: Url,
    headers: Vec<Header>,
    body: Option<String>,
}

impl Request {
    fn get(url: impl Into<Url>) -> Self {
        Request {
            url: url.into(),
            headers: Vec::new(),
            body: None,
        }
    }

    fn header(mut self, name: impl Into<String>, value: impl Into<String>) -> Self {
        self.headers.push(Header::new(name, value));
        self
    }

    fn body(mut self, body: impl Into<String>) -> Self {
        self.body = Some(body.into());
        self
    }
}

// Usage reads cleanly - no .to_string() noise
fn example() -> Result<(), ClientError> {
    let req = Request::get("https://api.example.com/users")
        .header("Authorization", "Bearer token123")
        .header("Content-Type", "application/json")
        .body(r#"{"name": "Alice"}"#);

    // If send() returned Result<_, io::Error>, the ? would
    // convert it to ClientError via our From impl
    Ok(())
}
```

Every `impl Into<T>` parameter accepts multiple types. Every `From` impl on `ClientError` makes `?` work. The builder methods chain naturally. No `.to_string()` calls litter the call site.

## The conversion trait map

Here's how all the conversion traits relate:

```
Implement From<T> for U
    |
    +-- gives you Into<U> for T            (blanket impl)
    +-- gives you TryFrom<T> for U         (blanket impl, Error = Infallible)
    +-- gives you TryInto<U> for T         (blanket impl, Error = Infallible)
    +-- enables ? operator (error paths)   (via FromResidual)

Implement TryFrom<T> for U
    |
    +-- gives you TryInto<U> for T         (blanket impl)

AsRef<T> / AsMut<T>
    |
    +-- independent, for borrowing only
    +-- no ownership transfer
```

## Quick reference

When designing your own types:

1. **Newtype wrapping an inner value?** Implement `From<Inner>` for your type, and `From<YourType>` for the inner type if the reverse is meaningful.

2. **Error type with multiple sources?** Implement `From<SourceError>` for each variant, or use `thiserror` with `#[from]`.

3. **Constructor that takes strings?** Accept `impl Into<String>` instead of `String`.

4. **Function that reads a path?** Accept `impl AsRef<Path>` instead of `&Path`.

5. **Validated wrapper type?** Implement `TryFrom` with a descriptive error type.

6. **Multiple reasonable conversions from the same source type?** Don't use `From` - use named methods.

The conversion traits are simple on their own. Their power comes from how they compose - with blanket impls, the `?` operator, and `impl Into<T>` bounds - into an ecosystem where type conversions are safe, discoverable, and zero-cost.
