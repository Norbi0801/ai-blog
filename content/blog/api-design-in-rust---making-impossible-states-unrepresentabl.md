+++
title = "API design in Rust - making impossible states unrepresentable"
date = 2025-05-30
description = "How to use newtypes, NonZero, validated wrappers, enums, and typestate to push invalid states out of your API surface entirely - so bugs become compile errors."

[taxonomies]
tags = ["rust", "api-design", "type-system", "design-patterns"]
+++

You write a function that takes a `u32` for a port number. Someone passes `0`. Your code panics somewhere deep in the networking stack, three layers away from the call site, with an error message that blames the wrong function. You add a runtime check: `assert!(port > 0)`. Now it panics earlier, with a better message. Progress? Sort of. The bug still compiles. The caller still has to remember the constraint. The next person to use your API won't read the doc comment. They'll discover the invariant the hard way - at runtime.

There's a better approach. Instead of documenting constraints and hoping callers respect them, you encode constraints into the type system. A `NonZeroU16` instead of a `u16`. An `Email(String)` instead of a raw `String`. A builder that won't let you call `.build()` until every required field is set. The compiler enforces the rules, and invalid states stop being "bugs you test for" and become "code that doesn't compile."

This idea - making impossible states unrepresentable - isn't new. It comes from the ML/Haskell world (Yaron Minsky coined the phrase for OCaml). But Rust is uniquely suited for it because the type system is expressive enough to encode real constraints, move semantics prevent reuse of consumed values, and zero-cost abstractions mean the type-level encoding compiles away to nothing.

This post walks through five levels of the technique, from simple newtypes to full typestate, with real standard library examples along the way.

<!-- more -->

## The spectrum: runtime checks vs compile-time types

Before getting into techniques, look at the same constraint enforced two ways. The rule: a database connection pool size must be between 1 and 128.

**Runtime validation:**

```rust
struct PoolConfig {
    size: u32,
    timeout_ms: u64,
}

impl PoolConfig {
    fn new(size: u32, timeout_ms: u64) -> Result<Self, ConfigError> {
        if size == 0 {
            return Err(ConfigError::InvalidPoolSize(
                "pool size must be at least 1".into(),
            ));
        }
        if size > 128 {
            return Err(ConfigError::InvalidPoolSize(
                "pool size must be at most 128".into(),
            ));
        }
        Ok(PoolConfig { size, timeout_ms })
    }

    fn size(&self) -> u32 {
        self.size
    }
}
```

This works. But every function that receives a `PoolConfig` has to trust that it was constructed through `new()`. If someone constructs it directly (the fields are `pub` - or maybe they're in the same module), the invariant is gone. And downstream code that calls `config.size()` gets a `u32` - is that guaranteed non-zero? Only if you go read the constructor.

**Compile-time encoding:**

```rust
use std::num::NonZeroU8;

struct PoolConfig {
    size: PoolSize,
    timeout_ms: u64,
}

/// Pool size between 1 and 128 (inclusive).
struct PoolSize(NonZeroU8);

impl PoolSize {
    /// Returns `None` if `n` is 0 or greater than 128.
    pub fn new(n: u8) -> Option<Self> {
        if n > 128 {
            return None;
        }
        NonZeroU8::new(n).map(PoolSize)
    }

    pub fn get(&self) -> u8 {
        self.0.get()
    }
}
```

Now `PoolSize` is its own type. You can't construct one without going through `new()`, which enforces the range. Any function that receives a `PoolSize` knows the value is 1-128 without checking. The `NonZeroU8` inside handles the "not zero" part at the type level. The range check (at most 128) is still a runtime check in the constructor - you can't express arbitrary ranges in Rust's type system (no dependent types). But the validation happens once, at construction, and the type carries the proof forward.

The key difference: with runtime validation, every consumer re-checks or blindly trusts. With a newtype, you validate once at the boundary and the type carries the guarantee everywhere it travels.

## Level 1: newtypes - the workhorse

A newtype is a single-field tuple struct that wraps an existing type to give it a new identity. It's the most common and most practical technique on this list.

```rust
pub struct Email(String);

impl Email {
    pub fn new(raw: &str) -> Result<Self, EmailError> {
        // Minimal check: contains exactly one @, has something before and after
        let at_pos = raw.find('@').ok_or(EmailError::MissingAt)?;
        if at_pos == 0 {
            return Err(EmailError::EmptyLocal);
        }
        if at_pos == raw.len() - 1 {
            return Err(EmailError::EmptyDomain);
        }
        if raw[at_pos + 1..].find('@').is_some() {
            return Err(EmailError::MultipleAt);
        }
        Ok(Email(raw.to_lowercase()))
    }

    pub fn as_str(&self) -> &str {
        &self.0
    }

    pub fn domain(&self) -> &str {
        // Safe: we validated @ exists in new()
        self.0.split('@').nth(1).unwrap()
    }
}

#[derive(Debug)]
pub enum EmailError {
    MissingAt,
    EmptyLocal,
    EmptyDomain,
    MultipleAt,
}
```

The inner `String` is private. Nobody outside this module can write `Email("garbage".into())`. The only path to an `Email` is through `new()`, which validates the data. Once you have an `Email`, the invariants are guaranteed - `domain()` can call `unwrap()` safely because the constructor already proved the `@` exists.

Compare this to passing `String` everywhere:

```rust
// Before: what does this function expect?
fn send_notification(to: String, subject: String, body: String) { /* ... */ }

// After: the types tell you
fn send_notification(to: &Email, subject: &Subject, body: &MessageBody) { /* ... */ }
```

The second signature is self-documenting. You can't accidentally swap `subject` and `body` if they're different types.

### The cost: zero

A newtype with one field has identical layout to the inner type. Rust guarantees this for `#[repr(transparent)]` structs, but even without the attribute, the compiler generates the same code:

```rust
pub struct Port(u16);

impl Port {
    pub fn new(n: u16) -> Option<Self> {
        if n == 0 { None } else { Some(Port(n)) }
    }

    pub fn get(&self) -> u16 {
        self.0
    }
}

#[no_mangle]
pub fn use_port(p: Port) -> u16 {
    p.get()
}

#[no_mangle]
pub fn use_raw(p: u16) -> u16 {
    p
}
```

Both compile to the same assembly on [Godbolt](https://rust.godbolt.org/) with `-O`:

```asm
use_port:
    mov     eax, edi
    ret

use_raw:
    mov     eax, edi
    ret
```

No wrapper overhead. The newtype is erased completely.

### When to reach for a newtype

Good candidates for newtypes:

- **IDs:** `UserId(String)`, `OrderId(Uuid)`, `TransactionId(u64)`. If you've read my [phantom types post](/blog/phantom-types-in-rust-compile-time-constraints-with-zero-runtime-cost/), you saw `Id<T>` with a phantom parameter for this. Newtypes are the simpler alternative when you have a small, fixed set of ID types without shared behavior.
- **Validated strings:** `Email`, `Url`, `Hostname`, `JsonPath`. Validate once at construction, trust everywhere after.
- **Bounded numbers:** `Port(u16)`, `Percentage(f64)`, `PoolSize(NonZeroU8)`. Enforce ranges in the constructor.
- **Domain concepts:** `Money { cents: i64, currency: Currency }`, `Temperature(f64)`. Prevent arithmetic between incompatible values.

## Level 2: NonZero and niche optimization

The standard library ships a family of types that encode "not zero" directly into the type: `NonZeroU8`, `NonZeroU16`, `NonZeroU32`, `NonZeroU64`, `NonZeroU128`, `NonZeroUsize`, and their signed counterparts. Since Rust 1.79, there's also the unified [`NonZero<T>`](https://doc.rust-lang.org/stable/std/num/struct.NonZero.html) wrapper that works with any integer type.

The API is simple:

```rust
use std::num::NonZeroU32;

// Construction - fallible
let maybe = NonZeroU32::new(42);  // Some(NonZeroU32(42))
let nope = NonZeroU32::new(0);    // None

// Unsafe construction when you've already checked
let definitely = unsafe { NonZeroU32::new_unchecked(42) };

// Extraction
let value: u32 = maybe.unwrap().get();  // 42
```

But the interesting part isn't the API - it's what happens to memory layout.

### Niche optimization

`NonZeroU32` tells the compiler that the bit pattern `0x00000000` is never valid. The compiler exploits this: it uses that forbidden bit pattern to represent `None` in `Option<NonZeroU32>`. The result:

```rust
use std::mem::size_of;
use std::num::NonZeroU32;

assert_eq!(size_of::<u32>(), 4);
assert_eq!(size_of::<NonZeroU32>(), 4);
assert_eq!(size_of::<Option<u32>>(), 8);       // 4 bytes value + 4 bytes discriminant
assert_eq!(size_of::<Option<NonZeroU32>>(), 4); // zero overhead!
```

`Option<u32>` is 8 bytes - the compiler needs a discriminant to know whether the option is `Some` or `None`. But `Option<NonZeroU32>` is 4 bytes - the compiler stores `None` as `0` and `Some(n)` as the raw value of `n`. No discriminant needed.

This optimization - called niche filling - extends further. `Result<NonZeroU32, ()>` is also 4 bytes. The compiler is remarkably good at finding unused bit patterns and exploiting them.

Where does this show up in practice? Everywhere you have IDs that can't be zero:

```rust
use std::num::NonZeroU64;

/// A database row ID. Always positive.
pub struct RowId(NonZeroU64);

impl RowId {
    pub fn new(id: u64) -> Option<Self> {
        NonZeroU64::new(id).map(RowId)
    }

    pub fn get(&self) -> u64 {
        self.0.get()
    }
}

// A function that might not find a row
fn find_row(table: &str, id: RowId) -> Option<Row> {
    // Option<RowId> in a struct costs 8 bytes, not 16
    // This matters when you have millions of rows with optional foreign keys
    todo!()
}
```

In data-heavy applications - caches, indexes, large `Vec`s of records with optional foreign keys - the 50% size reduction from `Option<NonZeroU64>` vs `Option<u64>` adds up. It's both a correctness tool (can't store a zero ID) and a performance tool (better cache utilization).

### NonZero arithmetic

`NonZeroU32` also provides checked arithmetic that preserves the non-zero guarantee:

```rust
use std::num::NonZeroU32;

let a = NonZeroU32::new(10).unwrap();
let b = NonZeroU32::new(3).unwrap();

// checked_mul returns Option<NonZeroU32> - None on overflow
let product = a.checked_mul(b); // Some(30)

// saturating_mul clamps at MAX, which is still non-zero
let big = a.saturating_mul(NonZeroU32::MAX);

// Division by NonZeroU32 can never panic
let x: u32 = 100;
let safe_div = x / b; // no division-by-zero possible
```

That last line is the real payoff. Division by `NonZeroU32` is guaranteed safe. The compiler doesn't insert a zero-check branch because the type already proved it's unnecessary. You get both correctness and a (tiny) performance win from eliminating the branch.

## Level 3: the Validated wrapper pattern

Newtypes work great for specific domain types. But when you find yourself writing the same "validate at construction, expose via getter" boilerplate for the tenth time, you start wanting a generic solution.

Here's a pattern that extracts validation into a trait:

```rust
use std::fmt;
use std::marker::PhantomData;

/// A value of type `T` that has been validated according to rule `V`.
pub struct Validated<T, V: ValidationRule<T>> {
    value: T,
    _rule: PhantomData<V>,
}

pub trait ValidationRule<T> {
    type Error: fmt::Debug;

    fn validate(value: &T) -> Result<(), Self::Error>;
}

impl<T, V: ValidationRule<T>> Validated<T, V> {
    pub fn new(value: T) -> Result<Self, V::Error> {
        V::validate(&value)?;
        Ok(Validated {
            value,
            _rule: PhantomData,
        })
    }

    pub fn into_inner(self) -> T {
        self.value
    }

    pub fn get(&self) -> &T {
        &self.value
    }
}
```

Now define rules as zero-sized types:

```rust
/// String must be non-empty and at most 255 bytes.
pub struct NonEmptyShortString;

impl ValidationRule<String> for NonEmptyShortString {
    type Error = &'static str;

    fn validate(value: &String) -> Result<(), Self::Error> {
        if value.is_empty() {
            return Err("string must not be empty");
        }
        if value.len() > 255 {
            return Err("string must be at most 255 bytes");
        }
        Ok(())
    }
}

/// Integer must be in [1, 100].
pub struct Percentage;

impl ValidationRule<u32> for Percentage {
    type Error = &'static str;

    fn validate(value: &u32) -> Result<(), Self::Error> {
        if *value == 0 || *value > 100 {
            return Err("percentage must be between 1 and 100");
        }
        Ok(())
    }
}

// Usage
type Name = Validated<String, NonEmptyShortString>;
type Score = Validated<u32, Percentage>;

fn create_user(name: Name, score: Score) {
    println!("User: {} (score: {})", name.get(), score.get());
}
```

If you've read my [phantom types post](/blog/phantom-types-in-rust-compile-time-constraints-with-zero-runtime-cost/), you recognize the `PhantomData<V>` pattern - the validation rule exists only in the type, not in memory. `Validated<String, NonEmptyShortString>` and `Validated<String, Percentage>` are different types even though both wrap a `String`. You can't pass a validated name where a validated score is expected.

The `PhantomData<V>` is zero-sized, so `Validated<T, V>` has the same layout as `T`. The validation rule is a type-level tag - it guides the compiler but vanishes from the binary.

### Tradeoffs of Validated<T, V>

This pattern is useful when:

- You have many validated types with similar structure (all wrap a String, all wrap a number)
- You want a consistent API (`new()`, `get()`, `into_inner()`) across all of them
- You want to compose validation rules (a `Validated<String, And<NonEmpty, MaxLen<255>>>` type)

It's overkill when:

- You have a handful of domain types - just write dedicated newtypes with their own methods
- The validation logic needs access to external state (database lookups, config) - traits with `&self` on validation rules get awkward fast
- You need serde support - implementing `Deserialize` for `Validated<T, V>` requires careful handling to run validation during deserialization

If you've worked with boundary validation before (I wrote about [JSON Schema for this purpose](/blog/json-schema-validating-data-at-the-boundary/)), the `Validated` wrapper is the Rust-native complement: JSON Schema validates data at the system boundary (HTTP, config files), and `Validated<T, V>` carries the proof through your Rust code.

## Level 4: enums to eliminate boolean blindness

One of the most common API design mistakes is using `bool` parameters:

```rust
fn process_order(order: &Order, expedited: bool, gift_wrap: bool) {
    // ...
}

// At the call site - what do these booleans mean?
process_order(&order, true, false);
```

This is called "boolean blindness." The call site is unreadable without checking the function signature. Worse, swapping the two booleans compiles without error and silently changes behavior.

The fix is enums:

```rust
pub enum Shipping {
    Standard,
    Expedited,
}

pub enum Wrapping {
    None,
    GiftWrap,
}

fn process_order(order: &Order, shipping: Shipping, wrapping: Wrapping) {
    // ...
}

// Now the call site reads clearly
process_order(&order, Shipping::Expedited, Wrapping::None);
```

You can't swap the arguments - `Shipping` and `Wrapping` are different types. And if you later add `Shipping::Overnight` or `Wrapping::PremiumBox`, the `match` arms in `process_order` will force you to handle the new variants.

### Modeling mutually exclusive states

Enums shine when you have fields that are mutually exclusive or conditionally present. The classic anti-pattern:

```rust
// Bad: which combinations are valid?
struct Payment {
    credit_card_number: Option<String>,
    credit_card_expiry: Option<String>,
    credit_card_cvv: Option<String>,
    bank_account_number: Option<String>,
    bank_routing_number: Option<String>,
    paypal_email: Option<String>,
}
```

This struct has 64 possible states (2^6 Options). How many are valid? Three: all credit card fields set, both bank fields set, or the PayPal email set. The other 61 combinations are bugs waiting to happen. Every function that receives a `Payment` must check which fields are actually present.

```rust
// Good: impossible states are literally unrepresentable
enum PaymentMethod {
    CreditCard {
        number: CardNumber,
        expiry: Expiry,
        cvv: Cvv,
    },
    BankTransfer {
        account: AccountNumber,
        routing: RoutingNumber,
    },
    PayPal {
        email: Email,
    },
}

struct Payment {
    method: PaymentMethod,
    amount: Money,
}
```

Now there are exactly three valid states. You can't have a credit card number without a CVV. You can't have a bank routing number alongside a PayPal email. The invalid states don't exist - they're not checked at runtime, they're not representable in the type system.

The `match` expression forces exhaustive handling:

```rust
fn process_payment(payment: &Payment) -> Result<Receipt, PaymentError> {
    match &payment.method {
        PaymentMethod::CreditCard { number, expiry, cvv } => {
            charge_card(number, expiry, cvv, &payment.amount)
        }
        PaymentMethod::BankTransfer { account, routing } => {
            initiate_transfer(account, routing, &payment.amount)
        }
        PaymentMethod::PayPal { email } => {
            paypal_charge(email, &payment.amount)
        }
    }
}
```

No `if payment.credit_card_number.is_some()` chains. No "what if both credit card and PayPal fields are set?" edge case. The enum eliminated the entire category of bugs.

### State machines with enums

When an object transitions through a linear sequence of states, enums model this cleanly:

```rust
pub enum ConnectionState {
    Disconnected,
    Connecting { address: String, attempt: u32 },
    Connected { stream: TcpStream, connected_at: Instant },
    Disconnecting { reason: String },
}

pub struct Connection {
    state: ConnectionState,
}

impl Connection {
    pub fn connect(&mut self, addr: &str) -> Result<(), ConnectionError> {
        match &self.state {
            ConnectionState::Disconnected => {
                self.state = ConnectionState::Connecting {
                    address: addr.to_string(),
                    attempt: 1,
                };
                Ok(())
            }
            ConnectionState::Connected { .. } => {
                Err(ConnectionError::AlreadyConnected)
            }
            ConnectionState::Connecting { .. } => {
                Err(ConnectionError::AlreadyConnecting)
            }
            ConnectionState::Disconnecting { .. } => {
                Err(ConnectionError::Disconnecting)
            }
        }
    }
}
```

Each state variant carries only the data relevant to that state. `Connecting` has an attempt counter. `Connected` has the stream and timestamp. `Disconnecting` has a reason. There's no `stream: Option<TcpStream>` that might or might not be `Some` depending on the current state.

This is the runtime version of the typestate pattern. If you've read my [phantom types post](/blog/phantom-types-in-rust-compile-time-constraints-with-zero-runtime-cost/), you saw the compile-time version where each state is a separate type and transitions consume `self`. The enum approach is more flexible (states can change at runtime, values can be stored in collections) but less strict (invalid transitions are runtime errors, not compile errors). Pick enums when the state is determined by runtime logic. Pick typestate when the state is determined by the shape of the code.

## Level 5: typestate builders

I covered the typestate pattern in depth in the [phantom types post](/blog/phantom-types-in-rust-compile-time-constraints-with-zero-runtime-cost/), so I won't repeat the fundamentals. But builders deserve a closer look because they're the most practical application of typestate that most codebases can adopt immediately.

The standard builder pattern in Rust looks like this:

```rust
struct HttpClient {
    base_url: String,
    timeout: Duration,
    auth_token: Option<String>,
}

struct HttpClientBuilder {
    base_url: Option<String>,
    timeout: Option<Duration>,
    auth_token: Option<String>,
}

impl HttpClientBuilder {
    fn new() -> Self {
        HttpClientBuilder {
            base_url: None,
            timeout: None,
            auth_token: None,
        }
    }

    fn base_url(mut self, url: &str) -> Self {
        self.base_url = Some(url.to_string());
        self
    }

    fn timeout(mut self, d: Duration) -> Self {
        self.timeout = Some(d);
        self
    }

    fn auth_token(mut self, token: &str) -> Self {
        self.auth_token = Some(token.to_string());
        self
    }

    fn build(self) -> Result<HttpClient, &'static str> {
        Ok(HttpClient {
            base_url: self.base_url.ok_or("base_url is required")?,
            timeout: self.timeout.unwrap_or(Duration::from_secs(30)),
            auth_token: self.auth_token,
        })
    }
}
```

The problem: `build()` returns a `Result`. You find out you forgot `base_url` at runtime. In tests, sure, that's fine. In production, that's a startup crash.

With typestate, `build()` is only available when required fields are set:

```rust
use std::marker::PhantomData;
use std::time::Duration;

struct Missing;
struct Present;

struct HttpClientBuilder<BaseUrl> {
    base_url: Option<String>,
    timeout: Duration,
    auth_token: Option<String>,
    _base_url: PhantomData<BaseUrl>,
}

impl HttpClientBuilder<Missing> {
    fn new() -> Self {
        HttpClientBuilder {
            base_url: None,
            timeout: Duration::from_secs(30),
            auth_token: None,
            _base_url: PhantomData,
        }
    }

    fn base_url(self, url: &str) -> HttpClientBuilder<Present> {
        HttpClientBuilder {
            base_url: Some(url.to_string()),
            timeout: self.timeout,
            auth_token: self.auth_token,
            _base_url: PhantomData,
        }
    }
}

// Optional methods available in any state
impl<B> HttpClientBuilder<B> {
    fn timeout(mut self, d: Duration) -> Self {
        self.timeout = d;
        self
    }

    fn auth_token(mut self, token: &str) -> Self {
        self.auth_token = Some(token.to_string());
        self
    }
}

// build() only available when base_url is Present
impl HttpClientBuilder<Present> {
    fn build(self) -> HttpClient {
        HttpClient {
            base_url: self.base_url.unwrap(), // safe: state guarantees it
            timeout: self.timeout,
            auth_token: self.auth_token,
        }
    }
}
```

Now this compiles:

```rust
let client = HttpClientBuilder::new()
    .base_url("https://api.example.com")
    .timeout(Duration::from_secs(10))
    .build();
```

And this doesn't:

```rust
let client = HttpClientBuilder::new()
    .timeout(Duration::from_secs(10))
    .build(); // ERROR: no method named `build` found for
              //   `HttpClientBuilder<Missing>` in the current scope
```

The error message tells you exactly what's wrong: `build` exists on `HttpClientBuilder<Present>`, not `HttpClientBuilder<Missing>`. Set the base URL first.

### Scaling typestate builders

With one required field, one type parameter works fine. With two or three, you get `HttpClientBuilder<Present, Missing, Present>`. That's still manageable. Beyond that, it gets unwieldy - as I mentioned in the phantom types post, five type parameters with 32 possible states is where you should reconsider.

The [`typed-builder`](https://crates.io/crates/typed-builder) crate automates this with a derive macro:

```rust
use typed_builder::TypedBuilder;

#[derive(TypedBuilder)]
struct HttpClient {
    base_url: String,
    #[builder(default = Duration::from_secs(30))]
    timeout: Duration,
    #[builder(default, setter(strip_option))]
    auth_token: Option<String>,
}

// Compiles - all required fields set
let client = HttpClient::builder()
    .base_url("https://api.example.com".into())
    .build();

// Won't compile - missing base_url
// let client = HttpClient::builder().build();
```

The macro generates the typestate machinery for you. Same compile-time guarantees, zero boilerplate.

## Real-world examples from the standard library

These patterns aren't academic. Rust's standard library uses them extensively.

### File descriptors: OwnedFd and BorrowedFd

Before Rust 1.63, file descriptor handling was unsafe by convention. You'd use `RawFd` (a type alias for `i32`) and hope that:

1. The descriptor was valid
2. Nobody else closed it while you were using it
3. Exactly one owner would close it when done

All three were runtime assumptions. [RFC 3128](https://rust-lang.github.io/rfcs/3128-io-safety.html) introduced `OwnedFd` and `BorrowedFd` to encode these invariants in the type system:

```rust
// OwnedFd: this code owns the file descriptor and will close it on drop.
// Implements Drop to call close(). Moving it transfers ownership.
pub struct OwnedFd { fd: RawFd }

// BorrowedFd<'a>: borrowed access to a file descriptor.
// The lifetime 'a guarantees the fd won't be closed while this exists.
pub struct BorrowedFd<'a> { fd: RawFd, _phantom: PhantomData<&'a OwnedFd> }
```

The `PhantomData<&'a OwnedFd>` connects the borrowed descriptor's lifetime to the owner. If the owner gets dropped, any outstanding `BorrowedFd` references become invalid - and the borrow checker catches it at compile time.

Both types are `#[repr(transparent)]` with a `RawFd` inside, so they have identical layout to a raw `i32`. And both use niche optimization: `Option<OwnedFd>` is the same size as `RawFd` because `-1` (the invalid fd sentinel) serves as the `None` discriminant.

The old API:

```rust
// Anyone can fabricate a fd, close it twice, or use it after close
fn dangerous(fd: RawFd) {
    unsafe { libc::write(fd, b"hi".as_ptr() as _, 2); }
}
```

The new API:

```rust
// The borrow checker ensures the fd is valid for the duration of this call
fn safe(fd: BorrowedFd<'_>) {
    // fd is guaranteed valid - the borrow checker enforces it
}
```

Same operation. Same performance. But one encodes the safety invariants in the type system and the other relies on programmer discipline.

### std::fs::File itself

`File` is essentially a newtype over `OwnedFd`:

```rust
pub struct File {
    inner: fs::FileInner, // platform-specific, wraps OwnedFd
}
```

You can't access the raw file descriptor without calling `as_raw_fd()` or `into_raw_fd()`. The `File` type enforces that:

- Opening a file returns a `Result<File, io::Error>` - you can't have a `File` that failed to open
- Dropping a `File` closes the descriptor - no forgotten `close()` calls
- `File` implements `Read` and `Write` but not `Clone` - there's one owner, always

Compare this to C where `FILE*` might be null, might be closed, might be pointing to freed memory. In Rust, if you have a `File`, it's valid. Period. The type makes the invalid states (null handle, closed handle, double-free) unrepresentable.

### String vs &str vs &[u8]

If you've read my [string types post](/blog/why-rust-has-so-many-string-types/), you already know that `String` guarantees valid UTF-8 while `Vec<u8>` doesn't. This is another instance of making impossible states unrepresentable: a `String` cannot contain invalid UTF-8 because the only way to construct one goes through validation. The `from_utf8()` function returns `Result<String, FromUtf8Error>` - if it succeeds, the bytes are valid UTF-8 forever. No re-validation needed downstream.

`String::from_utf8_unchecked()` exists as an escape hatch, but it's `unsafe` - marking the point where the programmer takes responsibility for an invariant the type system can no longer verify.

## The comparison table

Here's when to use each technique:

| Technique | Invalid state detected | Runtime cost | Ergonomic cost | Best for |
|---|---|---|---|---|
| Runtime `assert!` | Runtime (panic) | Branch per check | Low | Quick prototypes |
| `Result` validation | Runtime (error) | Branch per check | Medium | Boundary input |
| Newtype with private field | Construction time | Zero after construction | Low | Domain types |
| `NonZero` / niche types | Construction time | Zero (niche optimized) | Low | Numeric constraints |
| `Validated<T, V>` wrapper | Construction time | Zero after construction | Medium | Many similar constraints |
| Enum variants | N/A - state is structural | Discriminant byte | Low | Mutually exclusive states |
| Typestate (phantom types) | Compile time | Zero | High | Protocol/lifecycle enforcement |

The techniques aren't mutually exclusive. A well-designed API often uses several:

- **Newtypes** for domain primitives (`Email`, `UserId`, `Money`)
- **NonZero** for numeric fields that can't be zero
- **Enums** for mutually exclusive alternatives
- **Typestate** for builder patterns or protocol state machines
- **Runtime validation** for constraints that can't be expressed in types (regex patterns, database uniqueness)

## When to stop

Type-level encoding has diminishing returns. Here's where to draw the line:

**Stop when the type signature becomes unreadable.** `ConnectionManager<Authenticated, Pooled, WithTls, Compressed, Traced>` with five type parameters is harder to work with than a runtime-validated config struct. If your type signature needs a comment to explain it, the types aren't making things clearer.

**Stop when the constraint is truly dynamic.** If the valid range of a value comes from a config file loaded at startup, you can't encode it in the type system. Use runtime validation and a newtype to carry the "validated" proof.

**Stop when serde integration matters more than strictness.** Deserializing into a typestate builder is painful. If your types cross serialization boundaries frequently, the ergonomic cost of compile-time enforcement might outweigh the benefits. A newtype with `Deserialize` that validates in a custom deserializer is often the right middle ground.

**Stop when your team hasn't seen the pattern.** A phantom type parameter that says `HttpClientBuilder<Missing>` is clear to someone who knows typestate. To someone who doesn't, it's a cryptic error message that sends them down a rabbit hole. Document the pattern, or use simpler newtypes until the team is ready.

The goal isn't to encode every possible constraint in the type system. The goal is to encode the constraints that cause the most bugs. Start with newtypes for your most-confused types (string IDs are the #1 candidate). Add `NonZero` where division-by-zero or zero-as-invalid is a real risk. Reach for typestate only when the state machine matters for correctness - protocol handlers, connection lifecycles, builders with genuinely required fields.

Every type-level constraint you add is a class of bugs that disappears. Not "caught by tests." Not "documented in comments." Disappeared - impossible to write, rejected by the compiler. That's the payoff.
