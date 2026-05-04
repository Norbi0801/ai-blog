+++
title = "Phantom types in Rust - compile-time constraints with zero runtime cost"
date = 2025-12-03
description = "How PhantomData<T> lets you encode units, permissions, and state machines into the type system - catching bugs at compile time without adding a single byte at runtime."

[taxonomies]
tags = ["rust", "type-system", "design-patterns", "zero-cost-abstractions"]
+++

Rust's type system can do more work than most people let it. One of the underused tools is `PhantomData<T>` - a type that exists only at compile time, takes up zero bytes at runtime, and lets you encode constraints that the compiler enforces for free.

If you've ever mixed up meters and feet, passed a read-only handle to a function that writes, or called `.build()` on an incomplete builder - phantom types prevent all of those at compile time. No runtime checks, no panics, no overhead.

<!-- more -->

## The problem: unused type parameters

Say you're building a typed wrapper around numeric IDs. You want `UserId` and `SessionId` to be different types so you can't accidentally pass one where the other is expected:

```rust
struct Id<T> {
    value: u64,
}
```

Try to compile this and Rust gives you [E0392](https://doc.rust-lang.org/error_codes/E0392.html):

```
error[E0392]: type parameter `T` is never used
 --> src/lib.rs:1:11
  |
1 | struct Id<T> {
  |           ^ unused type parameter
  |
  = help: consider removing `T`, referring to it in a field,
          or using a marker such as `PhantomData`
```

The compiler needs to know how `T` affects the struct's layout, variance, and auto-trait implementations. If `T` isn't used in any field, the compiler can't figure that out. `PhantomData<T>` solves this - it tells the compiler "this type is parameterized over `T`" without actually storing anything.

```rust
use std::marker::PhantomData;

struct Id<T> {
    value: u64,
    _marker: PhantomData<T>,
}

struct User;
struct Session;

type UserId = Id<User>;
type SessionId = Id<Session>;
```

Now `UserId` and `SessionId` are distinct types. Pass a `SessionId` where a `UserId` is expected and the compiler stops you:

```
error[E0308]: mismatched types
  --> src/main.rs:15:18
   |
15 |     get_user(session_id);
   |              ^^^^^^^^^^ expected `Id<User>`, found `Id<Session>`
```

## Zero bytes, zero cost

`PhantomData` is a [zero-sized type](https://doc.rust-lang.org/nomicon/exotic-sizes.html#zero-sized-types-zsts) (ZST). It compiles away completely. You can verify this:

```rust
use std::marker::PhantomData;
use std::mem::size_of;

struct WithPhantom<T> {
    value: u64,
    _marker: PhantomData<T>,
}

struct WithoutPhantom {
    value: u64,
}

fn main() {
    assert_eq!(size_of::<WithPhantom<String>>(), 8);
    assert_eq!(size_of::<WithoutPhantom>(), 8);
    assert_eq!(size_of::<PhantomData<String>>(), 0);
}
```

Both structs are 8 bytes. The `PhantomData` field contributes nothing to layout. If you throw this into [Godbolt](https://rust.godbolt.org/) with `-O`, the `sizes` function compiles down to loading constants into registers - no trace of the phantom field in the assembly.

Looking at the [actual definition in `core::marker`](https://github.com/rust-lang/rust/blob/main/library/core/src/marker.rs), `PhantomData` is a lang item:

```rust
#[lang = "phantom_data"]
pub struct PhantomData<T: ?Sized>;
```

The `#[lang = "phantom_data"]` attribute marks it as special to the compiler. All trait implementations - `Copy`, `Clone`, `Default`, `Hash`, `PartialEq`, `Eq`, `PartialOrd`, `Ord` - are implemented manually rather than derived.

## Use case 1: units that can't be mixed

The Mars Climate Orbiter was lost because one system used pound-seconds and another used newton-seconds. In Rust, phantom types make that category of bug impossible:

```rust
use std::marker::PhantomData;
use std::ops::{Add, Mul};

struct Meters;
struct Feet;
struct Seconds;

#[derive(Debug, Clone, Copy)]
struct Quantity<U> {
    value: f64,
    _unit: PhantomData<U>,
}

impl<U> Quantity<U> {
    fn new(value: f64) -> Self {
        Quantity {
            value,
            _unit: PhantomData,
        }
    }
}

// Addition only works within the same unit
impl<U> Add for Quantity<U> {
    type Output = Quantity<U>;
    fn add(self, rhs: Self) -> Self::Output {
        Quantity::new(self.value + rhs.value)
    }
}

// Explicit conversion between units
impl Quantity<Meters> {
    fn to_feet(self) -> Quantity<Feet> {
        Quantity::new(self.value * 3.28084)
    }
}

impl Quantity<Feet> {
    fn to_meters(self) -> Quantity<Meters> {
        Quantity::new(self.value / 3.28084)
    }
}

fn main() {
    let a = Quantity::<Meters>::new(100.0);
    let b = Quantity::<Meters>::new(50.0);
    let c = a + b; // compiles: both Meters
    println!("Total: {:.1} meters", c.value); // Total: 150.0 meters

    let d = Quantity::<Feet>::new(200.0);
    // a + d; // ERROR: expected Quantity<Meters>, found Quantity<Feet>

    // Conversion must be explicit
    let d_in_meters = d.to_meters();
    let total = a + d_in_meters; // compiles: both Meters now
    println!("Combined: {:.1} meters", total.value);
}
```

The key insight: `Quantity<Meters>` and `Quantity<Feet>` have identical memory layouts. They're both just an `f64`. But the compiler treats them as completely different types. The unit tag exists only in the type system - it's erased before code generation.

If you've worked with the [uom](https://crates.io/crates/uom) crate (units of measurement), it builds on exactly this principle. So does [dimensioned](https://crates.io/crates/dimensioned).

## Use case 2: permission levels

Consider a file handle where you want to statically guarantee that read-only handles can't write:

```rust
use std::marker::PhantomData;
use std::io;

struct ReadOnly;
struct ReadWrite;

struct FileHandle<P> {
    path: String,
    _permission: PhantomData<P>,
}

// Available on ALL handles
impl<P> FileHandle<P> {
    fn read(&self) -> io::Result<Vec<u8>> {
        println!("Reading from {}", self.path);
        Ok(vec![])
    }

    fn path(&self) -> &str {
        &self.path
    }
}

// Only available on ReadOnly
impl FileHandle<ReadOnly> {
    fn open_readonly(path: &str) -> Self {
        FileHandle {
            path: path.to_string(),
            _permission: PhantomData,
        }
    }
}

// Only available on ReadWrite
impl FileHandle<ReadWrite> {
    fn open_readwrite(path: &str) -> Self {
        FileHandle {
            path: path.to_string(),
            _permission: PhantomData,
        }
    }

    fn write(&self, data: &[u8]) -> io::Result<()> {
        println!("Writing {} bytes to {}", data.len(), self.path);
        Ok(())
    }

    fn as_readonly(&self) -> &FileHandle<ReadOnly> {
        // Safe: same layout, just narrowing the permission
        unsafe { &*(self as *const FileHandle<ReadWrite> as *const FileHandle<ReadOnly>) }
    }
}

fn process_readonly(handle: &FileHandle<ReadOnly>) {
    let _ = handle.read(); // OK
    // handle.write(b"nope"); // ERROR: no method `write` on FileHandle<ReadOnly>
}

fn main() {
    let rw = FileHandle::open_readwrite("/tmp/data.txt");
    rw.read().unwrap();  // OK
    rw.write(b"hello").unwrap();  // OK

    let ro = FileHandle::open_readonly("/etc/config");
    ro.read().unwrap();  // OK
    // ro.write(b"hack"); // won't compile

    // Can pass a ReadWrite handle as ReadOnly
    process_readonly(rw.as_readonly());
}
```

The `write` method only exists in the `impl FileHandle<ReadWrite>` block. It's not that calling `write` on a read-only handle returns an error - the method doesn't exist at all. The compiler won't even let you write the call.

This pattern shows up in real code. Diesel uses phantom type parameters extensively throughout its query builder - `Bound<T, U>` carries a `PhantomData<T>` for SQL type information, `BoxedSelectStatement` tracks its backend type through a phantom parameter. The types ensure you can't accidentally build a Postgres query and execute it against SQLite.

## Use case 3: state machines (the typestate pattern)

This is where phantom types really shine. When combined with methods that consume `self` and return a new type, you get state machines checked entirely at compile time:

```rust
use std::marker::PhantomData;

// States
struct New;
struct HasHost;
struct HasPort;
struct Ready;

struct ServerConfig<S> {
    host: Option<String>,
    port: Option<u16>,
    max_conn: usize,
    _state: PhantomData<S>,
}

impl ServerConfig<New> {
    fn new() -> Self {
        ServerConfig {
            host: None,
            port: None,
            max_conn: 100,
            _state: PhantomData,
        }
    }

    fn host(self, host: &str) -> ServerConfig<HasHost> {
        ServerConfig {
            host: Some(host.to_string()),
            port: self.port,
            max_conn: self.max_conn,
            _state: PhantomData,
        }
    }
}

impl ServerConfig<HasHost> {
    fn port(self, port: u16) -> ServerConfig<Ready> {
        ServerConfig {
            host: self.host,
            port: Some(port),
            max_conn: self.max_conn,
            _state: PhantomData,
        }
    }
}

// Optional config available in any state
impl<S> ServerConfig<S> {
    fn max_connections(mut self, n: usize) -> Self {
        self.max_conn = n;
        self
    }
}

struct Server {
    host: String,
    port: u16,
    max_conn: usize,
}

// build() only available in Ready state
impl ServerConfig<Ready> {
    fn build(self) -> Server {
        Server {
            host: self.host.unwrap(),
            port: self.port.unwrap(),
            max_conn: self.max_conn,
        }
    }
}

fn main() {
    // This compiles - correct order
    let server = ServerConfig::new()
        .host("0.0.0.0")
        .max_connections(500)
        .port(8080)
        .build();

    // This won't - missing port
    // let bad = ServerConfig::new()
    //     .host("0.0.0.0")
    //     .build(); // ERROR: no method `build` on ServerConfig<HasHost>

    // This won't - wrong order
    // let bad = ServerConfig::new()
    //     .port(8080); // ERROR: no method `port` on ServerConfig<New>
}
```

Each method consumes `self` (taking ownership) and returns a new `ServerConfig` with a different state parameter. The old value is gone - you can't use a `ServerConfig<New>` after calling `.host()` on it because Rust's ownership system moves it. The state transition is both type-checked and move-checked.

This extends naturally to protocol state machines. Think TCP: a socket in the `Closed` state can `listen()` or `connect()`, but you can only `send()` on an `Established` connection. Model each state as a phantom type and each transition as a consuming method:

```rust
use std::marker::PhantomData;

struct Closed;
struct Listening;
struct Established;

struct TcpSocket<State> {
    fd: i32,
    _state: PhantomData<State>,
}

impl TcpSocket<Closed> {
    fn new() -> Self {
        TcpSocket { fd: 42, _state: PhantomData }
    }

    fn listen(self, port: u16) -> TcpSocket<Listening> {
        println!("Listening on :{}", port);
        TcpSocket { fd: self.fd, _state: PhantomData }
    }

    fn connect(self, addr: &str) -> TcpSocket<Established> {
        println!("Connected to {}", addr);
        TcpSocket { fd: self.fd, _state: PhantomData }
    }
}

impl TcpSocket<Listening> {
    fn accept(&self) -> TcpSocket<Established> {
        println!("Accepted connection");
        TcpSocket { fd: self.fd, _state: PhantomData }
    }
}

impl TcpSocket<Established> {
    fn send(&self, data: &[u8]) -> usize {
        println!("Sent {} bytes", data.len());
        data.len()
    }

    fn recv(&self, buf: &mut [u8]) -> usize {
        println!("Received data");
        0
    }

    fn close(self) -> TcpSocket<Closed> {
        println!("Connection closed");
        TcpSocket { fd: self.fd, _state: PhantomData }
    }
}

fn main() {
    // Server path
    let socket = TcpSocket::new();     // Closed
    let listener = socket.listen(8080); // Listening
    let conn = listener.accept();       // Established
    conn.send(b"hello");

    // Can't send on a listening socket
    // listener.send(b"data"); // ERROR: no method `send` on TcpSocket<Listening>

    // Can't accept on a closed socket
    // TcpSocket::new().accept(); // ERROR: no method `accept` on TcpSocket<Closed>
}
```

Hyper's earlier versions used this exact approach - `Response<Fresh>` vs `Response<Streaming>` tracked whether you could still set headers or had started writing the body.

## Under the hood: variance and drop check

If you only use phantom types for tagging (units, states, permissions), you can stop here and be productive. But `PhantomData` has deeper implications that matter when you're writing unsafe code or building generic containers.

### Variance

Variance determines how subtyping of the parameter affects subtyping of the whole type. If you've read the [trait bounds post](/blog/rust-trait-bounds-where-clauses-associated-types-and-the-rest-of-the-iceberg/), you know Rust's generics are powerful. Variance is the part of the type system that controls how lifetimes propagate through generic types.

The [Rustonomicon's variance table](https://doc.rust-lang.org/nomicon/phantom-data.html) for `PhantomData`:

| You write | Variance over `T` | Send/Sync | Drop check |
|---|---|---|---|
| `PhantomData<T>` | covariant | inherited from T | owns T (strict) |
| `PhantomData<&'a T>` | covariant | needs T: Sync | no ownership |
| `PhantomData<*const T>` | covariant | !Send + !Sync | no ownership |
| `PhantomData<*mut T>` | invariant | !Send + !Sync | no ownership |
| `PhantomData<fn(T)>` | contravariant | Send + Sync | no ownership |
| `PhantomData<fn() -> T>` | covariant | Send + Sync | no ownership |

The critical distinction most people miss: **`PhantomData<T>` and `PhantomData<fn() -> T>` are both covariant over `T`, but they behave differently**.

`PhantomData<T>` tells the compiler "I own a `T`." This means:
- Your type is `Send` only if `T: Send`
- Your type is `Sync` only if `T: Sync`
- The drop checker assumes your `Drop` impl might access `T`

`PhantomData<fn() -> T>` tells the compiler "I can produce a `T`, but I don't own one." This means:
- Your type is always `Send + Sync` regardless of `T`
- The drop checker doesn't assume ownership

When does this matter? If you're building a phantom-tagged type where `T` is just a marker (like our `Meters` or `ReadOnly` structs), both work fine because marker types are typically `Send + Sync` anyway. But if `T` could be any type - say you're building a generic container with unsafe internals - the choice matters.

The standard library's `Vec<T>` internally uses `PhantomData<T>` (via `Unique<T>`) because it genuinely owns `T` values. A type like `NonNull<T>` that holds a raw pointer but doesn't own the data uses `PhantomData<T>` anyway because it wants to opt into drop check for safety.

### The `#[may_dangle]` connection

There's an unstable attribute `#[may_dangle]` used in `Vec`'s `Drop` impl:

```rust
unsafe impl<#[may_dangle] T, A: Allocator> Drop for Vec<T, A> {
    fn drop(&mut self) { /* ... */ }
}
```

This tells the compiler "my destructor won't access `T` directly." But `Vec` still needs the drop checker to know it conceptually owns `T` values (because it drops them via `ptr::drop_in_place`). That's why `Vec` keeps `PhantomData<T>` - it reinstates the ownership signal that `#[may_dangle]` relaxes.

You won't need this unless you're writing your own allocator-aware collections. But understanding it explains why the variance table exists.

### Nightly: explicit variance markers

The situation with `PhantomData<fn() -> T>` being the idiom for "covariant but not owning" is admittedly confusing. There's an [open tracking issue (#135806)](https://github.com/rust-lang/rust/issues/135806) for new explicit types behind `#![feature(phantom_variance_markers)]`:

```rust
use std::marker::PhantomCovariant;
use std::marker::PhantomContravariant;
use std::marker::PhantomInvariant;
```

These are self-documenting: `PhantomCovariant<T>` replaces `PhantomData<fn() -> T>`, `PhantomContravariant<T>` replaces `PhantomData<fn(T)>`, etc. All are zero-sized, all are `Send + Sync`. Not stabilized yet, but the direction is clear - the ecosystem recognizes that `PhantomData<fn() -> T>` is a footgun for readability.

## Phantom types vs newtype vs enum

Three patterns solve overlapping problems. Here's when to reach for each:

### Newtypes

```rust
struct Meters(f64);
struct Feet(f64);
```

Newtypes wrap a value and give it a new identity. They're the right choice when:

- You need a different type but want to add **methods or trait impls** specific to that wrapper
- You're working around the [orphan rule](https://doc.rust-lang.org/reference/items/implementations.html#orphan-rules) (implementing a foreign trait on a foreign type)
- You have a **small, fixed set** of variants (2-3 distinct types)

Downside: each newtype is a completely separate type. If you have 20 unit types, you write 20 structs with duplicated method signatures. With phantom types, you write one generic struct and `impl` blocks for specific tags.

### Enums

```rust
enum Unit { Meters, Feet, Kilometers }

struct Distance {
    value: f64,
    unit: Unit,
}
```

Enums are the right choice when:

- The variant needs to be **determined at runtime** (user input, config files, deserialization)
- You need to **store mixed variants in the same collection** (`Vec<Distance>`)
- The set of variants might **change over the lifetime of a value**
- You want to **serialize** the variant (serde handles enums naturally)

Downside: invalid combinations are runtime errors, not compile errors. Nothing stops you from adding `Meters` to `Feet` - you'd need a runtime check.

### Phantom types

```rust
struct Distance<U> {
    value: f64,
    _unit: PhantomData<U>,
}
```

Phantom types are the right choice when:

- The variant is **known at compile time** and never changes
- You want the compiler to **reject invalid combinations** (adding meters to feet)
- You have **many variants** that share the same struct layout and most methods
- You want **zero runtime cost** - no discriminant byte, no match arms, no branching

Downside: you can't store `Distance<Meters>` and `Distance<Feet>` in the same `Vec` without boxing or using an enum wrapper. The types are fully erased at runtime, so there's no way to inspect what unit a value has through reflection.

### Decision table

| Criteria | Newtype | Enum | Phantom type |
|---|---|---|---|
| Variant known at compile time | yes | either | yes |
| Variant can change at runtime | no | yes | no |
| Zero runtime cost | yes | no (discriminant) | yes |
| Mixed variants in one collection | no | yes | no |
| Scales to many variants | poorly | ok | well |
| Compile-time safety | partial | none | full |
| Serde-friendly | yes | yes | needs custom impl |

## Real-world patterns

### Entity IDs that can't be confused

One of the most practical everyday uses - stop mixing up string IDs:

```rust
use std::marker::PhantomData;

#[derive(Debug, Clone, PartialEq, Eq, Hash)]
struct Id<T> {
    value: String,
    _entity: PhantomData<T>,
}

impl<T> Id<T> {
    fn new(value: impl Into<String>) -> Self {
        Id {
            value: value.into(),
            _entity: PhantomData,
        }
    }

    fn as_str(&self) -> &str {
        &self.value
    }
}

struct User;
struct Order;
struct Product;

fn get_user_orders(user_id: &Id<User>) -> Vec<Id<Order>> {
    // The compiler won't let you pass an Order ID here
    vec![]
}

fn main() {
    let user_id = Id::<User>::new("usr_abc123");
    let order_id = Id::<Order>::new("ord_xyz789");

    get_user_orders(&user_id); // OK
    // get_user_orders(&order_id); // ERROR: expected &Id<User>, found &Id<Order>
}
```

This is worth adopting in any codebase with more than two entity types. The number of bugs caused by passing the wrong ID string to a database query is... non-trivial.

### Sealed phantom states

Sometimes you want to restrict which types can be used as phantom parameters. Combine phantom types with a [sealed trait](https://rust-lang.github.io/api-guidelines/future-proofing.html#sealed-traits-protect-against-downstream-implementations-c-sealed):

```rust
use std::marker::PhantomData;

mod sealed {
    pub trait Permission {}
}

pub struct ReadOnly;
pub struct ReadWrite;
pub struct Admin;

impl sealed::Permission for ReadOnly {}
impl sealed::Permission for ReadWrite {}
impl sealed::Permission for Admin {}

pub struct Handle<P: sealed::Permission> {
    inner: u64,
    _perm: PhantomData<P>,
}

// Users can't create Handle<SomeRandomType> because
// SomeRandomType doesn't implement the sealed Permission trait
```

This prevents users of your API from inventing new permission levels. The sealed trait module isn't publicly accessible, so only your crate can implement `Permission`. If you've read the [semver post](/blog/semantic-versioning-what-breaking-changes-actually-means-in-rust/), you know sealed traits are also useful for maintaining backwards compatibility.

### Combining with generics

Phantom types compose well with trait bounds. If you covered the [trait bounds post](/blog/rust-trait-bounds-where-clauses-associated-types-and-the-rest-of-the-iceberg/), you already know about `where` clauses and associated types. Here's how they interact:

```rust
use std::marker::PhantomData;

trait Codec {
    fn content_type() -> &'static str;
}

struct Json;
struct Xml;
struct Protobuf;

impl Codec for Json {
    fn content_type() -> &'static str { "application/json" }
}
impl Codec for Xml {
    fn content_type() -> &'static str { "application/xml" }
}
impl Codec for Protobuf {
    fn content_type() -> &'static str { "application/protobuf" }
}

struct Response<C: Codec> {
    body: Vec<u8>,
    _codec: PhantomData<C>,
}

impl<C: Codec> Response<C> {
    fn new(body: Vec<u8>) -> Self {
        Response { body, _codec: PhantomData }
    }

    fn content_type(&self) -> &'static str {
        C::content_type()
    }
}

fn send_json(resp: &Response<Json>) {
    assert_eq!(resp.content_type(), "application/json");
}

fn main() {
    let resp = Response::<Json>::new(b"{}".to_vec());
    send_json(&resp); // OK

    let xml_resp = Response::<Xml>::new(b"<root/>".to_vec());
    // send_json(&xml_resp); // ERROR: expected Response<Json>, found Response<Xml>
}
```

This pattern is similar to the [strategy pattern](/blog/the-strategy-pattern-in-rust-polymorphism-done-right/) - but instead of choosing the strategy at runtime through dynamic dispatch or enum matching, you lock it in at compile time through the type parameter.

## When phantom types are the wrong tool

Not every problem needs type-level encoding. Phantom types add complexity to your API surface. Avoid them when:

- **States change at runtime based on user input.** If a file might be read-only or read-write depending on a config value, you need an enum or a trait object. Phantom types require the state to be known at compile time.

- **You need mixed collections.** `Vec<Distance<Meters>>` works, but you can't mix `Distance<Meters>` and `Distance<Feet>` in one `Vec` without `Box<dyn SomeTrait>` or an enum wrapper.

- **The type parameter explosion is getting out of hand.** `Builder<HasHost, HasPort, HasTls, HasTimeout, HasAuth>` with five phantom parameters and 32 possible states - at that point, consider a runtime validation approach or a proc macro.

- **Your team isn't familiar with the pattern.** Phantom types produce confusing error messages for developers who haven't seen them before. `expected ServerConfig<HasHost>`, found `ServerConfig<New>` makes sense when you know the pattern, but it's opaque otherwise.

A good rule of thumb: use phantom types for 2-4 states on a hot path or safety-critical boundary. Use enums for everything else.

## The assembly proof

To convince yourself this is truly zero-cost, here's a minimal comparison you can paste into [Godbolt](https://rust.godbolt.org/):

```rust
use std::marker::PhantomData;

struct Meters;
struct Feet;

struct Dist<U> {
    val: f64,
    _u: PhantomData<U>,
}

struct PlainDist {
    val: f64,
}

#[no_mangle]
pub fn add_phantom(a: Dist<Meters>, b: Dist<Meters>) -> Dist<Meters> {
    Dist { val: a.val + b.val, _u: PhantomData }
}

#[no_mangle]
pub fn add_plain(a: PlainDist, b: PlainDist) -> PlainDist {
    PlainDist { val: a.val + b.val }
}
```

Both functions produce identical x86-64 assembly under `-O`:

```asm
add_phantom:
    addsd   xmm0, xmm1
    ret

add_plain:
    addsd   xmm0, xmm1
    ret
```

One `addsd` instruction. The phantom type parameter, the `PhantomData` field, the generic instantiation - all erased. The compiler monomorphizes `Dist<Meters>` into a plain struct with one `f64` field, then optimizes as usual.

## Wrapping up

Phantom types are one of those features that feel like they shouldn't work - a type parameter that doesn't correspond to any data, enforcing constraints that exist only in the compiler's imagination. But they do work, and they're free.

The pattern is simple: define empty marker structs, use them as type parameters, gate your methods with specific `impl` blocks. The compiler does the rest. No runtime checks, no discriminant bytes, no match arms.

Start with entity IDs. If you have `fn get_user(id: String)` anywhere in your codebase, replacing that `String` with `Id<User>` is a five-minute change that prevents a whole class of bugs. Once you see the value, the other patterns - units, permissions, state machines - follow naturally.
