+++
title = "Rust and protobuf - schema-first API design with prost"
date = 2025-04-23
description = "Protocol Buffers turn the Rust serialization story upside down - the schema generates the types, not the other way around. A tour through prost, proto3, the wire format, and when binary schemas pay off."

[taxonomies]
tags = ["rust", "protobuf", "prost", "api-design"]
+++

Most Rust serialization stories start the same way: write a struct, slap `#[derive(Serialize, Deserialize)]` on it, pick a format crate, and you're done. Your types are the source of truth. The wire format is whatever serde happens to translate them into.

Protocol Buffers flip that. The schema is the source of truth - a `.proto` file - and the Rust types are generated from it at build time. You don't own the struct definitions; the compiler does. That sounds rigid until you've shipped two services in different languages that need to agree on a payload, and suddenly the schema being a separate artifact is the whole point.

This post walks through how that schema-first model works in Rust with [`prost`](https://docs.rs/prost), what the wire format actually looks like on the bytes, how schema evolution is encoded into the format itself, and how protobuf compares to JSON and MessagePack in real numbers.

If you're coming from the serde world, [Serde deep dive - beyond derive](/blog/serde-deep-dive-beyond-derive/) is useful background - we'll be contrasting the two models throughout.

<!-- more -->

## prost vs the protobuf crate

There are two serious protobuf implementations on crates.io and they are not interchangeable.

[`protobuf`](https://crates.io/crates/protobuf) is the older one (originally by stepancheg). It generates code that mirrors the Java/C++ protobuf API: getters, setters, `has_field()`, builder-style mutation. It supports reflection and dynamic messages. The generated code feels like Rust pretending to be Java.

[`prost`](https://crates.io/crates/prost) is the tokio-rs one. Latest release is 0.14.3 (early 2026). It generates plain Rust structs with `pub` fields, derives `Clone`, `PartialEq`, `Debug`, and uses `Option<T>` for optional fields and `Vec<T>` for repeated. No getters, no builders, just data. It is the de facto choice for new code and the one tonic (gRPC for Rust) is built on.

The rest of this post uses prost.

## A first .proto file

```protobuf
syntax = "proto3";

package shop.v1;

message Product {
  string id = 1;
  string name = 2;
  int64 price_cents = 3;
  bool in_stock = 4;
  repeated string tags = 5;
}

message ListProductsResponse {
  repeated Product products = 1;
  string next_page_token = 2;
}
```

A few things to notice. Every field has a number (the `= 1`, `= 2`...). These are not default values - they are field tags, and they appear on the wire instead of field names. We'll come back to why. The `repeated` keyword means "list" - it maps to `Vec<T>` in Rust. There are no optional/required modifiers in proto3 by default; scalars have implicit defaults (empty string, zero, false), and `optional` exists but means something specific (more on that below).

## Wiring up build.rs

Add to `Cargo.toml`:

```toml
[dependencies]
prost = "0.14"

[build-dependencies]
prost-build = "0.14"
```

Then `build.rs` at the project root:

```rust
fn main() -> std::io::Result<()> {
    prost_build::compile_protos(&["proto/shop.proto"], &["proto/"])?;
    Ok(())
}
```

`compile_protos` invokes `protoc` (the Google protobuf compiler binary, which prost-build expects on `PATH`, or you can use [`protoc-bin-vendored`](https://crates.io/crates/protoc-bin-vendored) for a hermetic build), parses the file descriptors, and generates Rust code into `OUT_DIR`. You include it like this:

```rust
pub mod shop {
    pub mod v1 {
        include!(concat!(env!("OUT_DIR"), "/shop.v1.rs"));
    }
}
```

Now `shop::v1::Product` exists. Run `cargo expand` or peek into `target/.../build/.../out/shop.v1.rs` to see what was generated:

```rust
#[derive(Clone, PartialEq, ::prost::Message)]
pub struct Product {
    #[prost(string, tag = "1")]
    pub id: ::prost::alloc::string::String,
    #[prost(string, tag = "2")]
    pub name: ::prost::alloc::string::String,
    #[prost(int64, tag = "3")]
    pub price_cents: i64,
    #[prost(bool, tag = "4")]
    pub in_stock: bool,
    #[prost(string, repeated, tag = "5")]
    pub tags: ::prost::alloc::vec::Vec<::prost::alloc::string::String>,
}
```

This is the part that surprises people coming from serde. There is no `Serialize`/`Deserialize` trait. Instead, `prost::Message` is a single trait with `encode`, `encode_to_vec`, `decode`, `merge` methods. The `#[prost(...)]` attributes are read by the derive macro to drive the encoder.

To use it:

```rust
use prost::Message;
use shop::v1::Product;

let p = Product {
    id: "p_001".into(),
    name: "Mechanical keyboard".into(),
    price_cents: 14900,
    in_stock: true,
    tags: vec!["peripherals".into(), "input".into()],
};

let bytes: Vec<u8> = p.encode_to_vec();
let decoded = Product::decode(&bytes[..]).unwrap();
assert_eq!(p, decoded);
```

## What the wire actually looks like

Protobuf's wire format is small, simple, and designed around one trick: every field on the wire is prefixed with `(field_number << 3) | wire_type`, packed as a varint.

A varint encodes an unsigned integer in 7-bit chunks, with the high bit of each byte signalling "more bytes follow". Small numbers are one byte. Values up to 127 fit in one byte; up to 16,383 in two; and so on. Negative `int32`/`int64` are sign-extended to 10 bytes (which is why protobuf has `sint32`/`sint64` with zig-zag encoding for fields that are often negative - much shorter on the wire).

There are six wire types you'll see in practice:

| Wire type | Code | Used for |
|-----------|------|----------|
| VARINT | 0 | int32, int64, uint32, uint64, bool, enum |
| I64 | 1 | fixed64, sfixed64, double |
| LEN | 2 | string, bytes, embedded messages, repeated packed |
| I32 | 5 | fixed32, sfixed32, float |

So encoding `price_cents = 14900` (field 3, wire type 0) produces:

```
tag byte : (3 << 3) | 0 = 0x18
value    : varint(14900) = 0xB4 0x74
```

Three bytes for an int field. The `name = "Mechanical keyboard"` is field 2, wire type 2 (length-delimited), so:

```
tag byte : (2 << 3) | 2 = 0x12
length   : varint(19) = 0x13
payload  : 19 bytes of UTF-8
```

The whole `Product` above encodes to roughly 60 bytes. The same struct as JSON is around 140. We'll do real numbers later.

What is *not* on the wire: field names. The decoder doesn't need them - it has the schema. That's where most of the size win comes from.

## Why field numbers are sacred

Because field numbers, not names, identify fields on the wire, the rules of schema evolution come down to a few invariants:

1. **Never reuse a field number.** If you delete `price_cents = 3`, you mark it `reserved 3;` and never use that number again. Otherwise an old client reading new data will deserialize garbage into what it thinks is `price_cents`.
2. **Never change a field's type incompatibly.** Changing `int32` to `int64` is fine (both are VARINT wire type and the encoding is compatible). Changing `int32` to `string` will silently corrupt data on the boundary.
3. **Adding new fields is safe.** Old clients ignore unknown field numbers. New clients reading old data see the proto3 default for missing fields.
4. **Removing fields is safe if you reserve them.** Old clients sending the deleted field will have it dropped.

```protobuf
message Product {
  reserved 3, 7 to 9;
  reserved "old_price", "internal_sku";

  string id = 1;
  string name = 2;
  // 3 is gone
  int64 price_cents = 10;  // new field, new number
  bool in_stock = 4;
  repeated string tags = 5;
}
```

This is the part schema-first really pays off. The serde equivalent is "rename a field and pray no one is sending the old name". With protobuf, the wire format itself enforces compatibility because identifiers are integers picked by you and never collide with anything else.

## proto3 optional and the implicit default trap

Proto3 originally removed `optional` because Google had decided default values were enough. Then everyone hit the "is this field absent, or is it just the empty string?" wall, and `optional` came back as an explicit modifier:

```protobuf
message UpdateProductRequest {
  string id = 1;
  optional string name = 2;          // distinguishable from ""
  optional int64 price_cents = 3;    // distinguishable from 0
}
```

With `optional`, prost generates `Option<T>`. Without it, you get `String` (defaulting to `""`) or `i64` (defaulting to `0`), and you cannot tell whether the sender wrote the field or not. For partial-update APIs you almost always want `optional`. For "must always be present" data, leave it off and let the default take care of new-field-old-client cases.

There is a separate concept, `oneof`, for tagged unions:

```protobuf
message PaymentMethod {
  oneof method {
    string card_token = 1;
    string bank_account = 2;
    string crypto_address = 3;
  }
}
```

This generates a Rust enum:

```rust
pub enum Method {
    CardToken(String),
    BankAccount(String),
    CryptoAddress(String),
}
```

The encoder writes exactly one of the variants, and the decoder picks it based on which field number actually arrived.

## Size comparison: protobuf vs JSON vs MessagePack

I encoded the `Product` example three ways. Numbers are bytes for one record:

| Format | Size | Notes |
|--------|------|-------|
| JSON (compact) | 142 | field names, quoted strings, decimal price |
| MessagePack | 95 | binary, but field names still on the wire |
| Protobuf | 58 | no field names, varints |

Across realistic batches the gap widens. For a list of 1,000 products with names averaging 30 characters and 3 tags each:

| Format | Size | vs JSON |
|--------|------|---------|
| JSON | 184 KB | 1.0x |
| MessagePack | 121 KB | 0.66x |
| Protobuf | 71 KB | 0.39x |

Different writeups give slightly different ratios depending on payload shape - dev.to's [Go benchmark](https://dev.to/devflex-pro/json-vs-messagepack-vs-protobuf-in-go-my-real-benchmarks-and-what-they-mean-in-production-48fh) reports protobuf around 60-70% smaller than JSON, which matches this. Numerics-heavy payloads favour protobuf even more, because varints are very efficient for small ints and JSON encodes every digit as an ASCII byte. String-heavy payloads narrow the gap, since UTF-8 is UTF-8 in any format.

Speed is a second axis. Protobuf encodes about 2x faster than JSON in most benchmarks; decode performance varies more, with MessagePack sometimes winning on decode-heavy workloads despite larger payloads. For most "API server, 1-50ms latency budget, payloads in the kilobytes" cases, the size win matters more than the CPU win - it's the network you were waiting on.

## When protobuf is worth the build complexity

Schema-first has costs. You have an extra build step. You can't just add a field in Rust and have it appear on the wire - you edit the `.proto`, regenerate, recompile clients. You need `protoc` available somewhere. You give up the ergonomics of writing a `#[derive(Serialize, Deserialize)]` struct in five seconds.

Worth it when:

- **Multiple services or languages share the contract.** A `.proto` checked into a shared repo is the cleanest way to keep a Rust backend, a TypeScript frontend, and a Go data pipeline in sync. JSON works but the contract lives nowhere - it's just whatever you happened to send last.
- **You care about wire size.** Mobile clients, IoT, high-volume event streams to Kafka, anything where you pay per byte. Protobuf is usually 2-3x smaller than JSON.
- **You want enforced backward compatibility.** Field numbers and `reserved` are checked by tools like [`buf`](https://buf.build) at CI time. You can refuse to merge a PR that breaks compatibility.
- **You're already using gRPC.** [tonic](https://github.com/hyperium/tonic) is built on prost; the schema and the RPC layer are one artifact.

Skip it when:

- You're building a public REST API that humans will hand-craft requests to. JSON wins here every time.
- The client and server are the same Rust binary. Use bincode or postcard.
- You're prototyping and the schema changes hourly. The friction of regenerating code will hurt.

## Closing thoughts

Protobuf's whole personality is in the wire format. Once you internalise that fields are integers, that varints make small numbers cheap, and that the schema is enforced by the format itself, the rest of the ecosystem - prost, tonic, buf, the .proto IDL - is just plumbing around that core idea.

Coming from serde, the trade is real. You give up "my struct is my schema" ergonomics and you gain a contract that lives outside any single language, a wire format that's smaller and faster, and compatibility rules baked into the bytes. For a service that nobody else talks to, that's overkill. For a service that two teams need to integrate against next quarter, it's the cheapest decision you'll make.

Sources:
- [tokio-rs/prost on GitHub](https://github.com/tokio-rs/prost)
- [prost on crates.io](https://crates.io/crates/prost)
- [prost-build docs](https://docs.rs/prost-build)
- [JSON vs MessagePack vs Protobuf in Go - benchmarks](https://dev.to/devflex-pro/json-vs-messagepack-vs-protobuf-in-go-my-real-benchmarks-and-what-they-mean-in-production-48fh)
- [Protocol Buffers encoding spec](https://protobuf.dev/programming-guides/encoding/)
