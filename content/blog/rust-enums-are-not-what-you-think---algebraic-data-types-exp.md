+++
title = "Rust enums are not what you think - algebraic data types explained"
date = 2026-01-23
description = "Rust enums are algebraic data types (sum types) where each variant carries different data - here is what that means, how the compiler lays them out in memory, and why it eliminates entire classes of bugs."

[taxonomies]
tags = ["rust", "type-system", "memory-layout", "compiler"]
+++

If you're coming from C, C#, Java, or Go, you probably think of an enum as a named integer. A list of constants. Monday is 0, Tuesday is 1, Wednesday is 2. That's it.

Rust's `enum` keyword reuses the name but means something fundamentally different. A Rust enum is a **sum type** - an algebraic data type where a value can be one of several variants, and each variant can carry completely different data. This isn't a cosmetic difference. It changes how you model problems, how you handle errors, and how entire categories of bugs simply stop existing.

<!-- more -->

## C enums: named integers

Let's start with what most languages call an enum:

```c
enum Color {
    Red,    // 0
    Green,  // 1
    Blue,   // 2
};
```

That's it. `Color` is an integer. You can assign `42` to it and the compiler won't blink. There's no associated data, no type safety beyond "this is an int", and no mechanism to attach meaning to each variant beyond a comment.

In Go, it's the same idea with `iota`. In Java, enums are slightly fancier objects, but the core concept is a fixed set of named values.

Rust has this too:

```rust
enum Color {
    Red,
    Green,
    Blue,
}
```

Looks the same. But this is just the degenerate case. The real power shows up when variants carry data.

## Variants that carry data

Here's where Rust enums diverge from everything you know:

```rust
enum Shape {
    Circle { radius: f64 },
    Rectangle { width: f64, height: f64 },
    Triangle { a: f64, b: f64, c: f64 },
}
```

Each variant has a different number of fields with different types. A `Shape` value is *one of* these three - and the data it carries depends on which variant it is. You can't have a `Circle` with a `width` field or a `Rectangle` with three side lengths. The type system enforces this.

Variants can carry data in three forms:

```rust
enum Message {
    Quit,                              // no data (unit variant)
    Move { x: i32, y: i32 },          // named fields (struct variant)
    Write(String),                     // positional fields (tuple variant)
    ChangeColor(u8, u8, u8),           // multiple positional fields
}
```

This is not "an int with extra steps". This is a **tagged union** - a type that can hold different kinds of data at different times, with the compiler tracking which kind it currently holds.

## The type theory perspective: sum types and product types

In type theory, there are two fundamental ways to compose types:

**Product types** combine types with AND. A struct `(A, B)` holds a value of type A *and* a value of type B. The number of possible values is the product: `|A| * |B|`. A `(bool, bool)` has `2 * 2 = 4` possible values.

**Sum types** combine types with OR. An enum `A | B` holds a value of type A *or* a value of type B. The number of possible values is the sum: `|A| + |B|`. An `Either<bool, bool>` has `2 + 2 = 4` possible values.

Together, these are called **algebraic data types** (ADTs) because you can reason about them with algebra. Rust structs are product types. Rust enums are sum types. Having both gives you a complete algebra for building data structures.

This matters in practice. Consider modeling a network connection:

```rust
enum Connection {
    Disconnected,
    Connecting { address: SocketAddr, attempt: u32 },
    Connected { stream: TcpStream, connected_at: Instant },
    Failed { error: io::Error, last_address: SocketAddr },
}
```

Each state has exactly the data that makes sense for that state. You can't access the `stream` when you're `Disconnected`. You can't read the `error` when you're `Connected`. The compiler won't let you. Compare this to the typical OOP approach: a single struct with all fields optional, and runtime checks everywhere to figure out which combination of `None`s and `Some`s you're actually in.

## Option and Result: enums hiding in plain sight

The two most-used types in Rust are enums:

```rust
// From the standard library
enum Option<T> {
    None,
    Some(T),
}

enum Result<T, E> {
    Ok(T),
    Err(E),
}
```

`Option<T>` replaces null pointers. Instead of a pointer that might be null (and will crash at runtime if you guess wrong), you get a type that *forces* you to handle both cases. There is no way to access the inner value without first checking whether it exists.

`Result<T, E>` replaces exceptions and error codes. Instead of an integer return value that you might forget to check (looking at you, C), or an exception that might fly past 12 stack frames uncaught (looking at you, Java), you get a type where the error case is part of the value.

Tony Hoare called null pointers his ["billion dollar mistake"](https://en.wikipedia.org/wiki/Null_pointer). `Option<T>` is the fix. Not a convention. Not a linting rule. A type-level guarantee that you cannot dereference something that doesn't exist.

## Pattern matching: exhaustive by default

Here's the key mechanic that makes sum types practical: pattern matching with exhaustiveness checking.

```rust
fn area(shape: &Shape) -> f64 {
    match shape {
        Shape::Circle { radius } => std::f64::consts::PI * radius * radius,
        Shape::Rectangle { width, height } => width * height,
        Shape::Triangle { a, b, c } => {
            let s = (a + b + c) / 2.0;
            (s * (s - a) * (s - b) * (s - c)).sqrt()
        }
    }
}
```

The `match` expression is **exhaustive** - the compiler verifies that every possible variant is handled. If you add `Shape::Polygon { sides: Vec<f64> }` tomorrow, every `match` on `Shape` in your entire codebase will fail to compile until you handle the new case. Not at runtime. Not when a user hits the code path. At compile time.

This is what makes Rust enums eliminate whole categories of bugs. In C, if you add a new value to an enum and forget to update a switch statement, the default branch runs (or worse, there is no default and you get undefined behavior). In Rust, you get a compiler error.

Patterns can also destructure nested data, bind variables, use guards, and match on ranges:

```rust
fn describe(msg: &Message) -> String {
    match msg {
        Message::Quit => "quitting".to_string(),
        Message::Move { x, y } if *x == 0 && *y == 0 => "staying put".to_string(),
        Message::Move { x, y } => format!("moving to ({}, {})", x, y),
        Message::Write(text) if text.is_empty() => "writing nothing".to_string(),
        Message::Write(text) => format!("writing: {}", text),
        Message::ChangeColor(r, g, b) => format!("color: #{:02x}{:02x}{:02x}", r, g, b),
    }
}
```

If you've used `match` in the context of [enum dispatch vs trait objects](/blog/the-strategy-pattern-in-rust-polymorphism-done-right), you already know the performance characteristics. But here the benefit is correctness, not speed.

## How other languages solve this (or don't)

### C: tagged unions, manually

C doesn't have sum types. You fake them with a struct containing a tag and a union:

```c
typedef enum { CIRCLE, RECTANGLE, TRIANGLE } ShapeTag;

typedef struct {
    ShapeTag tag;
    union {
        struct { double radius; } circle;
        struct { double width; double height; } rectangle;
        struct { double a, b, c; } triangle;
    } data;
} Shape;

double area(Shape* s) {
    switch (s->tag) {
        case CIRCLE:
            return 3.14159 * s->data.circle.radius * s->data.circle.radius;
        case RECTANGLE:
            return s->data.rectangle.width * s->data.rectangle.height;
        case TRIANGLE: {
            double a = s->data.triangle.a, b = s->data.triangle.b, c = s->data.triangle.c;
            double p = (a + b + c) / 2.0;
            return sqrt(p * (p-a) * (p-b) * (p-c));
        }
        // Forgot to add a case? Compiles fine. UB at runtime.
    }
    return 0.0; // "just in case"
}
```

Two problems. First, nothing stops you from reading `s->data.circle.radius` when the tag is `RECTANGLE`. The compiler doesn't know the tag and the union are related. Second, `switch` isn't exhaustive - if you add `POLYGON` to the enum and forget to update this function, it compiles. The `return 0.0` silently swallows the bug.

This is the source of real vulnerabilities. The compiler cannot verify that you always check the tag before reading the data. Rust's enum makes this structurally impossible.

### TypeScript: discriminated unions

TypeScript gets surprisingly close:

```typescript
type Shape =
    | { kind: "circle"; radius: number }
    | { kind: "rectangle"; width: number; height: number }
    | { kind: "triangle"; a: number; b: number; c: number };

function area(shape: Shape): number {
    switch (shape.kind) {
        case "circle":
            return Math.PI * shape.radius ** 2;
        case "rectangle":
            return shape.width * shape.height;
        case "triangle": {
            const s = (shape.a + shape.b + shape.c) / 2;
            return Math.sqrt(s * (s - shape.a) * (s - shape.b) * (s - shape.c));
        }
    }
}
```

With `strictNullChecks` and `noImplicitReturns` enabled, TypeScript can enforce exhaustiveness. The narrowing in each `case` branch correctly restricts which fields are accessible. It works.

But there are gaps. The discriminant field (`kind`) is a stringly-typed convention, not a language primitive. Each variant is a separate type you manage yourself. And since it all compiles to JavaScript, the discriminant is a runtime string comparison, not a zero-cost tag. You also have to manually maintain the union type when adding variants.

Rust enums do this with less boilerplate, zero runtime overhead for the discriminant, and the guarantee is structural, not opt-in.

### Kotlin: sealed classes

Kotlin's `sealed class` is the OOP world's answer to sum types:

```kotlin
sealed class Shape {
    data class Circle(val radius: Double) : Shape()
    data class Rectangle(val width: Double, val height: Double) : Shape()
    data class Triangle(val a: Double, val b: Double, val c: Double) : Shape()
}

fun area(shape: Shape): Double = when (shape) {
    is Shape.Circle -> Math.PI * shape.radius.pow(2)
    is Shape.Rectangle -> shape.width * shape.height
    is Shape.Triangle -> {
        val s = (shape.a + shape.b + shape.c) / 2
        sqrt(s * (s - shape.a) * (s - shape.b) * (s - shape.c))
    }
}
```

`when` is exhaustive when used as an expression (returning a value). The compiler knows all subclasses of a sealed class because they must be defined in the same file. Smart casts narrow the type in each branch.

This is genuinely good. But sealed classes are heap-allocated objects with vtable pointers. Each "variant" is a separate class on the heap. In Rust, all variants live in the same stack-allocated value, and the tag is an integer (often optimized away entirely).

## Memory layout: what the compiler actually does

Now the fun part. How does the compiler represent `Shape` in memory?

A Rust enum is laid out as a **tagged union**: an integer discriminant (the tag) followed by a union of all variant payloads. The size equals the discriminant plus the largest variant, plus alignment padding.

Let's look at concrete numbers:

```rust
use std::mem::size_of;

enum Shape {
    Circle { radius: f64 },                        // 8 bytes payload
    Rectangle { width: f64, height: f64 },          // 16 bytes payload
    Triangle { a: f64, b: f64, c: f64 },            // 24 bytes payload
}

fn main() {
    println!("Shape: {} bytes", size_of::<Shape>());  // 32 bytes
    println!("f64:   {} bytes", size_of::<f64>());     // 8 bytes
}
```

`Shape` is 32 bytes: 8 bytes for the discriminant (padded to the alignment of `f64`), plus 24 bytes for the largest variant (`Triangle`). When the value is a `Circle`, 16 of those 24 bytes are unused padding. That's the tradeoff of a union - every value occupies the space of the largest variant.

The discriminant itself is an integer. For the default `repr(Rust)` layout, the compiler picks the smallest integer that fits the number of variants. Three variants need at least 2 bits, so a `u8` suffices. But the compiler may use more for alignment reasons.

You can control this explicitly:

```rust
#[repr(u8)]
enum PackedShape {
    Circle { radius: f64 },
    Rectangle { width: f64, height: f64 },
    Triangle { a: f64, b: f64, c: f64 },
}
```

With `#[repr(u8)]`, the discriminant is guaranteed to be 1 byte, and the layout follows the [documented rules in the Rust Reference](https://doc.rust-lang.org/reference/type-layout.html). If you're doing FFI with C code, `#[repr(C)]` gives you the same layout as a C tagged union - I wrote about how `#[non_exhaustive]` interacts with evolving enum APIs in [the semver post](/blog/semantic-versioning-what-breaking-changes-actually-means-in-rust).

## Niche optimization: where it gets clever

Here's the part that impresses me most about rustc. Consider `Option<&T>`:

```rust
use std::mem::size_of;

fn main() {
    println!("&u8:           {} bytes", size_of::<&u8>());           // 8
    println!("Option<&u8>:   {} bytes", size_of::<Option<&u8>>());   // 8
}
```

Both are 8 bytes. `Option<&u8>` doesn't need a separate discriminant. Why? Because `&T` in Rust is guaranteed to be non-null. That means the bit pattern `0x0000000000000000` (null) is never a valid `&T`. The compiler uses that forbidden bit pattern to represent `None`.

This is **niche optimization** (sometimes called null pointer optimization or NPO). The compiler finds "niches" - bit patterns that a type guarantees it will never use - and repurposes them as discriminant values.

It works for more than just references:

```rust
use std::num::NonZeroU64;
use std::mem::size_of;

fn main() {
    println!("u64:               {} bytes", size_of::<u64>());               // 8
    println!("Option<u64>:       {} bytes", size_of::<Option<u64>>());       // 16
    println!("NonZeroU64:        {} bytes", size_of::<NonZeroU64>());        // 8
    println!("Option<NonZeroU64>:{} bytes", size_of::<Option<NonZeroU64>>()); // 8
}
```

`Option<u64>` is 16 bytes - it needs a discriminant because every bit pattern is a valid `u64`. But `NonZeroU64` can never be zero, so `Option<NonZeroU64>` is 8 bytes. The zero pattern represents `None`, and any other pattern is `Some(value)`.

This is a zero-cost abstraction in the truest sense. `Option<&T>` has the exact same runtime representation as a nullable pointer in C. But unlike C, the compiler forces you to check for null before dereferencing. Same performance, strictly more safety.

The niche optimization is recursive. The compiler walks the type structure, asking each nested type "what niches do you have?" and bubbles the answers up:

```rust
use std::mem::size_of;

fn main() {
    // bool has a niche: only 0 and 1 are valid, so 2-255 are available
    println!("bool:           {} bytes", size_of::<bool>());           // 1
    println!("Option<bool>:   {} bytes", size_of::<Option<bool>>());   // 1

    // Nested option - the inner Option<&u8> uses null for None,
    // but the outer Option needs its own discriminant
    println!("Option<Option<bool>>: {} bytes",
             size_of::<Option<Option<bool>>>());                       // 1
}
```

`Option<bool>` is 1 byte because `bool` only uses values `0` and `1`, leaving 254 niches for the discriminant. The compiler uses one niche for `None`, but there are still 253 unused bit patterns left. This means `Option<Option<bool>>` is *also* just 1 byte - the outer `Option` grabs another unused bit pattern from the remaining 253 niches. The optimization cascades as deep as niches allow.

The `NonZero` types from `std::num` exist precisely to give the compiler these niches. If you're building a data structure where IDs are never zero, using `NonZeroU32` instead of `u32` means your `Option<NonZeroU32>` costs nothing extra. [The niche mechanism](https://www.0xatticus.com/posts/understanding_rust_niche/) is implemented via the internal `#[rustc_layout_scalar_valid_range_start]` attribute that tells the compiler which bit patterns are invalid for a type.

## Recursive enums

Enums can reference themselves, which is how you build tree structures and ASTs:

```rust
enum Expr {
    Literal(f64),
    Add(Box<Expr>, Box<Expr>),
    Mul(Box<Expr>, Box<Expr>),
    Neg(Box<Expr>),
}

fn eval(expr: &Expr) -> f64 {
    match expr {
        Expr::Literal(n) => *n,
        Expr::Add(a, b) => eval(a) + eval(b),
        Expr::Mul(a, b) => eval(a) * eval(b),
        Expr::Neg(e) => -eval(e),
    }
}

fn main() {
    // Represents: -(2.0 + 3.0) * 4.0
    let expr = Expr::Mul(
        Box::new(Expr::Neg(
            Box::new(Expr::Add(
                Box::new(Expr::Literal(2.0)),
                Box::new(Expr::Literal(3.0)),
            )),
        )),
        Box::new(Expr::Literal(4.0)),
    );
    println!("{}", eval(&expr)); // -20.0
}
```

The `Box` is necessary because without it, `Expr` would be infinitely sized - each `Add` contains two `Expr`s, which might contain more `Expr`s. `Box` provides a level of indirection (a heap pointer), making the size finite. If you've read the [Markdown parser post](/blog/writing-a-markdown-parser-in-rust), you've seen this pattern in action with recursive block and inline elements.

And here's the niche optimization paying off again: `Box<T>` is guaranteed non-null, so `Option<Box<Expr>>` is the same size as `Box<Expr>`. If your recursive enum has an optional child, it's free.

## Implementing traits on enums

Enums can implement traits just like structs, and `match` is how you dispatch to variant-specific behavior:

```rust
use std::fmt;

enum LogLevel {
    Debug,
    Info,
    Warn,
    Error,
}

impl fmt::Display for LogLevel {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        match self {
            LogLevel::Debug => write!(f, "DEBUG"),
            LogLevel::Info  => write!(f, "INFO"),
            LogLevel::Warn  => write!(f, "WARN"),
            LogLevel::Error => write!(f, "ERROR"),
        }
    }
}
```

This is the foundation of [enum dispatch](/blog/the-strategy-pattern-in-rust-polymorphism-done-right) - using `match` instead of dynamic dispatch (`dyn Trait`) when you know all variants at compile time. The compiler can inline each branch, while a vtable call requires an indirect jump.

## Methods on enums

You can define `impl` blocks on enums just like on structs:

```rust
enum Temperature {
    Celsius(f64),
    Fahrenheit(f64),
    Kelvin(f64),
}

impl Temperature {
    fn to_celsius(&self) -> f64 {
        match self {
            Temperature::Celsius(c) => *c,
            Temperature::Fahrenheit(f) => (f - 32.0) * 5.0 / 9.0,
            Temperature::Kelvin(k) => k - 273.15,
        }
    }

    fn is_freezing(&self) -> bool {
        self.to_celsius() <= 0.0
    }

    fn is_boiling(&self) -> bool {
        self.to_celsius() >= 100.0
    }
}
```

This gives you a single type that handles unit conversion without any intermediate conversions or wrapper types. The `Temperature` value carries both the magnitude and the unit, and you can't mix them up - you always know which unit you're in.

## What bugs this eliminates

Let's be concrete about the classes of bugs that Rust enums prevent.

**1. Accessing wrong union member (C)**

In C, reading the wrong field of a union is undefined behavior. The compiler doesn't track which variant is active. Rust makes this structurally impossible - `match` gives you exactly the fields of the matched variant, nothing else.

**2. Forgetting to handle a case**

Switch statements in C, JavaScript, and most languages are not exhaustive. Missing cases silently fall through to default or cause undefined behavior. Rust's `match` is exhaustive. Adding a variant is a compile error everywhere it's unhandled.

**3. Null pointer dereferences**

The entire category of "forgot to check for null" disappears when `Option<T>` replaces nullable pointers. There's no way to call methods on the inner value without unwrapping the `Option` first - and unwrapping forces you to handle the `None` case (or explicitly panic with `.unwrap()`).

**4. Unchecked error codes**

In C, functions return `int` where `-1` means error. Nobody checks. In Rust, `Result<T, E>` makes the error a different variant. You can't access the success value without matching on `Ok` first. And `#[must_use]` on `Result` means ignoring a `Result` produces a compiler warning.

**5. Invalid state representations**

With a flat struct, nothing prevents `is_connected: true` and `socket: None` from coexisting. With an enum, `Connected { socket: TcpStream }` and `Disconnected` are distinct variants - the impossible state is irrepresentable.

This is what "make illegal states unrepresentable" means in practice. It's not a design philosophy you aspire to. It's a feature of the type system you use.

## When not to use enums

Enums aren't always the right tool:

- **Open-ended extension**: if other crates need to add variants, use a trait instead. Enums are closed sets. You can mitigate this with `#[non_exhaustive]`, but consumers still can't add their own variants.
- **Shared behavior with different internal state**: if every variant implements the same interface and you need dynamic dispatch, `dyn Trait` or generics may be cleaner.
- **Large variant size disparity**: if one variant is 8 bytes and another is 2KB, every value occupies 2KB. Consider `Box`ing the large variant: `LargeVariant(Box<BigStruct>)`.

The [strategy pattern comparison](/blog/the-strategy-pattern-in-rust-polymorphism-done-right) covers the tradeoffs between enum dispatch, generics, and trait objects in depth.

## Putting it all together

Here's a real-world-ish example that ties together everything - a command parser for a key-value store:

```rust
use std::collections::HashMap;
use std::num::NonZeroUsize;

enum Command {
    Get { key: String },
    Set { key: String, value: String, ttl: Option<NonZeroUsize> },
    Delete { key: String },
    List { prefix: Option<String>, limit: Option<NonZeroUsize> },
    Ping,
}

struct Store {
    data: HashMap<String, String>,
}

impl Store {
    fn new() -> Self {
        Store { data: HashMap::new() }
    }

    fn execute(&mut self, cmd: Command) -> String {
        match cmd {
            Command::Get { key } => {
                match self.data.get(&key) {
                    Some(value) => value.clone(),
                    None => format!("(nil) key '{}' not found", key),
                }
            }
            Command::Set { key, value, ttl } => {
                let msg = match ttl {
                    Some(seconds) => format!("OK (expires in {}s)", seconds),
                    None => "OK".to_string(),
                };
                self.data.insert(key, value);
                msg
            }
            Command::Delete { key } => {
                if self.data.remove(&key).is_some() {
                    "(integer) 1".to_string()
                } else {
                    "(integer) 0".to_string()
                }
            }
            Command::List { prefix, limit } => {
                let iter = self.data.keys()
                    .filter(|k| match &prefix {
                        Some(p) => k.starts_with(p.as_str()),
                        None => true,
                    });

                let keys: Vec<_> = match limit {
                    Some(n) => iter.take(n.get()).collect(),
                    None => iter.collect(),
                };

                if keys.is_empty() {
                    "(empty list)".to_string()
                } else {
                    keys.iter()
                        .enumerate()
                        .map(|(i, k)| format!("{}) \"{}\"", i + 1, k))
                        .collect::<Vec<_>>()
                        .join("\n")
                }
            }
            Command::Ping => "PONG".to_string(),
        }
    }
}
```

Every command variant carries exactly the data it needs. `Get` has a key. `Set` has a key, value, and optional TTL. `Ping` has nothing. The `match` in `execute` is exhaustive - adding a new command forces you to handle it. And `Option<NonZeroUsize>` for the TTL and limit is niche-optimized to the size of a single `usize`.

No null checks. No invalid states. No forgotten cases. The types do the work.

## Wrapping up

Rust's `enum` isn't an upgrade to C enums. It's a different construct that happens to share a keyword. It's closer to Haskell's data types, OCaml's variants, or Swift's enums with associated values than to anything in C, Java, or Go.

The combination of sum types, exhaustive pattern matching, and niche optimization gives you:

- **Modeling precision** - each state has exactly the right data
- **Compile-time safety** - impossible states don't compile
- **Zero-cost abstractions** - `Option<&T>` is the same size as a nullable pointer
- **Forced error handling** - `Result<T, E>` makes errors part of the type

Once you internalize this, you start seeing enums everywhere. State machines, protocol messages, AST nodes, configuration variants, command types. They're the backbone of Rust's approach to correctness - not through runtime checks or discipline, but through types that make the wrong thing impossible to write.
