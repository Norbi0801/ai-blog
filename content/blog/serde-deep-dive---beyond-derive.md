+++
title = "Serde deep dive - beyond derive"
date = 2026-02-06
description = "Most Rust developers stop at #[derive(Serialize, Deserialize)] - here is what serde actually does under the hood, and the attributes, patterns, and tricks that make real-world serialization work."

[taxonomies]
tags = ["rust", "serde", "serialization", "api-design"]
+++

You `cargo add serde --features derive`, slap `#[derive(Serialize, Deserialize)]` on your struct, and move on. It works. It works so well, in fact, that most Rust developers never look deeper. But serde is not a derive macro with a JSON printer attached. It's a sophisticated framework with a 29-type data model, a Visitor-based deserialization protocol, four different enum representations, zero-copy deserialization, and a pile of attributes that let you reshape data without writing a single line of conversion code.

This post goes below the surface. We'll look at what the derive macro actually generates, how serde's data model decouples your types from wire formats, how to pick the right enum representation for your API, and when to reach for `serde_with` instead of writing custom deserializers by hand.

<!-- more -->

## The serde data model - why one derive works everywhere

The reason `#[derive(Serialize)]` works with JSON, TOML, bincode, MessagePack, postcard, and every other format is the **serde data model**. Your `Serialize` impl doesn't produce JSON. It describes your type in terms of 29 abstract types, and the format's `Serializer` decides how to encode them.

Those 29 types:

**14 primitives:** `bool`, `i8`, `i16`, `i32`, `i64`, `i128`, `u8`, `u16`, `u32`, `u64`, `u128`, `f32`, `f64`, `char`

**15 composites:**

| Type | What it maps to |
|------|-----------------|
| `string` | `String`, `&str` |
| `byte array` | `Vec<u8>`, `&[u8]` |
| `option` | `Option<T>` |
| `unit` | `()` |
| `unit_struct` | `struct Unit;` |
| `unit_variant` | `E::A` in `enum E { A }` |
| `newtype_struct` | `struct Mm(u8)` |
| `newtype_variant` | `E::N(u8)` |
| `seq` | `Vec<T>`, `HashSet<T>` |
| `tuple` | `(u8, String)` |
| `tuple_struct` | `struct Rgb(u8, u8, u8)` |
| `tuple_variant` | `E::T(u8, u8)` |
| `map` | `BTreeMap<K, V>` |
| `struct` | Named fields with compile-time keys |
| `struct_variant` | Enum variant with named fields |

Each type maps to a `serialize_*` method on the [`Serializer`](https://docs.rs/serde/latest/serde/ser/trait.Serializer.html) trait. When your struct calls `serializer.serialize_struct("Point", 2)`, serde_json opens a `{`, bincode writes a length prefix, and TOML starts a section header. Same call, different output.

This is the key architectural insight: serde separates the shape of your data from the encoding. If your `Serialize` impl speaks the data model, it works with every format - present and future. And that's exactly what the derive macro generates.

## What derive actually generates

Let's see what `#[derive(Deserialize)]` produces for a simple struct. You can run `cargo expand` (from the [cargo-expand](https://github.com/dtolnay/cargo-expand) crate) to see this yourself:

```rust
#[derive(Deserialize)]
struct Point {
    x: f64,
    y: f64,
}
```

The derive macro wraps everything in a `const _: () = { ... }` block for isolation, imports serde as `_serde` to avoid name conflicts, and generates two visitors. Simplified, here's the structure:

```rust
const _: () = {
    extern crate serde as _serde;

    impl<'de> _serde::Deserialize<'de> for Point {
        fn deserialize<__D>(__deserializer: __D) -> Result<Self, __D::Error>
        where
            __D: _serde::Deserializer<'de>,
        {
            // Field discriminant enum
            enum __Field { __field0, __field1, __ignore }

            // Visitor for deserializing field names
            struct __FieldVisitor;
            impl<'de> _serde::de::Visitor<'de> for __FieldVisitor {
                type Value = __Field;
                fn expecting(&self, f: &mut fmt::Formatter) -> fmt::Result {
                    f.write_str("field identifier")
                }
                fn visit_str<E: de::Error>(self, v: &str) -> Result<__Field, E> {
                    match v {
                        "x" => Ok(__Field::__field0),
                        "y" => Ok(__Field::__field1),
                        _ => Ok(__Field::__ignore),
                    }
                }
            }

            // Visitor for deserializing the struct itself
            struct __Visitor;
            impl<'de> _serde::de::Visitor<'de> for __Visitor {
                type Value = Point;
                fn expecting(&self, f: &mut fmt::Formatter) -> fmt::Result {
                    f.write_str("struct Point")
                }
                fn visit_map<A: MapAccess<'de>>(
                    self,
                    mut map: A,
                ) -> Result<Point, A::Error> {
                    let mut x: Option<f64> = None;
                    let mut y: Option<f64> = None;
                    while let Some(key) = map.next_key::<__Field>()? {
                        match key {
                            __Field::__field0 => x = Some(map.next_value()?),
                            __Field::__field1 => y = Some(map.next_value()?),
                            __Field::__ignore => { let _ = map.next_value::<IgnoredAny>()?; }
                        }
                    }
                    Ok(Point {
                        x: x.ok_or_else(|| de::Error::missing_field("x"))?,
                        y: y.ok_or_else(|| de::Error::missing_field("y"))?,
                    })
                }
            }

            const FIELDS: &[&str] = &["x", "y"];
            __deserializer.deserialize_struct("Point", FIELDS, __Visitor)
        }
    }
};
```

A few things to notice:

1. **Two visitors.** One for field name lookup (string matching), one for the struct itself (the `visit_map` loop). This is serde's Visitor pattern - the deserializer *drives* the visitor, calling whichever `visit_*` method matches the input format.

2. **Unknown fields are silently ignored** by default (the `__ignore` arm calls `next_value::<IgnoredAny>()` to consume and discard the value). Add `#[serde(deny_unknown_fields)]` to change this.

3. **Field matching is by string name, not position.** JSON field order doesn't matter. Fields can arrive in any sequence. This is why the generated code uses `Option<T>` temporaries and checks for `None` at the end - it handles arbitrary ordering.

4. **The `deserialize_struct` call passes the field names.** This is a hint, not a contract. Self-describing formats like JSON ignore it. Non-self-describing formats like bincode use it to know what to expect.

If you've read the [macros post](/blog/rust-macros-101-declarative-macros-with-macro-rules), you know how `macro_rules!` works. Serde's derive is a proc macro, not a declarative one - it uses `syn` to parse your struct's AST and `quote` to emit code. The macro pipeline has five stages: preprocess (replace `Self`), parse (extract `#[serde(...)]` attributes), validate (catch conflicting attributes), compute bounds (add `T: Serialize` only where needed), and emit code. If you want to read through it, the codegen lives in [`serde_derive/src/de.rs`](https://github.com/serde-rs/serde/blob/master/serde_derive/src/de.rs).

## The attributes you should know

Serde has over 30 container, variant, and field attributes. Here are the ones I use constantly and the ones most people don't know exist.

### `rename_all` - stop fighting naming conventions

Different APIs use different naming. Your Rust struct uses `snake_case`, but the JSON API sends `camelCase`. Instead of renaming every field:

```rust
#[derive(Serialize, Deserialize)]
#[serde(rename_all = "camelCase")]
struct UserProfile {
    user_id: String,          // "userId" in JSON
    display_name: String,     // "displayName" in JSON
    is_active: bool,          // "isActive" in JSON
}
```

The supported conventions: `lowercase`, `UPPERCASE`, `PascalCase`, `camelCase`, `snake_case`, `SCREAMING_SNAKE_CASE`, `kebab-case`, `SCREAMING-KEBAB-CASE`. You can also set different conventions for serialization and deserialization:

```rust
#[serde(rename_all(serialize = "camelCase", deserialize = "PascalCase"))]
```

### `flatten` - merge nested structures

```rust
#[derive(Serialize, Deserialize)]
struct Pagination {
    page: u32,
    per_page: u32,
}

#[derive(Serialize, Deserialize)]
struct ListUsersRequest {
    query: Option<String>,
    #[serde(flatten)]
    pagination: Pagination,
}
```

This serializes as `{"query": "foo", "page": 1, "per_page": 25}` - the pagination fields are inlined into the parent. Useful for composing request types without nesting.

One thing to watch: `flatten` disables the static field name optimization. Serde switches from compile-time field matching to a dynamic `Map<String, Value>` internally. For hot paths, this matters.

### `default` and `skip_serializing_if` - handling optional fields

```rust
#[derive(Serialize, Deserialize)]
struct Config {
    host: String,

    #[serde(default = "default_port")]
    port: u16,

    #[serde(default)]  // uses Default::default() -> false
    debug: bool,

    #[serde(skip_serializing_if = "Option::is_none")]
    api_key: Option<String>,
}

fn default_port() -> u16 { 8080 }
```

If `port` is missing during deserialization, it becomes `8080`. If `api_key` is `None`, it's omitted from the serialized output entirely. The `skip_serializing_if` attribute takes any path to a function with signature `fn(&T) -> bool`.

### `with`, `serialize_with`, `deserialize_with` - surgical overrides

When a single field needs custom logic but you don't want to implement `Serialize`/`Deserialize` for the whole struct:

```rust
mod unix_timestamp {
    use chrono::{DateTime, Utc, TimeZone};
    use serde::{self, Deserialize, Deserializer, Serializer};

    pub fn serialize<S>(date: &DateTime<Utc>, serializer: S) -> Result<S::Ok, S::Error>
    where S: Serializer {
        serializer.serialize_i64(date.timestamp())
    }

    pub fn deserialize<'de, D>(deserializer: D) -> Result<DateTime<Utc>, D::Error>
    where D: Deserializer<'de> {
        let ts = i64::deserialize(deserializer)?;
        Utc.timestamp_opt(ts, 0)
            .single()
            .ok_or_else(|| serde::de::Error::custom("invalid timestamp"))
    }
}

#[derive(Serialize, Deserialize)]
struct Event {
    name: String,

    #[serde(with = "unix_timestamp")]
    created_at: DateTime<Utc>,
}
```

The `with` attribute points to a module with `serialize` and `deserialize` functions. If you only need one direction, use `serialize_with` or `deserialize_with` with a direct function path.

This works, but it's boilerplate-heavy. Every custom format needs a module with specific function signatures. This is where `serde_with` comes in - but we'll get to that.

### `try_from` and `into` - validated deserialization

Want to enforce invariants during deserialization?

```rust
#[derive(Deserialize)]
#[serde(try_from = "String")]
struct EmailAddress(String);

impl TryFrom<String> for EmailAddress {
    type Error = String;
    fn try_from(s: String) -> Result<Self, Self::Error> {
        if s.contains('@') && s.contains('.') {
            Ok(EmailAddress(s))
        } else {
            Err(format!("invalid email: {}", s))
        }
    }
}
```

Serde deserializes a `String`, then runs your `TryFrom` conversion. If it fails, deserialization fails with your error message. If you've read the [From and Into post](/blog/from-and-into-traits-rust-s-conversion-magic), you know how these conversion traits work - serde hooks directly into that ecosystem.

The `into` attribute does the same for serialization: serde converts your type to the target type first, then serializes that.

## Enum representations - the four strategies

This is where most people's serde knowledge ends at "it works" without understanding why their JSON looks the way it does. Serde offers four ways to serialize enums, and picking the wrong one can break API compatibility, tank performance, or make your types impossible to deserialize from certain formats.

If you need a refresher on how Rust enums work at the type level and in memory, I covered that in [Rust enums are not what you think](/blog/rust-enums-are-not-what-you-think-algebraic-data-types-explained).

Given this enum:

```rust
#[derive(Serialize, Deserialize)]
enum Message {
    Ping,
    Text { body: String },
    Binary(Vec<u8>),
}
```

### Externally tagged (default)

No attribute needed. The variant name wraps the content:

```json
{"Text": {"body": "hello"}}
{"Ping": null}
{"Binary": [104, 101, 108, 108, 111]}
```

This is the fastest representation. The deserializer sees the variant name *before* it needs to parse the content, so there's no buffering. It's also the only strategy that works with non-self-describing formats like bincode and postcard - those formats need to know the variant upfront to decide how to interpret the following bytes.

**Use for:** internal APIs, binary protocols, cases where performance matters and you control both sides.

### Internally tagged: `#[serde(tag = "type")]`

The tag becomes a field inside the content:

```json
{"type": "Text", "body": "hello"}
{"type": "Ping"}
```

This is what most REST APIs expect. The tag sits alongside the data, not wrapping it. Clean, readable, familiar.

The cost: serde can't know the variant until it finds the `type` field, which might not be first in the JSON object. So it **buffers the entire object** into an internal `Content` enum (a tree structure mirroring serde's 29-type data model), finds the tag, then replays the buffered data through a `ContentDeserializer`. This means heap allocations proportional to the object size.

```rust
#[derive(Serialize, Deserialize)]
#[serde(tag = "type")]
enum Message {
    Ping,
    Text { body: String },
    // Binary(Vec<u8>), // NOT allowed - tuple variants can't hold the tag field
}
```

Limitation: tuple variants are forbidden. There's no field to insert the tag into. If you need them, use adjacent tagging.

**Use for:** public REST APIs, config files, anything humans read.

### Adjacently tagged: `#[serde(tag = "t", content = "c")]`

Tag and content are sibling fields:

```json
{"t": "Text", "c": {"body": "hello"}}
{"t": "Ping"}
{"t": "Binary", "c": [104, 101, 108, 108, 111]}
```

All variant types are allowed. Buffering only happens if the content appears before the tag in the input. If the tag comes first (which it does when serde serializes it), no buffering is needed.

**Use for:** APIs with mixed variant types (structs and tuples), Haskell-derived protocols.

### Untagged: `#[serde(untagged)]`

No tag at all. Serde tries each variant in declaration order:

```json
{"body": "hello"}
[104, 101, 108, 108, 111]
```

The entire input is buffered, then replayed against each variant until one succeeds. This is the slowest strategy, and error messages are terrible - if no variant matches, you get "data did not match any variant of untagged enum Message", with no indication of which variant came closest.

```rust
#[derive(Serialize, Deserialize)]
#[serde(untagged)]
enum ApiResponse {
    Success { data: serde_json::Value },
    Error { error: String, code: u32 },
}
```

**Use for:** polymorphic inputs where the structure implies the type, backwards-compatible APIs, parsing third-party JSON you don't control.

### Performance hierarchy

From fastest to slowest deserialization:

1. **Externally tagged** - no buffering, variant known upfront
2. **Adjacently tagged** (tag first) - minimal buffering
3. **Internally tagged** - full object buffered into `Content` tree
4. **Adjacently tagged** (content first in input) - content buffered until tag found
5. **Untagged** - full input buffered, replayed N times (once per variant)

The difference is not trivial. Serde issue [#1495](https://github.com/serde-rs/serde/issues/1495) documents cases where switching from manual tag-then-deserialize to `#[serde(tag = "...")]` caused a 2x deserialization slowdown for AST parsing, due to the buffering overhead.

## Zero-copy deserialization

Most of the time, deserializing a `String` field means: parse the JSON, allocate a `String` on the heap, copy the bytes into it. Zero-copy deserialization skips the allocation and copy by borrowing directly from the input buffer.

If you've read the [lifetimes post](/blog/lifetimes-in-rust-the-mental-model-that-finally-clicked), you know that lifetimes are the compiler's way of tracking how long references are valid. Zero-copy deserialization uses that same mechanism. The `'de` lifetime on `Deserializer<'de>` represents the lifetime of the input data. When your struct borrows from the input, it can't outlive that buffer.

```rust
#[derive(Deserialize)]
struct LogEntry<'a> {
    level: &'a str,      // borrows directly from input buffer
    message: &'a str,    // no String allocation
    timestamp: u64,      // copied (it's a number, no borrowing possible)
}

fn parse_logs(json_bytes: &[u8]) -> Vec<LogEntry<'_>> {
    // LogEntry borrows from json_bytes - no string allocations
    serde_json::from_slice(json_bytes).unwrap()
}
```

There are three ways serde can give you a string during deserialization:

| Visitor method | Argument | Lifetime | When it happens |
|---------------|----------|----------|-----------------|
| `visit_str` | `&str` | Transient - dies after the call | Buffered IO, escape processing |
| `visit_borrowed_str` | `&'de str` | Input lifetime | Direct slice of the input buffer |
| `visit_string` | `String` | Owned | Fallback, always works |

When the JSON contains `"hello"`, the deserializer can hand out a `&'de str` pointing directly into the input. But when it contains `"hello\nworld"`, the `\n` escape sequence needs processing - the raw bytes `\`, `n` need to become the byte `0x0A`. That requires a temporary buffer, and the resulting string can't borrow from the input anymore.

This is where `Cow<'a, str>` shines. I covered `Cow` in the [smart pointers post](/blog/smart-pointers-in-rust-box-rc-arc-cow-explained) - it's a copy-on-write pointer that can hold either a borrowed reference or an owned value:

```rust
use std::borrow::Cow;

#[derive(Deserialize)]
struct Config<'a> {
    #[serde(borrow)]
    name: Cow<'a, str>,
    #[serde(borrow)]
    description: Cow<'a, str>,
}
```

If the JSON string is a clean literal, `Cow` holds a `Borrowed(&str)` - zero allocation. If it contains escape sequences, `Cow` holds an `Owned(String)`. Your code handles both transparently through `Deref<Target = str>`.

Note the `#[serde(borrow)]` attribute. For `&str` and `&[u8]`, serde infers borrowing automatically. But for `Cow`, custom types, and nested structs, you need the explicit annotation. It generates the appropriate `'de: 'a` lifetime bound.

**`Deserialize<'de>` vs `DeserializeOwned`**

These two bounds look similar but have very different implications:

- `T: Deserialize<'de>` - the caller provides input with lifetime `'de`, and the output can borrow from it. Zero-copy possible.
- `T: DeserializeOwned` - equivalent to `for<'de> Deserialize<'de>`. The type must work with *any* input lifetime, which means it can't borrow from the input. All data must be owned.

When you write `serde_json::from_reader`, you need `DeserializeOwned` because data is read incrementally from IO - there's no stable buffer to borrow from. When you write `serde_json::from_str` or `from_slice`, you can use `Deserialize<'de>` because the entire input exists in memory.

## serde_with - composable field transformations

The `#[serde(with = "...")]` attribute works, but it's painful to compose. You need a module with exactly the right function signatures for every transformation. The [`serde_with`](https://docs.rs/serde_with/latest/serde_with/) crate (v3.18.0) replaces this with composable, type-level annotations through `serde_as`:

```rust
use serde_with::{serde_as, DisplayFromStr, DurationSeconds};
use std::collections::BTreeMap;
use std::time::Duration;

#[serde_as]
#[derive(Serialize, Deserialize)]
struct ServerConfig {
    #[serde_as(as = "DisplayFromStr")]
    bind_addr: std::net::SocketAddr,    // "127.0.0.1:8080"

    #[serde_as(as = "DurationSeconds<u64>")]
    read_timeout: Duration,             // 30

    #[serde_as(as = "BTreeMap<DisplayFromStr, _>")]
    port_labels: BTreeMap<u16, String>, // {"8080": "http", "443": "https"}
}
```

The `_` placeholder means "use the default serde behavior." This is the composability that `#[serde(with)]` lacks - you can nest transformers: `Vec<DisplayFromStr>` turns `Vec<u16>` into `["8080", "3000"]`. `BTreeMap<_, DisplayFromStr>` transforms only the values.

Helpers I use regularly:

```rust
#[serde_as]
#[derive(Serialize, Deserialize)]
struct Examples {
    // Hex-encode bytes
    #[serde_as(as = "serde_with::hex::Hex")]
    key: Vec<u8>,                       // "deadbeef"

    // Base64-encode bytes
    #[serde_as(as = "serde_with::base64::Base64")]
    payload: Vec<u8>,                   // "aGVsbG8="

    // Accept single item or array
    #[serde_as(as = "serde_with::OneOrMany<_>")]
    tags: Vec<String>,                  // "rust" OR ["rust", "serde"]

    // Use default on deserialization error instead of failing
    #[serde_as(as = "serde_with::DefaultOnError")]
    count: u32,                         // 0 if field is "not a number"
}
```

And `#[skip_serializing_none]` from serde_with saves you from writing `#[serde(skip_serializing_if = "Option::is_none")]` on every optional field:

```rust
use serde_with::skip_serializing_none;

#[skip_serializing_none]
#[derive(Serialize, Deserialize)]
struct UpdateUser {
    name: Option<String>,
    email: Option<String>,
    bio: Option<String>,
    // All None fields are omitted from output, no per-field attribute needed
}
```

## Writing a custom deserializer - the Visitor pattern

Sometimes no attribute combination does what you need. Maybe you're parsing a legacy format that represents durations as `"5m30s"` strings. You need a hand-written `Deserialize` impl.

The core protocol: your type's `deserialize` method creates a `Visitor` and hands it to the deserializer. The deserializer inspects the input and calls one of the `visit_*` methods on your visitor. Your visitor produces the final value.

```rust
use std::fmt;
use serde::de::{self, Deserializer, Visitor};
use std::time::Duration;

struct DurationVisitor;

impl<'de> Visitor<'de> for DurationVisitor {
    type Value = Duration;

    fn expecting(&self, formatter: &mut fmt::Formatter) -> fmt::Result {
        formatter.write_str("a duration string like \"5m30s\" or \"100ms\"")
    }

    fn visit_str<E>(self, v: &str) -> Result<Duration, E>
    where
        E: de::Error,
    {
        parse_duration_string(v).map_err(E::custom)
    }

    // Also handle the case where the format provides an owned String
    fn visit_string<E>(self, v: String) -> Result<Duration, E>
    where
        E: de::Error,
    {
        self.visit_str(&v)
    }

    // Accept plain integer as seconds
    fn visit_u64<E>(self, v: u64) -> Result<Duration, E>
    where
        E: de::Error,
    {
        Ok(Duration::from_secs(v))
    }
}

impl<'de> Deserialize<'de> for MyDuration {
    fn deserialize<D>(deserializer: D) -> Result<Self, D::Error>
    where
        D: Deserializer<'de>,
    {
        deserializer.deserialize_str(DurationVisitor).map(MyDuration)
    }
}
```

A critical subtlety: `deserialize_str` is a **hint**, not a contract. It tells the format "I'd prefer a string." But JSON represents everything as text anyway, and bincode doesn't look at hints. Your visitor should implement multiple `visit_*` methods to handle what formats actually produce. If a `visit_*` method isn't overridden, its default returns a type error.

For `deserialize_any` (used in self-describing formats), the deserializer decides which `visit_*` to call based on the input. This is maximally flexible but won't work with non-self-describing formats like bincode - they need the type hint to know how to parse the byte stream.

## Practical patterns

### The `#[serde(other)]` catch-all

When you're reading an enum from an API that might add new variants in the future:

```rust
#[derive(Serialize, Deserialize)]
#[serde(rename_all = "snake_case")]
enum Status {
    Active,
    Inactive,
    Suspended,
    #[serde(other)]
    Unknown,
}
```

Any unrecognized string deserializes to `Unknown` instead of failing. This makes your code forward-compatible with API changes. The `other` attribute only works on unit variants and only with internally tagged or untagged enums.

### String-or-struct pattern

Some APIs accept both a shorthand string and a full object:

```rust
// Accept either "localhost:8080" or {"host": "localhost", "port": 8080}

use std::str::FromStr;

#[derive(Deserialize)]
struct Endpoint {
    host: String,
    port: u16,
}

impl FromStr for Endpoint {
    type Err = String;
    fn from_str(s: &str) -> Result<Self, Self::Err> {
        let (host, port) = s.rsplit_once(':')
            .ok_or("expected host:port")?;
        Ok(Endpoint {
            host: host.to_owned(),
            port: port.parse().map_err(|e| format!("{}", e))?,
        })
    }
}

fn string_or_struct<'de, D>(deserializer: D) -> Result<Endpoint, D::Error>
where
    D: serde::Deserializer<'de>,
{
    struct StringOrStruct;

    impl<'de> Visitor<'de> for StringOrStruct {
        type Value = Endpoint;
        fn expecting(&self, f: &mut fmt::Formatter) -> fmt::Result {
            f.write_str("string or map")
        }
        fn visit_str<E: de::Error>(self, v: &str) -> Result<Endpoint, E> {
            FromStr::from_str(v).map_err(E::custom)
        }
        fn visit_map<A: de::MapAccess<'de>>(self, map: A) -> Result<Endpoint, A::Error> {
            Deserialize::deserialize(de::value::MapAccessDeserializer::new(map))
        }
    }

    deserializer.deserialize_any(StringOrStruct)
}
```

This is a common pattern in config file parsers (think Docker Compose, GitHub Actions) where brevity matters for simple cases.

### Externally tagged but with a default

Combine `#[serde(tag)]` with `#[serde(default)]` for config enums with fallback behavior:

```rust
#[derive(Serialize, Deserialize)]
#[serde(tag = "backend")]
enum Storage {
    Memory,
    Sqlite { path: String },
    Postgres {
        url: String,
        #[serde(default = "default_pool_size")]
        pool_size: u32,
    },
}

fn default_pool_size() -> u32 { 10 }
```

Deserializing `{"backend": "Postgres", "url": "postgres://..."}` gives you a `pool_size` of 10 without requiring it in the input.

## Performance reality check

Serde's design prioritizes correctness and generality. For most applications, it's fast enough that you'll never think about it. But when you're parsing millions of JSON documents per second, the numbers matter.

Here's where things stand (from the [rust_serialization_benchmark](https://github.com/djkoloski/rust_serialization_benchmark), March 2026, rustc nightly):

| Library | Serialize | Deserialize | Size |
|---------|-----------|-------------|------|
| bitcode 0.6 | 146 us | 1.5 ms | 704 KB |
| bincode 2.0 | 332 us | 2.1 ms | 741 KB |
| **serde_json 1.0** | **3.87 ms** | **6.23 ms** | **1,827 KB** |

JSON is 4-20x slower than binary formats. That's the cost of a text-based, self-describing format - not serde itself. If you need raw speed and control both ends, use a binary format (bitcode, postcard, bincode) and serde will generate equally fast serialization code for those.

Within JSON specifically, alternatives like [sonic-rs](https://github.com/cloudwego/sonic-rs) use SIMD instructions to parse 2-3x faster than serde_json. And serde_json itself has gotten faster - a [recent optimization](https://purplesyringa.moe/blog/i-sped-up-serde-json-strings-by-20-percent/) improved string parsing by 20% using SWAR (SIMD Within A Register) techniques for escape detection.

**Compile time** is serde's real cost. Every `Serialize`/`Deserialize` impl generates specialized code for every format through monomorphization. In large projects, this compounds. Alternatives like [miniserde](https://github.com/dtolnay/miniserde) trade runtime performance for faster compiles by using runtime dispatch instead of monomorphization. Worth considering if your compile times are painful and serialization isn't a bottleneck.

## The sharp edges

A few things that will bite you if you're not aware:

**`flatten` + `deny_unknown_fields` don't compose.** If you flatten a struct and also deny unknown fields, the flattened fields are treated as unknown. This is a [known limitation](https://github.com/serde-rs/serde/issues/1358).

**Untagged enum errors are useless.** "data did not match any variant of untagged enum X" tells you nothing about which variant was closest to matching or why each one failed. For debugging, temporarily switch to internal tagging.

**`Option<Option<T>>` has surprising behavior.** In JSON, `null` and missing-field are conflated. `Some(None)` and `None` both serialize to either `null` or field omission. If you need to distinguish "field present with null value" from "field absent", you need a custom deserializer or a three-state enum.

**Internally tagged enums can't contain `flatten`.** The buffering mechanisms conflict. This is [tracked](https://github.com/serde-rs/serde/issues/1183) but unlikely to be fixed because of fundamental design constraints.

## When not to use serde

Serde is the right choice for 95% of serialization tasks in Rust. But there are cases where it's not:

- **Schema evolution at scale** - protobuf or Cap'n Proto give you forward/backward compatibility guarantees that serde doesn't.
- **Zero-copy without lifetimes** - [rkyv](https://github.com/rkyv/rkyv) does true zero-copy (archived data *is* the deserialized struct, backed by `mmap`) without infecting your API with lifetime parameters.
- **Compile time matters more than runtime** - miniserde or nanoserde, if you can live with fewer features.

For everything else, serde's combination of correctness, ecosystem support, and flexibility is hard to beat. Over 70,000 crates on crates.io depend on it. Every Rust JSON, TOML, YAML, MessagePack, and CBOR library speaks serde. The derive macro generates correct, efficient code, and the attribute system handles 90% of format mismatches without custom code.

The remaining 10% is where this post's patterns come in. Know the data model, understand the enum representations, reach for `serde_with` before writing custom visitors, and save hand-written `Deserialize` impls for when nothing else fits.
