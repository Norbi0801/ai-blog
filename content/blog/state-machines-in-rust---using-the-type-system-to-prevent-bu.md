+++
title = "State machines in Rust - using the type system to prevent bugs"
date = 2026-02-17
description = "How the typestate pattern encodes state transitions into Rust's type system so the compiler rejects invalid sequences at compile time - with zero runtime cost."

[taxonomies]
tags = ["rust", "type-system", "design-patterns", "state-machines"]
+++

You have an order processing system. An order starts as a draft, gets submitted, then paid, then shipped. Simple enough. You model it with a struct and some methods:

```rust
struct Order {
    id: String,
    items: Vec<String>,
    status: String,
    paid_amount: Option<u64>,
    tracking_number: Option<String>,
}

impl Order {
    fn submit(&mut self) {
        self.status = "submitted".to_string();
    }

    fn pay(&mut self, amount: u64) {
        self.paid_amount = Some(amount);
        self.status = "paid".to_string();
    }

    fn ship(&mut self, tracking: &str) {
        self.tracking_number = Some(tracking.to_string());
        self.status = "shipped".to_string();
    }
}
```

Nothing stops someone from calling `ship()` on a draft. Nothing prevents `pay()` after the order is already shipped. The `status` field is a string - typo it as "submited" and the system silently does the wrong thing. Every consumer of `Order` has to manually check the status before doing anything, and if they forget, it's a runtime bug that might not surface until production.

This is a state machine encoded as wishful thinking.

Rust can do better. The type system can encode the valid state transitions so the compiler rejects invalid ones at compile time. No runtime checks. No string comparisons. No "forgot to verify status" bugs. The technique is called the **typestate pattern**, and it turns state machine violations into compiler errors.

<!-- more -->

## Quick foundations

If you haven't read the earlier posts that build up to this one, here's the dependency chain:

- [Phantom types](/blog/phantom-types-in-rust-compile-time-constraints-with-zero-runtime-cost/) covers `PhantomData<T>`, zero-sized types, and the basics of using type parameters as compile-time tags. That post includes a brief state machine example. This post goes much deeper.
- [Rust enums are not what you think](/blog/rust-enums-are-not-what-you-think-algebraic-data-types-explained/) covers sum types, memory layout, and niche optimization.
- [API design - making impossible states unrepresentable](/blog/api-design-in-rust-making-impossible-states-unrepresentable/) covers the spectrum from runtime validation to typestate builders, with a comparison table.

This post assumes you know what `PhantomData` is and why it exists. We're going straight into the pattern, the tradeoffs, and the places where it shines vs where it falls apart.

## The typestate pattern: Order\<Draft\> to Order\<Shipped\>

The core idea: represent each state as a separate type (usually a zero-sized struct), and make the struct generic over the state. Methods that transition between states consume `self` and return the struct parameterized with the new state.

```rust
use std::marker::PhantomData;

// States - zero-sized, exist only in the type system
struct Draft;
struct Submitted;
struct Paid;
struct Shipped;

struct Order<S> {
    id: String,
    items: Vec<String>,
    _state: PhantomData<S>,
}
```

Each state gets its own `impl` block. Transitions consume the old value and produce a new one with a different type parameter:

```rust
impl Order<Draft> {
    fn new(id: &str) -> Self {
        Order {
            id: id.to_string(),
            items: Vec::new(),
            _state: PhantomData,
        }
    }

    fn add_item(mut self, item: &str) -> Self {
        self.items.push(item.to_string());
        self
    }

    fn submit(self) -> Order<Submitted> {
        Order {
            id: self.id,
            items: self.items,
            _state: PhantomData,
        }
    }
}

impl Order<Submitted> {
    fn pay(self, _amount: u64) -> Order<Paid> {
        Order {
            id: self.id,
            items: self.items,
            _state: PhantomData,
        }
    }

    fn cancel(self) -> Order<Draft> {
        Order {
            id: self.id,
            items: self.items,
            _state: PhantomData,
        }
    }
}

impl Order<Paid> {
    fn ship(self, _tracking: &str) -> Order<Shipped> {
        Order {
            id: self.id,
            items: self.items,
            _state: PhantomData,
        }
    }
}

// Methods available in ALL states
impl<S> Order<S> {
    fn id(&self) -> &str {
        &self.id
    }

    fn items(&self) -> &[String] {
        &self.items
    }
}
```

Usage:

```rust
fn main() {
    let order = Order::new("ORD-001")
        .add_item("Keyboard")
        .add_item("Mouse")
        .submit()
        .pay(9999)
        .ship("TRACK-123");

    println!("Shipped order {}", order.id());
}
```

Now try to ship a draft:

```rust
fn main() {
    let order = Order::new("ORD-002");
    order.ship("TRACK-456"); // compile error
}
```

```
error[E0599]: no method named `ship` found for struct `Order<Draft>`
              in the current scope
  --> src/main.rs:XX:XX
   |
   = note: the method was found for `Order<Paid>`
```

The compiler tells you exactly what happened: `ship` exists on `Order<Paid>`, not `Order<Draft>`. You can't ship an unpaid order. Not "you shouldn't" - you *can't*. The method literally doesn't exist on that type.

And because `submit()` takes `self` by value (not `&mut self`), the old `Order<Draft>` is moved. You can't use it after submitting:

```rust
let draft = Order::new("ORD-003");
let submitted = draft.submit();
draft.add_item("Oops"); // compile error: value used after move
```

Rust's ownership system and the typestate pattern reinforce each other. Move semantics guarantee that once you transition, the old state is gone. No stale references to a state that no longer applies.

## How this actually works at the machine level

The state parameter `S` is a zero-sized type. `PhantomData<S>` contributes zero bytes to the struct layout. So `Order<Draft>`, `Order<Submitted>`, `Order<Paid>`, and `Order<Shipped>` all have the exact same size and memory layout:

```rust
use std::mem::size_of;

fn main() {
    assert_eq!(size_of::<Order<Draft>>(), size_of::<Order<Submitted>>());
    assert_eq!(size_of::<Order<Draft>>(), size_of::<Order<Paid>>());
    assert_eq!(size_of::<Order<Draft>>(), size_of::<Order<Shipped>>());

    // PhantomData adds nothing
    assert_eq!(size_of::<PhantomData<Draft>>(), 0);
    assert_eq!(size_of::<PhantomData<Shipped>>(), 0);
}
```

When `submit()` converts `Order<Draft>` to `Order<Submitted>`, there's no runtime transformation. The compiler monomorphizes each generic instantiation into its own concrete type, but since the state parameter doesn't affect data layout, LLVM recognizes that the transition is just moving the same bytes. With optimizations on, the `submit()` call compiles down to... nothing. The data stays where it is.

You can verify this on [Godbolt](https://rust.godbolt.org/). A simplified version:

```rust
use std::marker::PhantomData;

struct A;
struct B;

struct Machine<S> {
    value: u64,
    _s: PhantomData<S>,
}

#[no_mangle]
pub fn transition(m: Machine<A>) -> Machine<B> {
    Machine { value: m.value, _s: PhantomData }
}
```

Compile with `-C opt-level=2`. The output:

```asm
transition:
    mov     rax, rdi
    ret
```

One `mov` instruction, which just satisfies the calling convention (returning the value in `rax`). The state transition itself is a no-op. If the function gets inlined (which it will in any real code), even the `mov` disappears. This is what zero-cost abstraction means - the entire state machine enforcement evaporates after type checking, leaving the same machine code you'd write by hand.

## The enum alternative

The other way to model state machines in Rust is with enums. I showed a `ConnectionState` example in the [API design post](/blog/api-design-in-rust-making-impossible-states-unrepresentable/), and the [enums post](/blog/rust-enums-are-not-what-you-think-algebraic-data-types-explained/) covered why enums are sum types. Here's the same order system with an enum:

```rust
enum OrderState {
    Draft { items: Vec<String> },
    Submitted { items: Vec<String> },
    Paid { items: Vec<String>, amount: u64 },
    Shipped { items: Vec<String>, amount: u64, tracking: String },
}

struct Order {
    id: String,
    state: OrderState,
}

impl Order {
    fn new(id: &str) -> Self {
        Order {
            id: id.to_string(),
            state: OrderState::Draft { items: Vec::new() },
        }
    }

    fn submit(&mut self) -> Result<(), &'static str> {
        match &self.state {
            OrderState::Draft { items } => {
                let items = items.clone();
                self.state = OrderState::Submitted { items };
                Ok(())
            }
            _ => Err("can only submit a draft order"),
        }
    }

    fn pay(&mut self, amount: u64) -> Result<(), &'static str> {
        match &self.state {
            OrderState::Submitted { items } => {
                let items = items.clone();
                self.state = OrderState::Paid { items, amount };
                Ok(())
            }
            _ => Err("can only pay a submitted order"),
        }
    }

    fn ship(&mut self, tracking: &str) -> Result<(), &'static str> {
        match &self.state {
            OrderState::Paid { items, amount } => {
                let items = items.clone();
                let amount = *amount;
                self.state = OrderState::Shipped {
                    items,
                    amount,
                    tracking: tracking.to_string(),
                };
                Ok(())
            }
            _ => Err("can only ship a paid order"),
        }
    }
}
```

This works. And for many use cases, it's the right choice. But notice the differences:

**Error detection time.** The enum version returns `Result` - invalid transitions are caught at runtime. The typestate version catches them at compile time. If your test suite doesn't exercise the wrong transition path, the enum version ships with a latent bug. The typestate version can't compile with the bug present.

**Data per state.** The enum version naturally holds different data per state - `Paid` has an `amount`, `Draft` doesn't. With typestate, you'd either need separate structs per state (more boilerplate) or use `Option` fields and unwrap them in state-specific methods (losing some type safety). More on this below.

**Runtime flexibility.** The enum version lets you store orders in a `Vec<Order>` regardless of state. You can deserialize JSON into an `Order` and figure out the state from the data. With typestate, `Order<Draft>` and `Order<Paid>` are different types - you can't put them in the same `Vec` without boxing or wrapping in an enum.

**Code at the call site.** Enum transitions return `Result`, so every caller has to handle the error case. Typestate transitions are infallible - if the code compiles, the transition is valid.

## Side-by-side comparison

| Aspect | Typestate | Enum |
|---|---|---|
| Invalid transition detected | Compile time | Runtime |
| State-specific data | Needs workaround | Natural (per-variant fields) |
| Heterogeneous collections | No (different types) | Yes (same type) |
| Serialization | Requires enum wrapper | Direct serde support |
| Method signatures | Clean, infallible | Returns Result |
| Boilerplate | Struct reconstruction per transition | Match arms per method |
| New state added | Add struct + impl block | Add variant + match arms |
| Runtime cost of transition | Zero | Discriminant write + possible branch |
| Testing | Less needed (compiler catches misuse) | Must test invalid transitions |
| Error messages | "no method found for Order\<Draft\>" | Runtime string/error type |

Neither approach dominates. They solve different problems.

## When typestate wins

**Protocol implementations.** TLS handshakes, TCP state machines, SMTP commands - these have a fixed sequence that never varies at runtime. You always go `ClientHello -> ServerHello -> KeyExchange -> ...`. The typestate pattern makes it impossible to send data before the handshake is complete.

The [phantom types post](/blog/phantom-types-in-rust-compile-time-constraints-with-zero-runtime-cost/) showed a `TcpSocket<State>` example. Here's a more realistic sketch of what this looks like for an SMTP client:

```rust
use std::marker::PhantomData;

struct Disconnected;
struct Connected;
struct Authenticated;
struct Ready;

struct SmtpClient<S> {
    host: String,
    _state: PhantomData<S>,
}

impl SmtpClient<Disconnected> {
    fn connect(host: &str) -> SmtpClient<Connected> {
        // ... TCP connect, read banner
        SmtpClient { host: host.to_string(), _state: PhantomData }
    }
}

impl SmtpClient<Connected> {
    fn authenticate(self, _user: &str, _pass: &str) -> SmtpClient<Authenticated> {
        // ... EHLO, AUTH LOGIN
        SmtpClient { host: self.host, _state: PhantomData }
    }
}

impl SmtpClient<Authenticated> {
    fn mail_from(self, _from: &str) -> SmtpClient<Ready> {
        // ... MAIL FROM
        SmtpClient { host: self.host, _state: PhantomData }
    }
}

impl SmtpClient<Ready> {
    fn send(self, _to: &str, _body: &str) -> SmtpClient<Authenticated> {
        // ... RCPT TO, DATA, send body
        // Returns to Authenticated - can send another email
        SmtpClient { host: self.host, _state: PhantomData }
    }
}

impl<S> SmtpClient<S> {
    fn quit(self) {
        // ... QUIT
        // self is consumed, connection dropped
    }
}
```

The state diagram is encoded directly in the type system. You can't call `send()` without first calling `mail_from()`, which requires `authenticate()`, which requires `connect()`. Anyone reading the code knows the exact protocol sequence by looking at the types.

**Builder patterns with required fields.** Covered in the [API design post](/blog/api-design-in-rust-making-impossible-states-unrepresentable/), so I won't repeat it. The short version: `build()` only exists when all required fields are set, so forgetting a field is a compile error, not a runtime panic.

**Embedded HAL APIs.** This is the canonical real-world example. In the [embedded Rust post](/blog/embedded-rust-running-code-on-microcontrollers/), I mentioned the HAL layer. The `stm32f1xx-hal` crate uses typestate extensively for GPIO pins. A pin configured as `Input<Floating>` doesn't have a `set_high()` method. You must first convert it to an output mode:

```rust
// Pseudocode based on stm32f1xx-hal
let pin = gpioa.pa5.into_push_pull_output(&mut gpioa.crl);
// pin is now Pin<'A', 5, Output<PushPull>>
pin.set_high();

// This won't compile:
// let input_pin = gpioa.pa6; // Input<Floating> by default
// input_pin.set_high(); // ERROR: no method `set_high` on Input<Floating>
```

On a microcontroller, writing to a pin that's configured as input can damage hardware. This isn't a logic bug - it's a physical safety issue. The typestate pattern prevents it at compile time, and the GPIO pin wrapper compiles down to a single register write. The [official Embedded Rust Book](https://doc.rust-lang.org/beta/embedded-book/design-patterns/hal/gpio.html) documents this pattern as the recommended approach for HAL design.

**Resource lifecycle management.** Database transactions, file locks, crypto contexts - anything where you acquire a resource, use it, and must release it in a specific way. A `Transaction<Active>` has `commit()` and `rollback()`. A `Transaction<Committed>` has neither. The consumed `self` ensures exactly-once finalization.

## When enums win

**Runtime-determined state.** If the next state depends on user input, a network response, or a config file, you can't know it at compile time. A game character whose behavior changes based on player actions, a network connection that might be interrupted at any point, a document workflow where an admin can override the normal sequence - these need enum state machines.

**Persistence and serialization.** You need to save state to a database or send it over the wire. Enums serialize naturally with serde:

```rust
use serde::{Serialize, Deserialize};

#[derive(Serialize, Deserialize)]
enum OrderState {
    Draft,
    Submitted,
    Paid { amount: u64 },
    Shipped { tracking: String },
}
```

Typestate values can't be serialized directly because `Order<Draft>` and `Order<Paid>` are different types. You'd need to convert to an enum for serialization and back for deserialization - at which point you're maintaining both representations.

**Heterogeneous collections.** You have a list of orders in various states. With enums, it's `Vec<Order>`. With typestate, you'd need something like:

```rust
enum AnyOrder {
    Draft(Order<Draft>),
    Submitted(Order<Submitted>),
    Paid(Order<Paid>),
    Shipped(Order<Shipped>),
}
```

Which is... just an enum state machine with extra steps.

**Branching transitions with fallibility.** Sometimes a transition might fail and you need to stay in the current state. With enums, `pay()` returns `Result<(), PaymentError>` and the order stays `Submitted` on failure. With typestate, `pay(self)` has consumed `self` - if payment fails, you need to either return the original `Order<Submitted>` back or use a `Result<Order<Paid>, (Order<Submitted>, PaymentError)>` return type, which gets unwieldy fast.

## Advanced pattern: state-specific data

The basic typestate pattern puts all fields in the generic struct. But what about data that only exists in certain states - like a tracking number that only `Shipped` orders have?

One approach: use separate structs per state, with a shared core.

```rust
use std::marker::PhantomData;

struct Draft;
struct Paid;
struct Shipped;

struct OrderCore {
    id: String,
    items: Vec<String>,
}

struct Order<S> {
    core: OrderCore,
    _state: PhantomData<S>,
}

struct PaidOrder {
    core: OrderCore,
    amount: u64,
}

struct ShippedOrder {
    core: OrderCore,
    amount: u64,
    tracking: String,
}
```

This works but you lose the single generic type. Alternatively, use associated types through a trait:

```rust
use std::marker::PhantomData;

trait OrderState {
    type Data;
}

struct Draft;
struct Paid;
struct Shipped;

impl OrderState for Draft {
    type Data = (); // no extra data
}

impl OrderState for Paid {
    type Data = u64; // payment amount
}

impl OrderState for Shipped {
    type Data = (u64, String); // amount + tracking
}

struct Order<S: OrderState> {
    id: String,
    items: Vec<String>,
    state_data: S::Data,
    _state: PhantomData<S>,
}

impl Order<Draft> {
    fn new(id: &str) -> Self {
        Order {
            id: id.to_string(),
            items: Vec::new(),
            state_data: (),
            _state: PhantomData,
        }
    }

    fn pay(self, amount: u64) -> Order<Paid> {
        Order {
            id: self.id,
            items: self.items,
            state_data: amount,
            _state: PhantomData,
        }
    }
}

impl Order<Paid> {
    fn ship(self, tracking: &str) -> Order<Shipped> {
        Order {
            id: self.id,
            items: self.items,
            state_data: (self.state_data, tracking.to_string()),
            _state: PhantomData,
        }
    }

    fn amount(&self) -> u64 {
        self.state_data
    }
}

impl Order<Shipped> {
    fn tracking(&self) -> &str {
        &self.state_data.1
    }
}
```

Now each state carries exactly the data it needs, and the associated type system connects them. If you've read the [trait bounds post](/blog/rust-trait-bounds-where-clauses-associated-types-and-the-rest-of-the-iceberg/), you know associated types enforce a one-to-one mapping: each state has exactly one `Data` type.

The `state_data` field's size changes per state. `Order<Draft>` has a ZST `()` for `state_data` - zero bytes. `Order<Paid>` stores a `u64`. `Order<Shipped>` stores a `(u64, String)`. The struct truly only carries what the current state needs.

## Advanced pattern: sealed state traits

You might want to restrict which types can be used as states. Without a bound, someone could write `Order<i32>` - meaningless, but it compiles. A sealed trait fixes this:

```rust
mod sealed {
    pub trait State {}
}

pub struct Draft;
pub struct Submitted;
pub struct Paid;
pub struct Shipped;

impl sealed::State for Draft {}
impl sealed::State for Submitted {}
impl sealed::State for Paid {}
impl sealed::State for Shipped {}

pub struct Order<S: sealed::State> {
    id: String,
    _state: std::marker::PhantomData<S>,
}
```

The `sealed::State` trait lives in a private module. External code can't implement it, so the only valid state types are the ones you define. I covered sealed traits in the [phantom types post](/blog/phantom-types-in-rust-compile-time-constraints-with-zero-runtime-cost/) - same principle, applied to state machine states.

## Advanced pattern: fallible transitions

Real state machines have transitions that can fail. Payment might be declined. Authentication might fail. The trick is to return `Result` with the original state in the error variant:

```rust
use std::marker::PhantomData;

struct Submitted;
struct Paid;

struct Order<S> {
    id: String,
    _state: PhantomData<S>,
}

struct PaymentError {
    reason: String,
    order: Order<Submitted>, // give the order back
}

impl Order<Submitted> {
    fn try_pay(self, amount: u64) -> Result<Order<Paid>, PaymentError> {
        if amount == 0 {
            return Err(PaymentError {
                reason: "amount must be positive".to_string(),
                order: self, // return the order unchanged
            });
        }
        Ok(Order {
            id: self.id,
            _state: PhantomData,
        })
    }
}
```

The caller gets the `Order<Submitted>` back on failure, so they can retry or take a different path. The type system still guarantees that you only get an `Order<Paid>` through a successful payment.

This pattern is clunky when you have many error cases, but it preserves the core guarantee: state transitions are either complete and type-safe, or rolled back. No half-states.

## The hybrid approach

In practice, many systems combine both patterns. Use typestate for the hot path where correctness matters most, and wrap in an enum for storage and transport:

```rust
use std::marker::PhantomData;

// Typestate for business logic
struct Draft;
struct Paid;
struct Shipped;

struct Order<S> {
    id: String,
    items: Vec<String>,
    _state: PhantomData<S>,
}

// Enum for persistence
#[derive(serde::Serialize, serde::Deserialize)]
enum StoredOrder {
    Draft { id: String, items: Vec<String> },
    Paid { id: String, items: Vec<String>, amount: u64 },
    Shipped { id: String, items: Vec<String>, tracking: String },
}

// Convert typestate -> enum for saving
impl From<Order<Draft>> for StoredOrder {
    fn from(o: Order<Draft>) -> Self {
        StoredOrder::Draft { id: o.id, items: o.items }
    }
}

// Convert enum -> typestate for processing
// This is the boundary where runtime checking happens
impl StoredOrder {
    fn into_draft(self) -> Option<Order<Draft>> {
        match self {
            StoredOrder::Draft { id, items } => Some(Order {
                id,
                items,
                _state: PhantomData,
            }),
            _ => None,
        }
    }
}
```

The runtime check happens once, at the boundary (deserialization). After that, the typestate takes over and the compiler enforces every transition. This is the same "validate once at the boundary, carry the proof forward" principle from the [API design post](/blog/api-design-in-rust-making-impossible-states-unrepresentable/) - just applied to state rather than data.

## Crates that help

If the boilerplate bothers you, several crates generate typestate machinery:

[`typed-builder`](https://crates.io/crates/typed-builder) (v0.20) generates typestate builders from a derive macro. Each field gets a type parameter that transitions from "missing" to "set". The `build()` method only appears when all required fields are set.

[`statum`](https://crates.io/crates/statum) (v0.1) provides `#[state]` and `#[machine]` macros that generate the state structs, transition methods, and sealed traits from a declarative description. Less boilerplate, same compile-time guarantees.

[`statig`](https://crates.io/crates/statig) (v0.3) takes a different approach - hierarchical state machines with `#[state_machine]` derive, supporting guard conditions, entry/exit actions, and async transitions. It's closer to the UML statechart model, targeting embedded and `no_std` environments.

Each occupies a different point on the convenience-vs-control spectrum. For simple linear state machines (A -> B -> C -> D), hand-rolling is fine. For complex machines with branching, guards, and hierarchy, a crate saves significant effort.

## The compile error experience

One underrated aspect of typestate: the error messages are genuinely helpful. When you try to call a method that doesn't exist on the current state, Rust doesn't just say "method not found." It tells you which concrete type you have and which type has the method you wanted.

```
error[E0599]: no method named `ship` found for struct
              `Order<Submitted>` in the current scope
  --> src/main.rs:42:11
   |
5  | struct Order<S> {
   | --------------- method `ship` not found for this struct
...
42 |     order.ship("TRACK-1");
   |           ^^^^ method not found in `Order<Submitted>`
   |
   = note: the method was found for
           - `Order<Paid>`
```

"The method was found for `Order<Paid>`." That's the compiler telling you the fix: you need to pay first, then ship. A new developer reading this error message learns the state machine's rules without reading any documentation.

Compare this to the enum version's runtime error: `"can only ship a paid order"`. Same information, but discovered during execution instead of compilation. And only if the test covers that path.

## Limitations worth knowing

**Type parameter proliferation.** If your state machine has branching (state A can go to B or C depending on a condition), you still need runtime logic at the branch point. Typestate handles linear sequences gracefully but gets awkward with diamonds and cycles.

**Generic code over states.** Writing a function that works with "any order" requires a trait bound:

```rust
fn log_order<S: sealed::State>(order: &Order<S>) {
    println!("Order: {}", order.id());
}
```

This is fine for reading. But if you want to "do different things based on state" generically, you're back to something that looks like runtime dispatch - at which point you might as well use an enum.

**Ownership friction.** Every transition consumes `self`. If you need to "peek" at the next state without committing, or if a transition involves async I/O that might fail partway through, the ownership dance gets complicated. You might need intermediate states (`Order<PaymentPending>`) or `Result` types that return the original state on failure.

**Learning curve.** Developers unfamiliar with the pattern will be confused by `Order<Draft>` and `Order<Paid>` being different types. The concept is simple once you see it, but it's not obvious from the code alone. Document the state diagram somewhere - a comment with ASCII art or a link to a diagram is worth its weight.

## Decision framework

Use **typestate** when:
- The state machine is part of your public API and misuse should be a compile error
- The transitions are mostly linear and known at compile time
- Correctness is critical (financial transactions, protocol handlers, hardware drivers)
- The struct stays on the stack and doesn't need serialization

Use **enums** when:
- States change based on runtime data (user input, network responses, config)
- You need to store mixed-state values in collections
- The state machine has many branches, cycles, or conditional transitions
- You need serde support or database persistence
- The state machine is internal implementation detail, not a public API

Use **both** when:
- You want compile-time safety for business logic AND runtime flexibility for persistence
- The boundary between "compile-time known" and "runtime determined" is clear and narrow

The question isn't "which is better." It's "where does the state get determined?" If the state is determined by the shape of the code (which functions were called in what order), typestate encodes that perfectly. If the state is determined by data that arrives at runtime, enums handle that naturally.

## Wrapping up

The typestate pattern is one of those techniques that feels like it should have a cost. You're encoding an entire state diagram into the type system, getting compile-time verification of every transition, and the runtime cost is... zero. The state parameter is a zero-sized type. The `PhantomData` field occupies no memory. The transition methods compile to the same code as a plain function. All the checking happens during compilation and is erased completely in the binary.

It's not always the right tool. Enums are simpler, more flexible, and handle the messy real-world cases where states are determined at runtime. But for the cases where typestate fits - protocol implementations, builders, resource lifecycle, hardware abstractions - it eliminates entire classes of bugs by making them unwritable. Not uncaught. Not untested. *Unwritable*.

Start with the question: "Would it be a bug if someone called these methods in the wrong order?" If yes, and the order is known at compile time, typestate turns that bug into a compiler error. That's a trade worth making.
