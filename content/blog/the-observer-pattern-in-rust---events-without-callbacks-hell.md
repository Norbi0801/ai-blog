+++
title = "The Observer pattern in Rust - events without callbacks hell"
date = 2026-03-17
description = "Building a type-safe EventBus in Rust with HashMap dispatch, generic typed events, async handlers, and Weak references for automatic cleanup."

[taxonomies]
tags = ["rust", "design-patterns", "async", "architecture"]
+++

You have a service that creates orders. When an order is created, five things need to happen: send a confirmation email, update inventory, log an audit trail, invalidate the product cache, and push a notification to the admin dashboard. You could call all five functions directly from the order handler. It compiles. It works. And now your `create_order` function knows about emails, inventory, caching, notifications, and audit logs. Change any one of those systems and you're editing order creation code. Add a sixth side effect and the function grows again. The coupling is total.

The Observer pattern breaks this. The order handler emits an event - "order created" - and walks away. It doesn't know who's listening or what they'll do. The email system, the cache invalidator, and the audit logger each subscribe to that event independently. They can be added, removed, or replaced without touching the handler that emits. The emitter and the observers are decoupled by design.

This post builds an `EventBus` in Rust from scratch. We start with the simplest version - a `HashMap` of topic strings to handler vectors - then add type safety with generics, async support, and automatic cleanup with weak references. Along the way, we'll compare it with Node.js's `EventEmitter` to see where Rust's type system turns runtime bugs into compile errors.

<!-- more -->

## The callback problem

If you've worked with event-driven JavaScript, you've seen this shape:

```javascript
const EventEmitter = require('events');
const emitter = new EventEmitter();

emitter.on('order_created', (order) => {
  sendEmail(order.email, () => {
    updateInventory(order.items, (err) => {
      if (err) {
        logError(err, () => {
          notifyAdmin(order.id, () => {
            // four levels deep and counting
          });
        });
      }
    });
  });
});
```

Node.js solved the nesting with Promises and `async`/`await`. But the `EventEmitter` itself still has deeper problems that no syntax sugar can fix:

**No type safety.** Events are identified by strings. Emit `"order_created"` in one place, listen for `"orderCreated"` in another - silent failure. The payload is `any`. You can emit an object with `{ email: "..." }` and the listener destructures `{ mail: "..." }` - no error until runtime.

**Memory leaks from forgotten listeners.** Register a listener, forget to call `.removeListener()`, and it stays alive for the process lifetime. Node even has a default `MaxListeners` warning (10 per event) specifically because this is so common.

**No ownership model.** Any code with a reference to the emitter can subscribe. Any code can emit. There's no compile-time guarantee that a listener won't outlive the data it references. In Rust terms, you'd have dangling closures everywhere.

Rust's type system, ownership model, and trait system let us fix all three of these problems at the language level. Not with runtime checks - with types.

## Version 1: the string-keyed EventBus

The simplest observer implementation mirrors what most languages do. A `HashMap` maps topic strings to a list of handler functions:

```rust
use std::collections::HashMap;

type Handler = Box<dyn Fn(&str) + Send + Sync>;

pub struct EventBus {
    listeners: HashMap<String, Vec<Handler>>,
}

impl EventBus {
    pub fn new() -> Self {
        Self {
            listeners: HashMap::new(),
        }
    }

    pub fn subscribe(&mut self, topic: &str, handler: Handler) {
        self.listeners
            .entry(topic.to_string())
            .or_default()
            .push(handler);
    }

    pub fn emit(&self, topic: &str, payload: &str) {
        if let Some(handlers) = self.listeners.get(topic) {
            for handler in handlers {
                handler(payload);
            }
        }
    }
}
```

Usage is straightforward:

```rust
fn main() {
    let mut bus = EventBus::new();

    bus.subscribe("order_created", Box::new(|payload| {
        println!("Audit log: {}", payload);
    }));

    bus.subscribe("order_created", Box::new(|payload| {
        println!("Send email for: {}", payload);
    }));

    bus.emit("order_created", "order-42");
    // Audit log: order-42
    // Send email for: order-42
}
```

This works. It also has every problem the Node.js version has. Topics are strings - misspell one and you get silence. Payloads are `&str` - you're serializing everything to strings or doing manual downcasting. The `dyn Fn` closures erase the handler's type signature. You can subscribe a handler that expects order data to a "user_deleted" event and the compiler won't blink.

Good enough for a prototype. Not good enough for production.

## Version 2: typed events with Any and TypeId

Rust has a tool for runtime type identification: [`std::any::Any`](https://doc.rust-lang.org/std/any/trait.Any.html). Every `'static` type has a unique `TypeId`. Instead of using string topics, we can use the event type itself as the key. The event *is* the topic.

```rust
use std::any::{Any, TypeId};
use std::collections::HashMap;

type Handler = Box<dyn Fn(&dyn Any) + Send + Sync>;

pub struct TypedEventBus {
    listeners: HashMap<TypeId, Vec<Handler>>,
}

impl TypedEventBus {
    pub fn new() -> Self {
        Self {
            listeners: HashMap::new(),
        }
    }

    pub fn subscribe<E: 'static>(&mut self, handler: impl Fn(&E) + Send + Sync + 'static) {
        let type_id = TypeId::of::<E>();
        let wrapped = Box::new(move |event: &dyn Any| {
            if let Some(e) = event.downcast_ref::<E>() {
                handler(e);
            }
        });
        self.listeners.entry(type_id).or_default().push(wrapped);
    }

    pub fn emit<E: 'static>(&self, event: &E) {
        let type_id = TypeId::of::<E>();
        if let Some(handlers) = self.listeners.get(&type_id) {
            for handler in handlers {
                handler(event);
            }
        }
    }
}
```

The key insight: `subscribe<E>` takes a concrete event type. Internally, it wraps the typed handler in a closure that does the `downcast_ref`. The `emit<E>` call uses `TypeId::of::<E>()` to look up the right handler list. The `Any` trait acts as an erasure boundary - but the public API is fully typed.

Define your events as plain structs:

```rust
struct OrderCreated {
    order_id: String,
    customer_email: String,
    total_cents: i64,
}

struct OrderCancelled {
    order_id: String,
    reason: String,
}
```

Now the compiler does the work:

```rust
fn main() {
    let mut bus = TypedEventBus::new();

    // This handler only receives OrderCreated events
    bus.subscribe(|event: &OrderCreated| {
        println!("Audit: order {} created, total: {}", event.order_id, event.total_cents);
    });

    // This handler only receives OrderCancelled events
    bus.subscribe(|event: &OrderCancelled| {
        println!("Notify: order {} cancelled: {}", event.order_id, event.reason);
    });

    let order = OrderCreated {
        order_id: "ord-42".into(),
        customer_email: "user@example.com".into(),
        total_cents: 4999,
    };

    bus.emit(&order);
    // Audit: order ord-42 created, total: 4999
    // (OrderCancelled handler is NOT called - different TypeId)
}
```

No string matching. No manual payload parsing. Subscribe a handler with the wrong type? It simply never fires - the `TypeId` lookup finds nothing. Try to access a field that doesn't exist on the event struct? Compile error.

If you've read the [strategy pattern post](/blog/the-strategy-pattern-in-rust-polymorphism-done-right/), you'll recognize the pattern. We're using `dyn Any` for dynamic dispatch internally but exposing a generic API externally. The caller never touches `Any` - they work with concrete types. The erasure is an implementation detail.

## Version 3: async handlers

The synchronous version blocks the emitter until every handler finishes. For an audit log writing to a file or a notification hitting an external API, that's a problem. The emitter shouldn't wait. If you've read the [tokio deep dive](/blog/understanding-tokio-the-rust-async-runtime-under-the-hood/), you know that async Rust runs futures on a task scheduler that multiplexes them across threads. We want each handler to be an async function that runs concurrently.

The challenge: you can't store `async fn` pointers directly in a `Vec`. An async function returns a `Future`, and different futures have different sizes. We need trait objects again, but for async.

Here's the approach - store handlers as boxed closures that return pinned futures:

```rust
use std::any::{Any, TypeId};
use std::collections::HashMap;
use std::future::Future;
use std::pin::Pin;
use std::sync::Arc;
use tokio::sync::RwLock;

type AsyncHandler = Box<
    dyn Fn(&dyn Any) -> Pin<Box<dyn Future<Output = ()> + Send + '_>>
        + Send
        + Sync,
>;

pub struct AsyncEventBus {
    listeners: Arc<RwLock<HashMap<TypeId, Vec<AsyncHandler>>>>,
}

impl AsyncEventBus {
    pub fn new() -> Self {
        Self {
            listeners: Arc::new(RwLock::new(HashMap::new())),
        }
    }

    pub async fn subscribe<E: Send + Sync + 'static>(
        &self,
        handler: impl Fn(&E) -> Pin<Box<dyn Future<Output = ()> + Send + '_>>
            + Send
            + Sync
            + 'static,
    ) {
        let type_id = TypeId::of::<E>();
        let wrapped: AsyncHandler = Box::new(move |event: &dyn Any| {
            if let Some(e) = event.downcast_ref::<E>() {
                handler(e)
            } else {
                Box::pin(async {})
            }
        });
        self.listeners
            .write()
            .await
            .entry(type_id)
            .or_default()
            .push(wrapped);
    }

    pub async fn emit<E: Send + Sync + 'static>(&self, event: &E) {
        let type_id = TypeId::of::<E>();
        let listeners = self.listeners.read().await;
        if let Some(handlers) = listeners.get(&type_id) {
            let futures: Vec<_> = handlers
                .iter()
                .map(|h| h(event))
                .collect();
            futures::future::join_all(futures).await;
        }
    }
}
```

A few things to notice:

**`Arc<RwLock<...>>`** instead of a plain `HashMap`. The bus can be shared across tasks. `RwLock` gives us concurrent reads (multiple `emit` calls) and exclusive writes (subscribe). This is tokio's [`RwLock`](https://docs.rs/tokio/1.50.0/tokio/sync/struct.RwLock.html), not `std::sync::RwLock` - it's async-aware and won't block the runtime.

**`Pin<Box<dyn Future<...>>>`** is the return type of each handler. This is the standard way to erase async function types. Every handler returns a boxed, pinned future. The `join_all` call runs them all concurrently.

**Handlers run concurrently, not sequentially.** `join_all` polls all futures together. If one handler takes 200ms and another takes 10ms, the total time is ~200ms, not 210ms.

Usage with a helper macro to reduce the Pin<Box<...>> boilerplate:

```rust
macro_rules! async_handler {
    ($closure:expr) => {
        |event| Box::pin($closure(event))
    };
}

#[tokio::main]
async fn main() {
    let bus = AsyncEventBus::new();

    bus.subscribe(async_handler!(|event: &OrderCreated| async move {
        println!("Sending email to {}", event.customer_email);
        // simulate async work
        tokio::time::sleep(std::time::Duration::from_millis(100)).await;
        println!("Email sent for order {}", event.order_id);
    }))
    .await;

    bus.subscribe(async_handler!(|event: &OrderCreated| async move {
        println!("Logging audit for order {}", event.order_id);
    }))
    .await;

    let order = OrderCreated {
        order_id: "ord-42".into(),
        customer_email: "user@example.com".into(),
        total_cents: 4999,
    };

    bus.emit(&order).await;
}
```

Compare this to the Node.js version. Same pattern - subscribe, emit, handlers fire. But the event type is checked at compile time, the payload is a concrete struct (not an untyped object), and async handlers run concurrently on the tokio runtime without callback nesting.

## Fire-and-forget with tokio::spawn

The `emit` above still waits for handlers to finish (it `await`s `join_all`). Sometimes you want true fire-and-forget - emit the event and return immediately. The handlers run in background tasks:

```rust
impl AsyncEventBus {
    pub async fn emit_detached<E: Send + Sync + 'static>(&self, event: Arc<E>) {
        let type_id = TypeId::of::<E>();
        let listeners = self.listeners.read().await;
        if let Some(handlers) = listeners.get(&type_id) {
            for handler in handlers {
                let future = handler(&*event);
                tokio::spawn(async move {
                    future.await;
                });
            }
        }
    }
}
```

The event is wrapped in `Arc` so it can be shared across spawned tasks. Each handler gets its own tokio task. The emitter returns as soon as the tasks are spawned - it doesn't wait for any handler to finish.

This is the pattern you want for things like audit logging. The order handler shouldn't wait 50ms for the audit log to flush to disk.

## Decoupling: who knows about whom?

Here's the architectural payoff. Look at what each component needs to know:

```
create_order handler  -->  knows: OrderCreated struct
audit_logger          -->  knows: OrderCreated struct
email_sender          -->  knows: OrderCreated struct
cache_invalidator     -->  knows: OrderCreated struct
```

The handler doesn't import the logger. The logger doesn't import the handler. They both import the event struct - a plain data type with no behavior. The `EventBus` is the wiring layer. It connects producers to consumers at runtime, but neither side depends on the other at compile time.

This is the same decoupling principle behind the [adapter pattern](/blog/the-adapter-pattern-in-rust-wrapping-external-apis/) and the [repository pattern](/blog/the-repository-pattern-abstracting-data-access-in-rust/) - put an abstraction between the caller and the implementation. The difference is that adapters and repositories decouple two specific components. The observer pattern decouples one-to-many: one emitter, N listeners, where N can change at runtime.

In practice, structure it as a shared `events` module:

```rust
// src/events.rs
pub struct OrderCreated {
    pub order_id: String,
    pub customer_email: String,
    pub total_cents: i64,
}

pub struct OrderCancelled {
    pub order_id: String,
    pub reason: String,
}

pub struct InventoryLow {
    pub product_id: String,
    pub remaining: u32,
    pub threshold: u32,
}
```

Each module subscribes to what it cares about. The event struct is the contract. Change a field and the compiler tells you every subscriber that needs updating. Compare that to Node.js where renaming a JSON field means grep-and-pray.

## Weak references for auto-cleanup

One persistent problem with observer implementations: subscribers that outlive their usefulness. A UI component subscribes to events, gets destroyed, but the subscription stays in the bus. The handler closure captures a reference to the now-gone component. In garbage-collected languages, this prevents the component from being collected - a memory leak. In Rust, the borrow checker prevents dangling references, but closures that capture `Arc<T>` will keep `T` alive indefinitely.

The fix: [`Weak`](https://doc.rust-lang.org/std/sync/struct.Weak.html) references. Instead of the event bus holding a strong `Arc` to the subscriber, it holds a `Weak`. When all strong references to the subscriber are dropped, the `Weak::upgrade()` call returns `None`, and the bus knows to skip (and eventually remove) that handler.

```rust
use std::sync::{Arc, Weak};

pub trait EventHandler<E>: Send + Sync {
    fn handle(&self, event: &E);
}

pub struct WeakEventBus<E> {
    handlers: Vec<Weak<dyn EventHandler<E>>>,
}

impl<E> WeakEventBus<E> {
    pub fn new() -> Self {
        Self {
            handlers: Vec::new(),
        }
    }

    pub fn subscribe(&mut self, handler: &Arc<dyn EventHandler<E>>) {
        self.handlers.push(Arc::downgrade(handler));
    }

    pub fn emit(&mut self, event: &E) {
        // Retain only handlers that are still alive
        self.handlers.retain(|weak| weak.strong_count() > 0);

        for weak in &self.handlers {
            if let Some(handler) = weak.upgrade() {
                handler.handle(event);
            }
        }
    }
}
```

The `retain` call during `emit` is the cleanup. Dead weak references get pruned automatically. No explicit unsubscribe needed - drop the subscriber and it removes itself on the next emit cycle.

Here's how it looks in use:

```rust
struct AuditLogger;

impl EventHandler<OrderCreated> for AuditLogger {
    fn handle(&self, event: &OrderCreated) {
        println!("AUDIT: order {} total={}", event.order_id, event.total_cents);
    }
}

struct EmailSender;

impl EventHandler<OrderCreated> for EmailSender {
    fn handle(&self, event: &OrderCreated) {
        println!("EMAIL: sending to {}", event.customer_email);
    }
}

fn main() {
    let mut bus = WeakEventBus::<OrderCreated>::new();

    let logger: Arc<dyn EventHandler<OrderCreated>> = Arc::new(AuditLogger);
    let emailer: Arc<dyn EventHandler<OrderCreated>> = Arc::new(EmailSender);

    bus.subscribe(&logger);
    bus.subscribe(&emailer);

    let order = OrderCreated {
        order_id: "ord-1".into(),
        customer_email: "a@b.com".into(),
        total_cents: 2500,
    };

    bus.emit(&order);
    // AUDIT: order ord-1 total=2500
    // EMAIL: sending to a@b.com

    drop(emailer);  // EmailSender is gone

    bus.emit(&order);
    // AUDIT: order ord-1 total=2500
    // (no email - the Weak upgraded to None, handler was pruned)
}
```

No `unsubscribe` method. No callback IDs to track. No manual cleanup. The ownership system handles it. This is fundamentally impossible in JavaScript's `EventEmitter` - there's no concept of "this reference is weak" in the GC model. You either hold a reference or you don't. Node's [WeakRef](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/WeakRef) exists but isn't integrated into `EventEmitter` and has no deterministic cleanup timing.

## Use cases

Three concrete scenarios where the observer pattern pays for its complexity.

### Audit logging

Every state-changing operation emits an event. An audit subscriber captures them all:

```rust
struct AuditEvent {
    action: String,
    entity_type: String,
    entity_id: String,
    actor_id: String,
    timestamp: chrono::DateTime<chrono::Utc>,
}

// The audit subscriber writes to a separate audit table/file.
// It never blocks the main operation.
// If the audit system is down, the main operation still succeeds.
```

The key benefit: adding audit logging to a new operation means emitting one event. You don't touch the audit module. You don't touch the operation's handler beyond the emit call.

### Cache invalidation

The hardest problem in computer science (after naming things). The observer pattern makes it explicit:

```rust
struct ProductUpdated {
    product_id: String,
    changed_fields: Vec<String>,
}

// Cache subscriber
impl EventHandler<ProductUpdated> for CacheInvalidator {
    fn handle(&self, event: &ProductUpdated) {
        self.cache.invalidate(&format!("product:{}", event.product_id));
        if event.changed_fields.contains(&"price".to_string()) {
            self.cache.invalidate(&format!("pricing:{}", event.product_id));
        }
    }
}
```

The product update handler doesn't know about caching. The cache invalidator doesn't know how products are updated. They share a struct definition - that's it.

### Notification fan-out

One event, multiple notification channels:

```rust
struct UserSignedUp {
    user_id: String,
    email: String,
    plan: String,
}

// Three separate subscribers, independently deployed/tested:
// 1. WelcomeEmailSender - sends onboarding email
// 2. SlackNotifier - posts to #new-signups channel
// 3. AnalyticsTracker - records conversion event
```

Adding a fourth channel (push notification, SMS, carrier pigeon) means adding one subscriber. Zero changes to the signup handler.

## What about tokio's built-in channels?

Before building your own `EventBus`, consider whether tokio's channel types already cover your case:

**[`broadcast`](https://docs.rs/tokio/1.50.0/tokio/sync/broadcast/index.html)** - Multiple producers, multiple consumers. Every consumer gets every message. This is the closest thing to an event bus built into tokio. The catch: it's single-type. A `broadcast::Sender<OrderCreated>` can only send `OrderCreated`. For multiple event types, you'd need an enum wrapper or multiple channels.

```rust
use tokio::sync::broadcast;

let (tx, _) = broadcast::channel::<OrderCreated>(100);

let mut rx1 = tx.subscribe();
let mut rx2 = tx.subscribe();

// Spawn listeners
tokio::spawn(async move {
    while let Ok(event) = rx1.recv().await {
        println!("Logger: {}", event.order_id);
    }
});

tokio::spawn(async move {
    while let Ok(event) = rx2.recv().await {
        println!("Email: {}", event.customer_email);
    }
});

tx.send(OrderCreated {
    order_id: "ord-42".into(),
    customer_email: "user@example.com".into(),
    total_cents: 4999,
})?;
```

**[`watch`](https://docs.rs/tokio/1.50.0/tokio/sync/watch/index.html)** - Single producer, multiple consumers, but consumers only see the latest value. Good for config changes or status updates, not for event streams.

**When to build your own:** When you need multiple event types routed through a single bus, when you want the `Weak` reference cleanup, or when you need middleware-style hooks (logging all events, filtering, etc.). The `TypedEventBus` we built handles heterogeneous events through a single API. Tokio's channels are homogeneous - one type per channel.

## Memory layout

What does our `TypedEventBus` actually look like in memory? Understanding this helps you reason about performance.

```
TypedEventBus
  listeners: HashMap<TypeId, Vec<Handler>>
    TypeId(OrderCreated) -> [
      Box<dyn Fn(&dyn Any)>   // 16 bytes: vtable ptr + data ptr
      Box<dyn Fn(&dyn Any)>   // 16 bytes
    ]
    TypeId(OrderCancelled) -> [
      Box<dyn Fn(&dyn Any)>   // 16 bytes
    ]
```

Each `TypeId` is 8 bytes (it's a `u64` hash internally). Each boxed handler is a fat pointer: 8 bytes for the data pointer, 8 bytes for the vtable pointer. The `HashMap` itself uses a flat array with Robin Hood hashing (Rust's `hashbrown` implementation). For a bus with 10 event types and 50 total handlers, the overhead is roughly: 10 * 8 (TypeId keys) + 10 * 24 (Vec metadata) + 50 * 16 (boxed handlers) = ~1.1 KB. Negligible.

The `downcast_ref` inside each handler is a single `TypeId` comparison - one integer compare instruction. No string matching, no hash lookup.

## When not to use the observer pattern

The pattern introduces indirection. When you `emit`, there's no way to know at the call site what will run. Stack traces through an event bus are harder to follow. If you have exactly one consumer that will never change, a direct function call is simpler and more traceable.

Specifically avoid it when:

- **There's only one listener and always will be.** Direct call is clearer.
- **Order matters.** Observers don't guarantee execution order (unless you add priority, which is more complexity).
- **You need a return value.** The observer pattern is fire-and-forget. If the emitter needs a result from the handler, use a different pattern (request-response, or the strategy pattern which I covered [previously](/blog/the-strategy-pattern-in-rust-polymorphism-done-right/)).
- **Debug traceability is critical.** Following event flow through a bus requires logging/tracing at emit and handler entry points. Direct calls give you a stack trace for free.

## Summary

Here's the progression:

| Version | Key type | Events identified by | Type-safe | Async |
|---------|---------|---------------------|-----------|-------|
| String-keyed | `HashMap<String, Vec<Box<dyn Fn>>>` | String topic | No | No |
| TypeId-keyed | `HashMap<TypeId, Vec<Box<dyn Fn(&dyn Any)>>>` | Struct TypeId | Yes | No |
| Async | `Arc<RwLock<HashMap<TypeId, Vec<AsyncHandler>>>>` | Struct TypeId | Yes | Yes |
| Weak | `Vec<Weak<dyn EventHandler<E>>>` | Generic type parameter | Yes | No |

The Rust version fixes the three fundamental problems with JavaScript's `EventEmitter`: events are types, not strings (typos become compile errors). Payloads are structs, not untyped objects (field access is checked). Weak references enable automatic cleanup (no forgotten `.off()` calls).

The pattern costs you indirection and debuggability. It pays you back in decoupling - and for systems with multiple independent side effects triggered by the same action, that tradeoff is worth it every time.
