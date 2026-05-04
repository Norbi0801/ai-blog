+++
title = "The newtype pattern in Rust - type safety without runtime cost"
date = 2026-03-14
description = "How wrapping a type in a single-field struct catches bugs at compile time, costs nothing at runtime, and why C's typedef can't do the same."

[taxonomies]
tags = ["rust", "type-system", "design-patterns", "performance"]
+++

You have a function that takes a user ID and a product ID. Both are `u64`. You swap the arguments by accident. The code compiles, the tests that don't cover this path pass, and production silently charges user 7291 for product 4822's order - except 4822 is a user ID and 7291 is a product ID.

```rust
fn purchase(user_id: u64, product_id: u64, quantity: u32) {
    // ...
}

let user = 4822_u64;
let product = 7291_u64;

// Whoops - swapped. Compiles fine.
purchase(product, user, 1);
```

This class of bug exists in every language that relies on primitive types for domain concepts. The newtype pattern eliminates it entirely - at compile time, with zero runtime cost.

<!-- more -->

## The pattern

A newtype is a single-field tuple struct that wraps an existing type:

```rust
struct UserId(u64);
struct ProductId(u64);
```

That's it. Two lines. Now the compiler won't let you mix them up:

```rust
fn purchase(user_id: UserId, product_id: ProductId, quantity: u32) {
    // ...
}

let user = UserId(4822);
let product = ProductId(7291);

// purchase(product, user, 1);
//          ^^^^^^^ expected `UserId`, found `ProductId`
```

The compiler catches the bug before you even run the program. No tests needed for this case, no runtime checks, no performance penalty.

## What it costs at runtime: nothing

If you've read the [closures post](/blog/closures-in-rust-fn-fnmut-fnonce-demystified), you've seen how Rust turns closures into structs that compile down to the same code as hand-written function calls. Newtypes work the same way - the wrapper is erased completely by the optimizer.

Here's the proof. Take these two functions:

```rust
pub struct UserId(u64);

#[no_mangle]
pub fn add_one_raw(x: u64) -> u64 {
    x + 1
}

#[no_mangle]
pub fn add_one_newtype(x: UserId) -> UserId {
    UserId(x.0 + 1)
}
```

Compile with `--release` (or paste into [Godbolt](https://rust.godbolt.org/) with `-C opt-level=2`) and you get identical assembly for both:

```asm
add_one_raw:
        lea     rax, [rdi+1]
        ret

add_one_newtype:
        lea     rax, [rdi+1]
        ret
```

Same instruction, same registers, same everything. The `UserId` wrapper doesn't exist in the binary. The compiler sees a struct with one `u64` field and passes it in a register, exactly like a bare `u64`.

You can also verify the memory layout at compile time:

```rust
use std::mem::{size_of, align_of};

struct UserId(u64);

const _: () = assert!(size_of::<UserId>() == size_of::<u64>());
const _: () = assert!(align_of::<UserId>() == align_of::<u64>());
```

These assertions are evaluated by the compiler. If they fail, your code won't compile. They don't end up in the binary at all.

## Why C's typedef doesn't solve this

If you're coming from C or C++, you might think `typedef` does the same thing. It doesn't. `typedef` creates an alias, not a new type. The compiler treats the alias and the original as interchangeable:

```c
typedef unsigned int UserId;
typedef unsigned int ProductId;

void purchase(UserId user, ProductId product);

UserId u = 42;
ProductId p = 99;
purchase(p, u);  // Compiles without a warning. Arguments swapped.
```

This extends to more dangerous scenarios. Consider PCI-DSS compliance where you handle credit card numbers:

```c
typedef char* FullPan;    // Full card number - must never be logged
typedef char* MaskedPan;  // Masked card number - safe to log

FullPan full = "4111111111111111";
MaskedPan masked = full;  // Compiles. Full card number just leaked.
```

In Rust, these are genuinely different types:

```rust
struct FullPan(String);
struct MaskedPan(String);

let full = FullPan("4111111111111111".into());
// let masked: MaskedPan = full;
//             ^^^^^^^^^ expected `MaskedPan`, found `FullPan`
```

You'd need an explicit conversion, which is exactly where you'd put the masking logic:

```rust
impl From<FullPan> for MaskedPan {
    fn from(pan: FullPan) -> Self {
        let masked = format!(
            "****-****-****-{}",
            &pan.0[pan.0.len() - 4..]
        );
        MaskedPan(masked)
    }
}
```

C++ has some workarounds - [`BOOST_STRONG_TYPEDEF`](https://www.boost.org/doc/libs/release/libs/serialization/doc/strong_typedef.html), or the [`strong_type`](https://github.com/rollbear/strong_type) library - but they're all library-level hacks working against the language. In Rust, `struct Email(String);` is a first-class language feature. One line. C23 has a [proposal for strong typedefs](https://www.open-std.org/jtc1/sc22/wg14/www/docs/n3320.htm), but it's been years in the making with no consensus yet. Also worth noting: C++'s `using Meters = int;` has the exact same problem as `typedef` - it's an alias, not a distinct type.

## Beyond type safety: custom behavior

Newtypes aren't just about preventing argument swaps. They let you attach behavior to domain concepts. An email address isn't just a `String` - it has validation rules, a specific display format, and invariants that should be enforced at construction time.

```rust
#[derive(Debug, Clone, PartialEq)]
pub struct Email(String);

#[derive(Debug)]
pub enum EmailError {
    Empty,
    NoAtSign,
    TooLong,
}

impl Email {
    pub fn new(raw: &str) -> Result<Self, EmailError> {
        let trimmed = raw.trim().to_lowercase();
        if trimmed.is_empty() {
            return Err(EmailError::Empty);
        }
        if !trimmed.contains('@') {
            return Err(EmailError::NoAtSign);
        }
        if trimmed.len() > 254 {
            return Err(EmailError::TooLong);
        }
        Ok(Email(trimmed))
    }

    pub fn as_str(&self) -> &str {
        &self.0
    }

    pub fn domain(&self) -> &str {
        // Safe because we validated '@' exists in new()
        self.0.split('@').nth(1).unwrap()
    }
}

impl std::fmt::Display for Email {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        write!(f, "{}", self.0)
    }
}
```

The key insight: the inner `String` is private. Nobody outside this module can construct an `Email` without going through `new()`, which enforces all the invariants. Once you have an `Email`, you know it's valid. No need to re-validate at every call site.

This is the "parse, don't validate" philosophy. Instead of passing around a `String` and checking `is_valid_email()` everywhere, you parse it into an `Email` once. The type system carries the proof of validity from that point forward.

## Encapsulation through privacy

Because the inner field is private by default (no `pub`), the newtype controls all access:

```rust
mod types {
    pub struct Celsius(f64);

    impl Celsius {
        pub fn new(temp: f64) -> Self {
            Celsius(temp)
        }

        pub fn to_fahrenheit(&self) -> Fahrenheit {
            Fahrenheit(self.0 * 9.0 / 5.0 + 32.0)
        }

        pub fn value(&self) -> f64 {
            self.0
        }
    }

    pub struct Fahrenheit(f64);

    impl Fahrenheit {
        pub fn value(&self) -> f64 {
            self.0
        }
    }
}

use types::{Celsius, Fahrenheit};

let boiling = Celsius::new(100.0);
let f = boiling.to_fahrenheit();

// Can't accidentally add Celsius to Fahrenheit
// Can't construct invalid values
// Can't access the inner f64 without going through .value()
```

If you make the inner field `pub` (like `struct UserId(pub u64)`), you lose encapsulation but gain convenience. It's a tradeoff. For IDs where the value itself is opaque, keeping it private makes sense. For simple wrappers where you frequently need the inner value, `pub` saves you from writing getters.

## The boilerplate problem

The biggest complaint about newtypes is boilerplate. You define `struct UserId(u64)`, and suddenly you can't print it, compare it, hash it, or serialize it without manually implementing or deriving a bunch of traits.

The standard library derives help:

```rust
#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash, PartialOrd, Ord)]
struct UserId(u64);
```

But you still can't do `UserId(1) + UserId(2)`, format it with `Display`, or convert from the inner type without extra code. This is where [`derive_more`](https://crates.io/crates/derive_more) (v2.1.1 at the time of writing) comes in.

### derive_more

`derive_more` generates trait implementations that delegate to the inner type. It supports 34+ derives including `From`, `Into`, `Display`, `Deref`, `Add`, `Mul`, and more.

```toml
[dependencies]
derive_more = { version = "2", features = ["from", "into", "display", "deref"] }
```

```rust
use derive_more::{Display, From, Into, Deref};

#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash, From, Into, Display, Deref)]
pub struct UserId(u64);

#[derive(Debug, Clone, PartialEq, Eq, Hash, From, Display, Deref)]
pub struct Email(String);
```

Now you get:

```rust
let id = UserId::from(42_u64);     // From<u64>
let raw: u64 = id.into();          // Into<u64>
println!("User: {id}");            // Display delegates to u64's Display
let len = Email::from("a@b.com".to_string()).len(); // Deref to &str gives .len()
```

Each derive is a separate feature flag, so you only compile what you use. No binary bloat.

### nutype: validation built into the type

If you want validation baked directly into the newtype definition, [`nutype`](https://crates.io/crates/nutype) (v0.6.2) takes a different approach. Instead of deriving traits, it generates a constructor that sanitizes and validates:

```rust
use nutype::nutype;

#[nutype(
    sanitize(trim, lowercase),
    validate(not_empty, len_char_max = 254),
    derive(Debug, Clone, PartialEq, AsRef, Display),
)]
pub struct Username(String);
```

This generates `Username::try_new()` instead of a plain constructor:

```rust
let user = Username::try_new("  FooBar  ").unwrap();
assert_eq!(user.as_ref(), "foobar");  // trimmed + lowercased

let err = Username::try_new("   ");
assert!(err.is_err());  // empty after trimming
```

You can also define custom validation with your own error type:

```rust
#[derive(Debug)]
enum PortError {
    TooLow,
    Reserved,
}

#[nutype(
    validate(with = validate_port, error = PortError),
    derive(Debug, Clone, Copy, PartialEq),
)]
pub struct Port(u16);

fn validate_port(port: &u16) -> Result<(), PortError> {
    if *port == 0 {
        Err(PortError::TooLow)
    } else if *port < 1024 {
        Err(PortError::Reserved)
    } else {
        Ok(())
    }
}
```

The `nutype` approach is useful when you want the newtype to be self-documenting - the validation rules are right there in the type definition.

## repr(transparent): the FFI guarantee

Earlier I showed that `UserId(u64)` compiles to the same assembly as `u64`. But there's a subtle distinction between "the optimizer happens to erase it" and "the language guarantees identical layout". By default, Rust structs use `repr(Rust)`, which makes no promises about layout - the compiler is free to reorder fields, add padding, or use different calling conventions in future versions.

For pure Rust code, this distinction rarely matters. But if you're passing newtypes across FFI boundaries, or doing `transmute` between a newtype and its inner type, you need `#[repr(transparent)]`:

```rust
#[repr(transparent)]
pub struct Meters(f64);
```

[RFC 1758](https://rust-lang.github.io/rfcs/1758-repr-transparent.html) specifies that `repr(transparent)` guarantees the struct has the same layout AND calling convention as its single non-zero-sized field. This matters on architectures like ARM64, where a `struct { f64 }` might be passed in a general-purpose register while a bare `f64` goes in a floating-point register. Without `repr(transparent)`, that mismatch could cause a segfault across an FFI boundary.

The rules for `repr(transparent)`:
- Exactly one non-zero-sized field
- Any number of zero-sized fields (`PhantomData<T>`, etc.) are fine
- Can't be combined with other `repr` attributes

```rust
use std::marker::PhantomData;

#[repr(transparent)]
pub struct Handle<T> {
    raw: u64,
    _marker: PhantomData<T>,  // ZST, allowed
}

// Safe to pass to C as if it were a u64
extern "C" {
    fn close_handle(handle: Handle<()>);
}
```

For typical Rust newtypes that never cross FFI, you don't need `repr(transparent)`. But it's good practice to add it when the "zero-cost" guarantee matters for correctness, not just performance.

## Common newtype patterns in the wild

Here are patterns you'll see across the Rust ecosystem:

**Semantic IDs:**
```rust
use derive_more::{Display, From, Into};
use uuid::Uuid;

#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash, Display, From, Into)]
pub struct UserId(Uuid);

#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash, Display, From, Into)]
pub struct OrderId(Uuid);
```

**Constrained numerics:**
```rust
#[derive(Debug, Clone, Copy, PartialEq, PartialOrd)]
pub struct Percentage(f64);

impl Percentage {
    pub fn new(value: f64) -> Option<Self> {
        if (0.0..=100.0).contains(&value) {
            Some(Percentage(value))
        } else {
            None
        }
    }

    pub fn as_fraction(&self) -> f64 {
        self.0 / 100.0
    }
}
```

**Orphan rule workaround:**

If you recall from the [adapter pattern post](/blog/the-adapter-pattern-in-rust---wrapping-external-apis/), the orphan rule prevents you from implementing a foreign trait on a foreign type. Newtypes are the standard workaround - because your newtype is a local type, you can implement any trait on it:

```rust
// Can't impl Display for Vec<T> directly - both are foreign
// But you can wrap it:
struct CommaSeparated(Vec<String>);

impl std::fmt::Display for CommaSeparated {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        write!(f, "{}", self.0.join(", "))
    }
}
```

**Units of measure:**
```rust
#[derive(Debug, Clone, Copy, PartialEq)]
pub struct Meters(pub f64);

#[derive(Debug, Clone, Copy, PartialEq)]
pub struct Seconds(pub f64);

#[derive(Debug, Clone, Copy, PartialEq)]
pub struct MetersPerSecond(pub f64);

impl std::ops::Div<Seconds> for Meters {
    type Output = MetersPerSecond;
    fn div(self, rhs: Seconds) -> MetersPerSecond {
        MetersPerSecond(self.0 / rhs.0)
    }
}

let distance = Meters(100.0);
let time = Seconds(9.58);
let speed: MetersPerSecond = distance / time;  // Type-checked dimensional analysis
```

This is the [Mars Climate Orbiter](https://en.wikipedia.org/wiki/Mars_Climate_Orbiter) problem solved at compile time. Lockheed Martin used pound-seconds, NASA used newton-seconds, and a $327 million spacecraft was lost. With newtypes, that unit mismatch is a compiler error.

## When newtypes are overkill

Not every primitive needs a wrapper. Here are cases where a plain `u64` or `String` is fine:

**Local variables with obvious meaning.** If a variable lives for 5 lines in a single function and the name makes the intent clear, wrapping it adds noise:

```rust
// This is fine as-is
fn average(values: &[f64]) -> f64 {
    let sum: f64 = values.iter().sum();
    sum / values.len() as f64
}
```

**Internal implementation details.** If a type only exists inside one module and never appears in a public API, the cost-benefit tilts toward simplicity.

**Prototyping.** When you're exploring an idea and the API will change five times before it stabilizes, newtypes add friction to refactoring. Add them when the design settles.

**The two-of-the-same-type rule.** A good heuristic: if a function takes two parameters of the same type (two `u64`s, two `String`s), those are strong candidates for newtypes. If each parameter has a unique type, the compiler already prevents swaps.

## Implementing From/Into properly

When you write conversions for newtypes, always implement `From`, not `Into`. Rust has a blanket impl that gives you `Into` for free when `From` exists:

```rust
struct UserId(u64);

// Implement From for wrapping
impl From<u64> for UserId {
    fn from(val: u64) -> Self {
        UserId(val)
    }
}

// Implement From for unwrapping
impl From<UserId> for u64 {
    fn from(id: UserId) -> Self {
        id.0
    }
}

// Now you get both:
let id = UserId::from(42);        // From
let id2: UserId = 42_u64.into();  // Into (free via blanket impl)
let raw: u64 = id.into();         // Unwrap back
```

For fallible conversions, use `TryFrom`:

```rust
struct PositiveId(u64);

impl TryFrom<i64> for PositiveId {
    type Error = &'static str;

    fn try_from(val: i64) -> Result<Self, Self::Error> {
        if val <= 0 {
            Err("ID must be positive")
        } else {
            Ok(PositiveId(val as u64))
        }
    }
}
```

Or just use `derive_more` and skip the boilerplate entirely:

```rust
use derive_more::{From, Into};

#[derive(From, Into)]
struct UserId(u64);
```

## Deref: convenient but dangerous

You might be tempted to implement `Deref` on your newtype to get access to the inner type's methods:

```rust
use std::ops::Deref;

struct Email(String);

impl Deref for Email {
    type Target = str;
    fn deref(&self) -> &str {
        &self.0
    }
}

let email = Email("user@example.com".into());
println!("Length: {}", email.len());     // str::len() works
println!("Upper: {}", email.to_uppercase()); // str::to_uppercase() works
```

This is convenient but breaks encapsulation. Every `str` method is now available on `Email`, including ones that don't make sense (like `Email::split_whitespace()`). If your newtype is purely a wrapper with no invariants, `Deref` is fine. If it enforces rules (like "must contain @"), think carefully about which inner methods should be exposed. A targeted `as_str(&self) -> &str` method gives you control over the API surface.

## Wrapping up

The newtype pattern is one of those things that looks trivially simple but has deep implications for how you structure Rust code. One line - `struct Email(String);` - gives you:

- **Compile-time type safety** - no accidental parameter swaps or data mixing
- **Encapsulation** - private inner field means validation invariants are enforced
- **Custom behavior** - `Display`, `From`, `Ord`, and any trait you want
- **Orphan rule workaround** - implement foreign traits on wrapped foreign types
- **Zero runtime cost** - the wrapper is erased by the compiler

The cost is some boilerplate, which `derive_more` and `nutype` reduce to almost nothing. And unlike C's `typedef`, which gives you a false sense of type safety while treating aliases as interchangeable, Rust's newtypes are genuinely distinct types that the compiler enforces.

Start with the two-of-the-same-type rule. If a function takes two `String` parameters or two `u64` parameters, wrap at least one. You'll catch bugs that would have made it to production, and the binary won't be a single byte larger.
