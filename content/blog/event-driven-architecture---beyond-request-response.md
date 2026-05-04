+++
title = "Event-driven architecture - beyond request-response"
date = 2025-09-07
description = "Why request-response isn't always the answer, how event-driven systems work under the hood, and how to build a typed event bus in Rust with tokio channels."

[taxonomies]
tags = ["rust", "architecture", "async", "design-patterns"]
+++

Every HTTP framework teaches you the same flow: request comes in, handler runs, response goes out. You call a function, it returns a value, done. This model is simple, debuggable, and correct for probably 80% of what you'll build.

But then you hit the cases where it falls apart. User signs up and you need to send a welcome email, update analytics, provision a trial workspace, and notify the sales team on Slack. You stuff all of that into the signup handler. It takes 4 seconds. The user stares at a spinner. One of those downstream calls fails and now - what? Roll back the signup? Swallow the error? Retry everything?

This is where event-driven architecture earns its place. Not as a replacement for request-response, but as a complement for the cases where "do X then do Y then do Z" stops being a sane approach.

<!-- more -->

## The synchronous chain problem

Here's a typical signup handler that does too much:

```rust
async fn signup(
    State(state): State<AppState>,
    Json(input): Json<SignupInput>,
) -> Result<Json<User>, AppError> {
    let user = state.user_repo.create(CreateUser {
        email: input.email.clone(),
        name: input.name.clone(),
    }).await?;

    // Welcome email - 200ms
    state.mailer.send_welcome(&user.email).await?;

    // Analytics tracking - 150ms
    state.analytics.track("user_signed_up", &user.id).await?;

    // Provision trial workspace - 500ms
    state.workspace_service.provision_trial(&user.id).await?;

    // Notify sales on Slack - 100ms
    state.slack.notify_channel(
        "#new-signups",
        &format!("{} just signed up", user.email),
    ).await?;

    Ok(Json(user))
}
```

This has three problems:

**Latency compounds.** Each call adds to the response time. The user waits for the sum of all calls, not the longest one. If you're lucky, everything takes 950ms. If Slack is slow today, you're over 2 seconds for a signup.

**Failure coupling.** If the Slack notification fails, should the entire signup fail? Of course not - the user was already created. But that `?` propagates the error all the way up. You can replace `?` with `if let Err(e) = ...` for each call, but now you're writing error handling for side effects that shouldn't affect the core operation.

**Knowledge coupling.** The signup handler knows about mailers, analytics, workspaces, and Slack. Every new "thing that should happen when a user signs up" means editing this handler. Three teams touching the same function.

## Events invert the dependency

The event-driven version looks like this conceptually:

```rust
async fn signup(
    State(state): State<AppState>,
    Json(input): Json<SignupInput>,
) -> Result<Json<User>, AppError> {
    let user = state.user_repo.create(CreateUser {
        email: input.email.clone(),
        name: input.name.clone(),
    }).await?;

    state.events.emit(UserSignedUp {
        user_id: user.id.clone(),
        email: user.email.clone(),
        signed_up_at: Utc::now(),
    });

    Ok(Json(user))
}
```

The handler does one thing: create the user, emit a fact. It doesn't know or care who's listening. The email service subscribes to `UserSignedUp`. So does analytics. So does workspace provisioning. So does the Slack notifier. They each run independently - if Slack is down, the email still goes out.

This is the core idea: **events are facts about things that happened, not instructions to do things**. `UserSignedUp` is a fact. `SendWelcomeEmail` would be a command. The distinction matters because facts can have zero listeners, one listener, or twenty - the emitter doesn't care. Commands imply exactly one handler that must execute successfully.

## Events vs commands vs queries

Before building anything, it's worth being precise about terminology because people mix these up constantly:

| | Event | Command | Query |
|---|---|---|---|
| **Direction** | Broadcast (one-to-many) | Directed (one-to-one) | Directed (one-to-one) |
| **Semantics** | "This happened" | "Do this" | "Tell me this" |
| **Failure** | Listener fails independently | Caller gets the error | Caller gets the error |
| **Coupling** | Producer doesn't know consumers | Caller knows the handler | Caller knows the handler |
| **Examples** | `OrderPlaced`, `PaymentReceived` | `ChargeCard`, `SendEmail` | `GetUser`, `ListOrders` |

Request-response is the natural fit for commands and queries. You call a function, it does the thing (or tells you the thing), and returns. Events are different - they're fire-and-forget from the producer's perspective.

In practice, many systems use both. The signup handler executes a command (create user), then emits an event (user signed up). The command is synchronous and its failure matters. The event is asynchronous and its listeners can fail independently.

## Building an event bus in Rust

Let's build a real, typed event bus using tokio channels. Not a toy example - something you could actually use in production.

First, the event trait. Every event needs to be serializable (for logging, replay, dead-letter queues) and carry a topic for routing:

```rust
use std::any::Any;
use std::fmt::Debug;
use serde::{Serialize, Deserialize};

pub trait Event: Send + Sync + Debug + Any + 'static {
    fn topic(&self) -> &'static str;
    fn event_type(&self) -> &'static str;
}
```

Some concrete events:

```rust
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct UserSignedUp {
    pub user_id: String,
    pub email: String,
    pub signed_up_at: chrono::DateTime<chrono::Utc>,
}

impl Event for UserSignedUp {
    fn topic(&self) -> &'static str { "users" }
    fn event_type(&self) -> &'static str { "user_signed_up" }
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct OrderPlaced {
    pub order_id: String,
    pub user_id: String,
    pub total_cents: i64,
}

impl Event for OrderPlaced {
    fn topic(&self) -> &'static str { "orders" }
    fn event_type(&self) -> &'static str { "order_placed" }
}
```

### Choosing the right channel

Tokio gives you several channel types. If you've read [my post on tokio internals](/blog/understanding-tokio-the-rust-async-runtime-under-the-hood/), you know the runtime already. Now let's talk about the channels it provides and which fits an event bus:

**`tokio::sync::mpsc`** - multiple producers, single consumer. Fast, bounded or unbounded. But one consumer means one listener per channel. You'd need a separate channel per listener and fan-out manually. Not ideal for an event bus.

**`tokio::sync::broadcast`** - multiple producers, multiple consumers. Every receiver gets every message. This is pub/sub semantics - exactly what an event bus needs. The catch: messages must be `Clone`, and slow receivers can miss messages (they get a `Lagged` error with the count of missed items).

**`tokio::sync::watch`** - single producer, multiple consumers, but only the latest value is kept. Good for config changes, not for events where every occurrence matters.

For an event bus, `broadcast` is the natural fit. Let's look at what happens under the hood.

### Broadcast channel internals

The broadcast channel implementation lives in [`tokio/src/sync/broadcast.rs`](https://github.com/tokio-rs/tokio/blob/master/tokio/src/sync/broadcast.rs). It uses a fixed-size ring buffer protected by a mutex. When a sender publishes:

1. It acquires the lock on the shared state
2. Writes the value into the next slot in the ring buffer
3. Increments the tail position
4. Wakes all waiting receivers

Each receiver tracks its own position in the ring buffer. When it calls `recv()`:

1. If `pos < tail`, there are unread messages - it reads the next one and advances
2. If `pos == tail`, nothing new - it registers a waker and yields
3. If `tail - pos > capacity`, it missed messages - returns `RecvError::Lagged(n)`

This design means no heap allocation per message send (it reuses the ring buffer slots), but the buffer capacity is fixed at creation time. If you pick 1024 and your slowest consumer falls 1025 messages behind, it starts missing events. In practice you want to size this based on your worst-case burst and handle `Lagged` by re-syncing state.

### The event bus implementation

Here's a production-grade event bus using broadcast:

```rust
use std::collections::HashMap;
use std::sync::Arc;
use tokio::sync::{broadcast, RwLock};

type ErasedEvent = Arc<dyn Any + Send + Sync>;

pub struct EventBus {
    channels: RwLock<HashMap<&'static str, broadcast::Sender<ErasedEvent>>>,
    capacity: usize,
}

impl EventBus {
    pub fn new(capacity: usize) -> Self {
        Self {
            channels: RwLock::new(HashMap::new()),
            capacity,
        }
    }

    pub async fn emit<E: Event + Clone>(&self, event: E) {
        let topic = event.topic();
        let erased: ErasedEvent = Arc::new(event);

        let channels = self.channels.read().await;
        if let Some(tx) = channels.get(topic) {
            // send returns Err only if there are no receivers - that's fine,
            // events with no listeners are silently dropped
            let _ = tx.send(erased);
        }
    }

    pub async fn subscribe<E: Event + Clone>(
        &self,
        topic: &'static str,
    ) -> EventReceiver<E> {
        let mut channels = self.channels.write().await;
        let tx = channels
            .entry(topic)
            .or_insert_with(|| broadcast::channel(self.capacity).0);
        let rx = tx.subscribe();
        EventReceiver {
            rx,
            _phantom: std::marker::PhantomData,
        }
    }
}

pub struct EventReceiver<E> {
    rx: broadcast::Receiver<ErasedEvent>,
    _phantom: std::marker::PhantomData<E>,
}

impl<E: Event + Clone> EventReceiver<E> {
    pub async fn recv(&mut self) -> Result<E, EventError> {
        loop {
            match self.rx.recv().await {
                Ok(erased) => {
                    if let Some(event) = erased.downcast_ref::<E>() {
                        return Ok(event.clone());
                    }
                    // wrong type for this topic - skip it
                }
                Err(broadcast::error::RecvError::Lagged(n)) => {
                    return Err(EventError::Lagged(n));
                }
                Err(broadcast::error::RecvError::Closed) => {
                    return Err(EventError::Closed);
                }
            }
        }
    }
}

#[derive(Debug)]
pub enum EventError {
    Lagged(u64),
    Closed,
}
```

The key design decisions:

**Type erasure with `Arc<dyn Any>`.** We need a single broadcast channel per topic, but events are different types. `Arc<dyn Any>` lets us store anything and downcast on the receiver side. The `Arc` is necessary because broadcast requires `Clone`, and cloning an `Arc` is just a reference count increment - cheap.

**Topics as routing keys.** Rather than one global channel (noisy) or one channel per event type (too many), we group by topic. All user-related events go through the `"users"` channel. Receivers filter by type via downcasting. This keeps the channel count manageable while still allowing selective listening.

**Lagged handling.** Instead of silently dropping missed events, we surface a `Lagged` error. The listener can decide what to do - log a warning, resync from the database, or crash. Don't hide data loss.

### Wiring up listeners

Now let's connect listeners. Each listener is a spawned task that loops on `recv()`:

```rust
use std::sync::Arc;

struct EmailListener {
    mailer: Arc<dyn Mailer>,
}

impl EmailListener {
    async fn run(self, mut rx: EventReceiver<UserSignedUp>) {
        loop {
            match rx.recv().await {
                Ok(event) => {
                    if let Err(e) = self.mailer.send_welcome(&event.email).await {
                        tracing::error!(
                            user_id = %event.user_id,
                            error = %e,
                            "failed to send welcome email"
                        );
                        // TODO: push to retry queue
                    }
                }
                Err(EventError::Lagged(n)) => {
                    tracing::warn!("email listener lagged by {} events", n);
                }
                Err(EventError::Closed) => {
                    tracing::info!("event bus closed, shutting down email listener");
                    break;
                }
            }
        }
    }
}
```

And the startup wiring:

```rust
#[tokio::main]
async fn main() {
    let bus = Arc::new(EventBus::new(1024));

    // Subscribe before any events are emitted
    let signup_rx = bus.subscribe::<UserSignedUp>("users").await;
    let analytics_rx = bus.subscribe::<UserSignedUp>("users").await;
    let order_rx = bus.subscribe::<OrderPlaced>("orders").await;

    // Spawn listeners as background tasks
    let email_listener = EmailListener { mailer: Arc::new(SmtpMailer::new()) };
    tokio::spawn(email_listener.run(signup_rx));

    let analytics = AnalyticsListener { client: Arc::new(analytics_client) };
    tokio::spawn(analytics.run(analytics_rx));

    let fulfillment = FulfillmentListener { service: Arc::new(fulfillment_svc) };
    tokio::spawn(fulfillment.run(order_rx));

    // Start the HTTP server with the bus in its state
    let app = Router::new()
        .route("/signup", post(signup_handler))
        .with_state(AppState { bus: bus.clone(), /* ... */ });

    let listener = tokio::net::TcpListener::bind("0.0.0.0:3000").await.unwrap();
    axum::serve(listener, app).await.unwrap();
}
```

Each listener runs independently. If the email service crashes and restarts, the analytics listener keeps counting signups. If you add a new listener next week - say, a CRM sync - you add a new subscriber and spawn it. The signup handler doesn't change.

## Eventual consistency - the trade-off you're signing up for

Most event-driven tutorials gloss over this: you're trading strong consistency for temporal decoupling.

In the synchronous version, when the handler returns 200, you know the email was sent, analytics were tracked, and the workspace was provisioned. In the event-driven version, when the handler returns 200, you know the user was created and an event was emitted. That's it. The email might go out 50ms later. Or 5 seconds later if the mailer is backed up. Or never, if the listener crashed.

This is **eventual consistency**. The system will converge to the correct state, but at any given moment, some subsystem might be behind.

For most side effects, this is fine. The user doesn't need to see their analytics tracking complete before the signup response arrives. But for some operations, it's not acceptable. If the user's subscription must be active before they can access premium features, and subscription activation is an event listener, you have a race condition: the signup response redirects to the dashboard, the dashboard checks subscription status, and the listener hasn't run yet.

The rule: **core state goes in the synchronous path. Side effects go in events.**

Creating the user? Synchronous - the response needs the user ID. Sending the welcome email? Event - the user doesn't need to see it complete. Activating their subscription? Depends - if the next page load depends on it, keep it synchronous. If they can retry in 2 seconds, maybe an event is fine.

## When events earn their complexity

Events aren't free. You're adding a message bus, listeners, error handling for async failures, and a mental model that's harder to debug than a stack trace. Here's when the trade-off pays off:

### Side effects with independent failure

The signup example. Email, analytics, notifications - each can fail without affecting the others or the core operation. If you have 2+ side effects triggered by the same action, events clean up the code dramatically.

### Audit trails

Events are naturally an append-only log of what happened. If you persist them before dispatching to listeners, you have a complete history:

```rust
pub async fn emit_with_log<E: Event + Clone + Serialize>(
    &self,
    event: E,
    event_log: &EventLog,
) {
    // Persist first - if the listener crashes, we can replay
    let entry = EventEntry {
        id: Uuid::new_v4().to_string(),
        event_type: event.event_type().to_string(),
        topic: event.topic().to_string(),
        payload: serde_json::to_value(&event).unwrap(),
        emitted_at: Utc::now(),
    };
    event_log.append(entry).await;

    // Then dispatch to listeners
    self.emit(event).await;
}
```

Now you can answer "what happened to order #1234?" by querying the event log instead of reconstructing state from multiple tables. This is the foundation of event sourcing - deriving current state from a sequence of events rather than storing only the latest snapshot.

### Cross-service communication

When your monolith splits into services, direct function calls become network calls. Events over a message broker (RabbitMQ, NATS, Kafka) let services communicate without knowing each other's APIs. The in-process `broadcast` channel we built translates directly to a message broker topic - the programming model is identical, only the transport changes.

### Replay and reprocessing

Because events are facts, you can replay them. Deployed a bug in your analytics listener that miscounted signups for a week? Fix the bug, replay the `UserSignedUp` events from your log, and your analytics are corrected. Try doing that with synchronous side effects that were fire-and-forget from the handler.

## When events add pain for no gain

### Simple CRUD

If your endpoint creates a record and returns it, there's nothing to decouple. Adding an event bus to a `POST /api/items` that just inserts a row is over-engineering. If you've read [my post on the repository pattern](/blog/the-repository-pattern-abstracting-data-access-in-rust/), you know the repo already handles `create`, `find_by_id`, `update`, `delete`. No events needed for basic data operations.

### Request-response with strong consistency requirements

Payment processing: charge the card, update the order status, return the result. You can't tell the user "we probably charged your card, check back later." This needs to be synchronous with proper error handling and possibly distributed transactions. Events can trigger post-payment side effects (send receipt, update inventory), but the core payment flow stays synchronous.

### Low-traffic systems with few side effects

If your app has 10 users and the signup handler sends one email, adding an event bus is complexity theater. Just call the mailer directly. You can always refactor later when the complexity is justified.

### Debugging-heavy development phases

Event-driven systems are harder to debug. A stack trace shows you a linear path from request to response. An event-driven system shows you "event was emitted" and then you have to check which listeners ran, in what order, and whether they succeeded. During early development when you're still figuring out the domain model, synchronous code lets you iterate faster.

## Performance: channels under contention

Let's talk numbers. How do tokio broadcast channels perform when you're pushing real volume through them?

Benchmarks from the [crossfire project](https://github.com/frostyplanet/crossfire-rs/wiki/benchmark-v2.0-2025%E2%80%9006%E2%80%9027) (Intel i7-8550U, Rust 1.87.0) show throughput for different channel implementations in a synchronous MPSC bounded scenario:

| Producers x Consumers | crossbeam-channel | flume |
|---|---|---|
| 1x1 | 17.8 Me/s | 8.0 Me/s |
| 4x1 | 19.7 Me/s | 2.0 Me/s |
| 8x1 | 19.6 Me/s | 0.7 Me/s |

(Me/s = million elements per second)

Crossbeam dominates in sync contexts, especially under contention - flume drops from 8 Me/s to 0.7 Me/s as you add producers. But for an async event bus, tokio's channels have an advantage: they integrate with the runtime's waker system, so a receiver waiting on `recv()` doesn't block a thread - it yields back to the scheduler. Crossbeam channels block the thread, which in an async context means blocking the tokio worker, which can deadlock your runtime if you're not careful.

For most event buses, throughput isn't the bottleneck anyway. You're emitting maybe thousands of events per second, not millions. The bottleneck is usually the listener's work (sending emails, hitting external APIs). Use tokio broadcast for async, crossbeam for sync, and don't optimize until profiling tells you to.

## A macro for reducing boilerplate

Writing `impl Event for ...` for every event type gets tedious. A simple declarative macro - if you've read [my post on Rust macros](/blog/rust-macros-101-declarative-macros/), this pattern should look familiar:

```rust
macro_rules! define_event {
    (
        topic = $topic:expr,
        $vis:vis struct $name:ident {
            $($field_vis:vis $field:ident : $ty:ty),* $(,)?
        }
    ) => {
        #[derive(Debug, Clone, serde::Serialize, serde::Deserialize)]
        $vis struct $name {
            $($field_vis $field: $ty),*
        }

        impl Event for $name {
            fn topic(&self) -> &'static str { $topic }
            fn event_type(&self) -> &'static str { stringify!($name) }
        }
    };
}

// Usage:
define_event! {
    topic = "users",
    pub struct UserSignedUp {
        pub user_id: String,
        pub email: String,
        pub signed_up_at: chrono::DateTime<chrono::Utc>,
    }
}

define_event! {
    topic = "orders",
    pub struct OrderPlaced {
        pub order_id: String,
        pub user_id: String,
        pub total_cents: i64,
    }
}
```

You get type safety, serde derives, and the `Event` impl - all from a single macro invocation.

## Dead letters and retry patterns

In production, listeners fail. The email provider is down, the database is full, the external API returns 503. You need a strategy for these failures beyond "log and move on."

A dead-letter queue collects events that listeners couldn't process:

```rust
use tokio::sync::mpsc;

pub struct DeadLetterQueue {
    tx: mpsc::Sender<DeadLetter>,
}

#[derive(Debug, Serialize)]
pub struct DeadLetter {
    pub event_type: String,
    pub topic: String,
    pub payload: serde_json::Value,
    pub error: String,
    pub listener: String,
    pub failed_at: chrono::DateTime<chrono::Utc>,
    pub attempt: u32,
}

impl DeadLetterQueue {
    pub fn new(capacity: usize) -> (Self, mpsc::Receiver<DeadLetter>) {
        let (tx, rx) = mpsc::channel(capacity);
        (Self { tx }, rx)
    }

    pub async fn push(&self, letter: DeadLetter) {
        if self.tx.send(letter).await.is_err() {
            tracing::error!("dead letter queue full - events are being dropped");
        }
    }
}
```

A listener with retry and dead-letter support:

```rust
impl EmailListener {
    async fn handle_with_retry(
        &self,
        event: UserSignedUp,
        dlq: &DeadLetterQueue,
    ) {
        let max_retries = 3;
        let mut attempt = 0;

        loop {
            attempt += 1;
            match self.mailer.send_welcome(&event.email).await {
                Ok(_) => {
                    tracing::info!(
                        user_id = %event.user_id,
                        "welcome email sent"
                    );
                    return;
                }
                Err(e) if attempt < max_retries => {
                    let backoff = Duration::from_millis(100 * 2u64.pow(attempt));
                    tracing::warn!(
                        attempt,
                        backoff_ms = backoff.as_millis() as u64,
                        "email send failed, retrying: {e}"
                    );
                    tokio::time::sleep(backoff).await;
                }
                Err(e) => {
                    tracing::error!(
                        attempt,
                        "email send failed permanently: {e}"
                    );
                    dlq.push(DeadLetter {
                        event_type: event.event_type().to_string(),
                        topic: event.topic().to_string(),
                        payload: serde_json::to_value(&event).unwrap(),
                        error: e.to_string(),
                        listener: "email_listener".to_string(),
                        failed_at: Utc::now(),
                        attempt,
                    }).await;
                    return;
                }
            }
        }
    }
}
```

Exponential backoff (100ms, 200ms, 400ms) absorbs transient failures. After `max_retries`, the event goes to the dead-letter queue where an operator or automated process can investigate and replay it later.

## From in-process to distributed

Everything we've built runs inside a single process. When you need to cross process boundaries, the architecture stays the same - only the transport changes:

| In-process | Distributed equivalent |
|---|---|
| `tokio::sync::broadcast` | NATS JetStream, RabbitMQ fanout exchange |
| Topics | Subjects (NATS) or routing keys (RabbitMQ) |
| `EventReceiver::recv()` | Consumer subscription |
| Dead-letter queue | Broker-managed DLQ |
| Event log (database table) | Kafka log / NATS stream |

The `Event` trait, the listener pattern, the retry logic - all of it transfers directly. You swap the channel for a broker client and the rest of your code stays the same. This is why it's worth building the abstraction even for in-process use: when you eventually need to split into services (and if your system is successful, you will), the migration is swapping a transport layer, not rewriting your domain logic.

If you structured your code using the patterns from [the adapter pattern post](/blog/the-adapter-pattern-in-rust-wrapping-external-apis/), you already have a trait boundary between your listeners and the transport. Moving from tokio broadcast to NATS means writing a new adapter, not touching the listener logic.

## Quick decision framework

Before reaching for events, ask yourself:

1. **Does the action have side effects that can fail independently?** If yes - events.
2. **Do you need an audit trail of what happened?** If yes - event log.
3. **Will multiple teams need to react to the same trigger?** If yes - events (decoupling).
4. **Does the caller need to know the result of the side effect?** If yes - keep it synchronous.
5. **Is this just CRUD with no side effects?** If yes - direct repo call.

The best systems use both patterns. The request-response path handles the operation the user is waiting for. Events handle everything else.

## Wrapping up

Event-driven architecture isn't complicated in concept. Something happened, tell whoever cares, let them deal with it on their own schedule. The complexity comes from the operational concerns: what if a listener is slow, what if it crashes, what if events arrive out of order, what if you need to replay a week of events because a bug corrupted state.

Rust's type system helps more than you'd expect here. A typed event bus with compile-time checked event types catches an entire class of "published event X but listener expected event Y" bugs. The ownership model ensures events are properly cloned (or `Arc`-shared) across listener boundaries without data races. And tokio's channels give you async-native pub/sub without pulling in an external message broker for in-process communication.

Start with synchronous code. When you feel the pain - handlers doing too much, failure in one side effect killing unrelated side effects, multiple teams stepping on each other in the same handler - that's when events earn their keep. Not before.
