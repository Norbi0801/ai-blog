+++
title = "Event sourcing - storing what happened, not what is"
date = 2025-09-10
description = "Why storing every state change as an immutable event gives you audit trails, time travel, and debugging superpowers - and when the complexity isn't worth it."

[taxonomies]
tags = ["rust", "architecture", "design-patterns", "databases"]
+++

Most applications store current state. You have an `orders` table. When an order ships, you UPDATE the status column from `confirmed` to `shipped`. The old value is gone. If someone asks "when was this order confirmed, by whom, and what was the shipping address at that time?" you shrug, because that information was overwritten.

Event sourcing flips this. Instead of updating rows, you append events: `OrderPlaced`, `OrderConfirmed`, `AddressChanged`, `OrderShipped`. The current state of an order isn't stored directly - it's computed by replaying those events from the beginning. The events are the source of truth. The current state is a derivation.

If you haven't read [the CQRS post](/blog/cqrs-in-practice-separating-reads-from-writes/) or the [event-driven architecture post](/blog/event-driven-architecture-beyond-request-response/), go read those first. Event sourcing builds directly on both ideas: CQRS gives you the read/write separation that makes event sourcing practical, and event-driven architecture gives you the mechanics of emitting and handling events. This post assumes you're comfortable with both.

<!-- more -->

## The core idea

Traditional state storage works like a whiteboard. You write the current state, and when it changes, you erase and write the new state. Event sourcing works like an accounting ledger. Every transaction is recorded in order. You never erase entries. To know the current balance, you sum all the transactions.

This isn't an abstract analogy. Accounting has worked this way for centuries because it solves real problems: you can audit every change, detect errors by replaying the math, and answer questions about any point in time.

In software, an event-sourced system has three core components:

**Events** - immutable records of things that happened. `OrderPlaced { order_id, customer_id, items, placed_at }`. Past tense, factual, never modified.

**An event store** - an append-only log. You write events to the end. You never update or delete them. Each event belongs to a stream (usually identified by an aggregate ID like `order-123`).

**Aggregates** - domain objects that enforce business rules. An aggregate loads its event history, replays them to rebuild state, validates commands against that state, and emits new events. The aggregate itself is never persisted - only its events are.

## Events as the source of truth

Here's what events look like for a bank account:

```rust
use chrono::{DateTime, Utc};
use serde::{Serialize, Deserialize};

#[derive(Debug, Clone, Serialize, Deserialize)]
pub enum BankAccountEvent {
    Opened {
        account_id: String,
        owner: String,
        opened_at: DateTime<Utc>,
    },
    Deposited {
        amount_cents: i64,
        description: String,
        at: DateTime<Utc>,
    },
    Withdrawn {
        amount_cents: i64,
        description: String,
        at: DateTime<Utc>,
    },
    Frozen {
        reason: String,
        at: DateTime<Utc>,
    },
    Unfrozen {
        at: DateTime<Utc>,
    },
}
```

Each variant is a fact. `Deposited { amount_cents: 50000, description: "salary" }` is not a command ("deposit this") - it's a record ("this was deposited"). The distinction matters. Commands can be rejected. Events already happened. You never validate an event; you validate the command that produces it.

## The aggregate: where business rules live

The aggregate holds transient state rebuilt from events. It never touches a database directly - it only consumes events and produces new ones.

```rust
#[derive(Debug, Default)]
pub struct BankAccount {
    pub id: String,
    pub owner: String,
    pub balance_cents: i64,
    pub is_frozen: bool,
    pub version: u64,
}

impl BankAccount {
    /// Replay a single event to update internal state.
    /// No validation here - events are facts that already happened.
    pub fn apply(&mut self, event: &BankAccountEvent) {
        match event {
            BankAccountEvent::Opened { account_id, owner, .. } => {
                self.id = account_id.clone();
                self.owner = owner.clone();
            }
            BankAccountEvent::Deposited { amount_cents, .. } => {
                self.balance_cents += amount_cents;
            }
            BankAccountEvent::Withdrawn { amount_cents, .. } => {
                self.balance_cents -= amount_cents;
            }
            BankAccountEvent::Frozen { .. } => {
                self.is_frozen = true;
            }
            BankAccountEvent::Unfrozen { .. } => {
                self.is_frozen = false;
            }
        }
        self.version += 1;
    }

    /// Rebuild state from a full event history.
    pub fn from_events(events: &[BankAccountEvent]) -> Self {
        let mut account = Self::default();
        for event in events {
            account.apply(event);
        }
        account
    }
}
```

The `apply` method is mechanical. No business logic, no validation, no side effects. It just mutates state based on what happened. This is important because `apply` is called both when replaying history and when processing new commands - it must be deterministic.

## Commands: where validation happens

Commands are requests to do something. Unlike events, they can be rejected. The command handler loads the aggregate's event history, rebuilds state, validates the command against that state, and if valid, returns new events.

```rust
#[derive(Debug)]
pub enum BankAccountCommand {
    Open { account_id: String, owner: String },
    Deposit { amount_cents: i64, description: String },
    Withdraw { amount_cents: i64, description: String },
    Freeze { reason: String },
    Unfreeze,
}

#[derive(Debug, thiserror::Error)]
pub enum BankAccountError {
    #[error("account already exists")]
    AlreadyExists,
    #[error("account is frozen")]
    Frozen,
    #[error("insufficient funds: balance {balance_cents}, withdrawal {requested_cents}")]
    InsufficientFunds { balance_cents: i64, requested_cents: i64 },
    #[error("deposit amount must be positive")]
    InvalidAmount,
}

impl BankAccount {
    /// Validate a command against current state and return events if valid.
    pub fn handle(
        &self,
        cmd: BankAccountCommand,
    ) -> Result<Vec<BankAccountEvent>, BankAccountError> {
        match cmd {
            BankAccountCommand::Open { account_id, owner } => {
                if !self.id.is_empty() {
                    return Err(BankAccountError::AlreadyExists);
                }
                Ok(vec![BankAccountEvent::Opened {
                    account_id,
                    owner,
                    opened_at: Utc::now(),
                }])
            }
            BankAccountCommand::Deposit { amount_cents, description } => {
                if amount_cents <= 0 {
                    return Err(BankAccountError::InvalidAmount);
                }
                if self.is_frozen {
                    return Err(BankAccountError::Frozen);
                }
                Ok(vec![BankAccountEvent::Deposited {
                    amount_cents,
                    description,
                    at: Utc::now(),
                }])
            }
            BankAccountCommand::Withdraw { amount_cents, description } => {
                if amount_cents <= 0 {
                    return Err(BankAccountError::InvalidAmount);
                }
                if self.is_frozen {
                    return Err(BankAccountError::Frozen);
                }
                if self.balance_cents < amount_cents {
                    return Err(BankAccountError::InsufficientFunds {
                        balance_cents: self.balance_cents,
                        requested_cents: amount_cents,
                    });
                }
                Ok(vec![BankAccountEvent::Withdrawn {
                    amount_cents,
                    description,
                    at: Utc::now(),
                }])
            }
            BankAccountCommand::Freeze { reason } => {
                Ok(vec![BankAccountEvent::Frozen {
                    reason,
                    at: Utc::now(),
                }])
            }
            BankAccountCommand::Unfreeze => {
                Ok(vec![BankAccountEvent::Unfrozen {
                    at: Utc::now(),
                }])
            }
        }
    }
}
```

Notice the pattern. The `handle` method is a pure function of (current state + command) -> Result<events>. It doesn't write to a database. It doesn't call external services. It checks invariants and returns facts. This makes it trivially testable:

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn cannot_withdraw_more_than_balance() {
        let events = vec![
            BankAccountEvent::Opened {
                account_id: "acc-1".into(),
                owner: "Alice".into(),
                opened_at: Utc::now(),
            },
            BankAccountEvent::Deposited {
                amount_cents: 10000,
                description: "initial deposit".into(),
                at: Utc::now(),
            },
        ];

        let account = BankAccount::from_events(&events);
        let result = account.handle(BankAccountCommand::Withdraw {
            amount_cents: 15000,
            description: "rent".into(),
        });

        assert!(matches!(
            result,
            Err(BankAccountError::InsufficientFunds { .. })
        ));
    }

    #[test]
    fn frozen_account_rejects_deposits() {
        let events = vec![
            BankAccountEvent::Opened {
                account_id: "acc-1".into(),
                owner: "Alice".into(),
                opened_at: Utc::now(),
            },
            BankAccountEvent::Frozen {
                reason: "suspicious activity".into(),
                at: Utc::now(),
            },
        ];

        let account = BankAccount::from_events(&events);
        let result = account.handle(BankAccountCommand::Deposit {
            amount_cents: 5000,
            description: "paycheck".into(),
        });

        assert!(matches!(result, Err(BankAccountError::Frozen)));
    }

    #[test]
    fn balance_tracks_deposits_and_withdrawals() {
        let events = vec![
            BankAccountEvent::Opened {
                account_id: "acc-1".into(),
                owner: "Alice".into(),
                opened_at: Utc::now(),
            },
            BankAccountEvent::Deposited {
                amount_cents: 50000,
                description: "salary".into(),
                at: Utc::now(),
            },
            BankAccountEvent::Withdrawn {
                amount_cents: 12000,
                description: "groceries".into(),
                at: Utc::now(),
            },
            BankAccountEvent::Deposited {
                amount_cents: 3000,
                description: "refund".into(),
                at: Utc::now(),
            },
        ];

        let account = BankAccount::from_events(&events);
        assert_eq!(account.balance_cents, 41000);
        assert_eq!(account.version, 4);
    }
}
```

No mocks. No database setup. No async runtime. You create events, build state, and assert business rules. This is the testing superpower of event sourcing - your domain logic is completely decoupled from infrastructure.

## The event store

Events need to be persisted somewhere. The event store is an append-only log with a few key properties: events belong to a stream (one per aggregate instance), they're ordered within that stream, and they support optimistic concurrency via version numbers.

Here's a minimal event store trait:

```rust
use async_trait::async_trait;

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct StoredEvent {
    pub stream_id: String,
    pub version: u64,
    pub event_type: String,
    pub payload: serde_json::Value,
    pub metadata: EventMetadata,
    pub stored_at: DateTime<Utc>,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct EventMetadata {
    pub correlation_id: String,
    pub causation_id: Option<String>,
    pub user_id: Option<String>,
}

#[async_trait]
pub trait EventStore: Send + Sync {
    /// Append events to a stream. Fails if expected_version doesn't match
    /// the current stream version (optimistic concurrency).
    async fn append(
        &self,
        stream_id: &str,
        expected_version: u64,
        events: Vec<NewEvent>,
    ) -> Result<u64, EventStoreError>;

    /// Load all events for a stream, in order.
    async fn load_stream(
        &self,
        stream_id: &str,
    ) -> Result<Vec<StoredEvent>, EventStoreError>;

    /// Load events for a stream starting from a specific version.
    async fn load_stream_from(
        &self,
        stream_id: &str,
        from_version: u64,
    ) -> Result<Vec<StoredEvent>, EventStoreError>;
}

pub struct NewEvent {
    pub event_type: String,
    pub payload: serde_json::Value,
    pub metadata: EventMetadata,
}

#[derive(Debug, thiserror::Error)]
pub enum EventStoreError {
    #[error("concurrency conflict: expected version {expected}, found {actual}")]
    ConcurrencyConflict { expected: u64, actual: u64 },
    #[error("storage error: {0}")]
    Storage(String),
}
```

The `expected_version` parameter is critical. It prevents two concurrent commands from corrupting the event stream. If two requests try to withdraw from the same account simultaneously:

1. Both load the stream at version 5 (balance: $500)
2. Request A appends `Withdrawn { 300 }` with `expected_version: 5` - succeeds, stream is now version 6
3. Request B appends `Withdrawn { 400 }` with `expected_version: 5` - fails with `ConcurrencyConflict`

Without this check, both withdrawals would succeed and the account would go to -$200. This is the same optimistic concurrency pattern you'd use with a version column in a traditional UPDATE, but here it protects the append rather than the overwrite.

A SQLite implementation of this store:

```rust
use sqlx::SqlitePool;

pub struct SqliteEventStore {
    pool: SqlitePool,
}

impl SqliteEventStore {
    pub async fn new(pool: SqlitePool) -> Result<Self, sqlx::Error> {
        sqlx::query(
            "CREATE TABLE IF NOT EXISTS events (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                stream_id TEXT NOT NULL,
                version INTEGER NOT NULL,
                event_type TEXT NOT NULL,
                payload TEXT NOT NULL,
                metadata TEXT NOT NULL,
                stored_at TEXT NOT NULL,
                UNIQUE(stream_id, version)
            )"
        )
        .execute(&pool)
        .await?;

        sqlx::query(
            "CREATE INDEX IF NOT EXISTS idx_events_stream
             ON events(stream_id, version)"
        )
        .execute(&pool)
        .await?;

        Ok(Self { pool })
    }
}

#[async_trait]
impl EventStore for SqliteEventStore {
    async fn append(
        &self,
        stream_id: &str,
        expected_version: u64,
        events: Vec<NewEvent>,
    ) -> Result<u64, EventStoreError> {
        let mut tx = self.pool.begin().await
            .map_err(|e| EventStoreError::Storage(e.to_string()))?;

        // Check current version
        let row: Option<(i64,)> = sqlx::query_as(
            "SELECT MAX(version) FROM events WHERE stream_id = ?"
        )
        .bind(stream_id)
        .fetch_optional(&mut *tx)
        .await
        .map_err(|e| EventStoreError::Storage(e.to_string()))?;

        let current_version = row
            .and_then(|r| r.0.into())
            .map(|v: i64| v as u64)
            .unwrap_or(0);

        if current_version != expected_version {
            return Err(EventStoreError::ConcurrencyConflict {
                expected: expected_version,
                actual: current_version,
            });
        }

        let mut new_version = current_version;
        let now = Utc::now().to_rfc3339();

        for event in &events {
            new_version += 1;
            sqlx::query(
                "INSERT INTO events (stream_id, version, event_type, payload, metadata, stored_at)
                 VALUES (?, ?, ?, ?, ?, ?)"
            )
            .bind(stream_id)
            .bind(new_version as i64)
            .bind(&event.event_type)
            .bind(serde_json::to_string(&event.payload).unwrap())
            .bind(serde_json::to_string(&event.metadata).unwrap())
            .bind(&now)
            .execute(&mut *tx)
            .await
            .map_err(|e| EventStoreError::Storage(e.to_string()))?;
        }

        tx.commit().await
            .map_err(|e| EventStoreError::Storage(e.to_string()))?;

        Ok(new_version)
    }

    async fn load_stream(
        &self,
        stream_id: &str,
    ) -> Result<Vec<StoredEvent>, EventStoreError> {
        self.load_stream_from(stream_id, 0).await
    }

    async fn load_stream_from(
        &self,
        stream_id: &str,
        from_version: u64,
    ) -> Result<Vec<StoredEvent>, EventStoreError> {
        let rows = sqlx::query_as::<_, (String, i64, String, String, String, String)>(
            "SELECT stream_id, version, event_type, payload, metadata, stored_at
             FROM events
             WHERE stream_id = ? AND version > ?
             ORDER BY version ASC"
        )
        .bind(stream_id)
        .bind(from_version as i64)
        .fetch_all(&self.pool)
        .await
        .map_err(|e| EventStoreError::Storage(e.to_string()))?;

        Ok(rows
            .into_iter()
            .map(|(stream_id, version, event_type, payload, metadata, stored_at)| {
                StoredEvent {
                    stream_id,
                    version: version as u64,
                    event_type,
                    payload: serde_json::from_str(&payload).unwrap_or_default(),
                    metadata: serde_json::from_str(&metadata).unwrap_or_default(),
                    stored_at: DateTime::parse_from_rfc3339(&stored_at)
                        .map(|dt| dt.with_timezone(&Utc))
                        .unwrap_or_else(|_| Utc::now()),
                }
            })
            .collect())
    }
}
```

The `UNIQUE(stream_id, version)` constraint is a second line of defense - even if the application-level version check somehow misses a conflict, the database will reject a duplicate version within the same stream.

## Tying it together

Here's how commands flow through the system end to end:

```rust
pub struct BankAccountService {
    store: Arc<dyn EventStore>,
}

impl BankAccountService {
    pub async fn execute(
        &self,
        stream_id: &str,
        cmd: BankAccountCommand,
        metadata: EventMetadata,
    ) -> Result<(), BankAccountError> {
        // 1. Load event history
        let stored = self.store.load_stream(stream_id).await
            .map_err(|e| BankAccountError::Store(e.to_string()))?;

        // 2. Rebuild aggregate state
        let mut account = BankAccount::default();
        for stored_event in &stored {
            let event: BankAccountEvent =
                serde_json::from_value(stored_event.payload.clone())
                    .map_err(|e| BankAccountError::Store(e.to_string()))?;
            account.apply(&event);
        }

        // 3. Validate command, produce new events
        let new_events = account.handle(cmd)?;

        // 4. Persist new events with optimistic concurrency
        let to_store: Vec<NewEvent> = new_events
            .iter()
            .map(|e| NewEvent {
                event_type: std::any::type_name_of_val(e).to_string(),
                payload: serde_json::to_value(e).unwrap(),
                metadata: metadata.clone(),
            })
            .collect();

        self.store.append(stream_id, account.version, to_store).await
            .map_err(|e| BankAccountError::Store(e.to_string()))?;

        Ok(())
    }
}
```

Load, replay, validate, append. Every command goes through this cycle. The aggregate is never stored - it's always rebuilt from events. This is the fundamental difference from traditional CRUD: the events table only grows, it never gets UPDATE or DELETE statements.

## Snapshots: when replay gets slow

The obvious problem with replaying every event from the beginning: it gets slow. An account with 10 events replays instantly. An account with 10,000 events takes noticeable time. A high-frequency trading account with millions of events is unusable without optimization.

Snapshots solve this by periodically saving the aggregate's state at a known version. Instead of replaying from event 1, you load the snapshot and replay only events after the snapshot version.

```rust
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Snapshot<T: Serialize> {
    pub stream_id: String,
    pub version: u64,
    pub state: T,
    pub taken_at: DateTime<Utc>,
}

#[async_trait]
pub trait SnapshotStore: Send + Sync {
    async fn load_latest<T: for<'de> Deserialize<'de> + Send>(
        &self,
        stream_id: &str,
    ) -> Result<Option<Snapshot<T>>, EventStoreError>;

    async fn save<T: Serialize + Send + Sync>(
        &self,
        snapshot: &Snapshot<T>,
    ) -> Result<(), EventStoreError>;
}
```

The service now loads the snapshot first, then only replays events that happened after it:

```rust
impl BankAccountService {
    pub async fn execute_with_snapshots(
        &self,
        stream_id: &str,
        cmd: BankAccountCommand,
        metadata: EventMetadata,
    ) -> Result<(), BankAccountError> {
        // 1. Try loading the latest snapshot
        let snapshot: Option<Snapshot<BankAccount>> = self.snapshots
            .load_latest(stream_id).await
            .map_err(|e| BankAccountError::Store(e.to_string()))?;

        // 2. Rebuild from snapshot + remaining events
        let (mut account, from_version) = match snapshot {
            Some(snap) => (snap.state, snap.version),
            None => (BankAccount::default(), 0),
        };

        let remaining = self.store
            .load_stream_from(stream_id, from_version).await
            .map_err(|e| BankAccountError::Store(e.to_string()))?;

        for stored_event in &remaining {
            let event: BankAccountEvent =
                serde_json::from_value(stored_event.payload.clone())
                    .map_err(|e| BankAccountError::Store(e.to_string()))?;
            account.apply(&event);
        }

        // 3. Validate and append (same as before)
        let new_events = account.handle(cmd)?;
        let to_store: Vec<NewEvent> = new_events
            .iter()
            .map(|e| NewEvent {
                event_type: std::any::type_name_of_val(e).to_string(),
                payload: serde_json::to_value(e).unwrap(),
                metadata: metadata.clone(),
            })
            .collect();

        let new_version = self.store
            .append(stream_id, account.version, to_store).await
            .map_err(|e| BankAccountError::Store(e.to_string()))?;

        // 4. Take snapshot every N events
        let snapshot_interval = 100;
        if new_version % snapshot_interval == 0 {
            // Apply the new events to our local state for the snapshot
            for event in &new_events {
                account.apply(event);
            }
            let snapshot = Snapshot {
                stream_id: stream_id.to_string(),
                version: new_version,
                state: account,
                taken_at: Utc::now(),
            };
            // Snapshot save failure is non-fatal - we can always rebuild from events
            if let Err(e) = self.snapshots.save(&snapshot).await {
                tracing::warn!("failed to save snapshot: {e}");
            }
        }

        Ok(())
    }
}
```

A few things to note. Snapshot saving is non-fatal. If it fails, you lose some performance on the next load but no data. The events are still the source of truth. You can delete all snapshots and rebuild them at any time. This is different from cache invalidation - there's no correctness risk, only a performance penalty.

The snapshot interval depends on your domain. For a bank account that gets a few events per day, you might snapshot every 50 events. For a real-time game entity getting hundreds of events per second, you might snapshot every 1000. Profile your replay time and pick a number that keeps it under your latency budget.

## CQRS: the natural companion

Event sourcing produces a write-optimized store - an append-only log. Reading from it means loading events and replaying them, which is terrible for queries like "show me all accounts with balance over $1000" or "total deposits this month." You'd have to replay every account's entire history.

This is where [CQRS](/blog/cqrs-in-practice-separating-reads-from-writes/) becomes essential rather than optional. The event store handles writes. A separate read model handles queries. Events flow from the write side to the read side through projections.

```rust
#[async_trait]
pub trait Projection: Send + Sync {
    async fn handle(&self, event: &StoredEvent) -> Result<(), ProjectionError>;
}

/// Maintains a read-optimized table of account balances.
pub struct AccountBalanceProjection {
    pool: SqlitePool,
}

#[async_trait]
impl Projection for AccountBalanceProjection {
    async fn handle(&self, event: &StoredEvent) -> Result<(), ProjectionError> {
        match event.event_type.as_str() {
            "Opened" => {
                let e: BankAccountEvent =
                    serde_json::from_value(event.payload.clone())?;
                if let BankAccountEvent::Opened { account_id, owner, .. } = e {
                    sqlx::query(
                        "INSERT INTO account_balances (id, owner, balance_cents, is_frozen)
                         VALUES (?, ?, 0, false)"
                    )
                    .bind(&account_id)
                    .bind(&owner)
                    .execute(&self.pool)
                    .await?;
                }
            }
            "Deposited" => {
                let e: BankAccountEvent =
                    serde_json::from_value(event.payload.clone())?;
                if let BankAccountEvent::Deposited { amount_cents, .. } = e {
                    sqlx::query(
                        "UPDATE account_balances
                         SET balance_cents = balance_cents + ?
                         WHERE id = ?"
                    )
                    .bind(amount_cents)
                    .bind(&event.stream_id)
                    .execute(&self.pool)
                    .await?;
                }
            }
            "Withdrawn" => {
                let e: BankAccountEvent =
                    serde_json::from_value(event.payload.clone())?;
                if let BankAccountEvent::Withdrawn { amount_cents, .. } = e {
                    sqlx::query(
                        "UPDATE account_balances
                         SET balance_cents = balance_cents - ?
                         WHERE id = ?"
                    )
                    .bind(amount_cents)
                    .bind(&event.stream_id)
                    .execute(&self.pool)
                    .await?;
                }
            }
            _ => {}
        }
        Ok(())
    }
}
```

The read model is a regular table that you can query with normal SQL: `SELECT * FROM account_balances WHERE balance_cents > 100000`. It's updated by processing events as they're appended. If the projection has a bug, you drop the table, fix the code, and replay all events to rebuild it. The events are immutable - the source of truth is always intact.

You can have multiple projections: one for balances, one for transaction history, one for monthly summaries, one for fraud detection. Each processes the same events and builds a different view optimized for different queries. This is Level 3 CQRS from the [CQRS post](/blog/cqrs-in-practice-separating-reads-from-writes/) - separate write store (event log) and read stores (projections), synchronized by events.

## Time travel and debugging

Because events are immutable and ordered, you can reconstruct the state of any aggregate at any point in time:

```rust
impl BankAccount {
    /// Rebuild state up to (and including) a specific version.
    pub fn at_version(events: &[BankAccountEvent], target_version: u64) -> Self {
        let mut account = Self::default();
        for (i, event) in events.iter().enumerate() {
            if (i as u64 + 1) > target_version {
                break;
            }
            account.apply(event);
        }
        account
    }
}

// What was the balance right before that disputed withdrawal?
let all_events = store.load_stream("acc-123").await?;
let events: Vec<BankAccountEvent> = all_events
    .iter()
    .map(|se| serde_json::from_value(se.payload.clone()).unwrap())
    .collect();
let state_before = BankAccount::at_version(&events, disputed_event_version - 1);
println!("balance before withdrawal: {}", state_before.balance_cents);
```

This is genuinely useful in practice. When a customer disputes a transaction, you can show the exact state of their account at every point. When a bug corrupts state, you can step through events to find exactly where things went wrong. When regulations require an audit trail, the event store *is* the audit trail - every state change is recorded with a timestamp, correlation ID, and the user who triggered it.

## The Rust ecosystem

If you're building event sourcing in Rust and don't want to implement everything from scratch, a few crates exist:

[`cqrs-es`](https://crates.io/crates/cqrs-es) is the most mature option. It provides an `Aggregate` trait, event store implementations for Postgres, MySQL, and DynamoDB, and a `CqrsFramework` that wires everything together. The aggregate trait looks similar to what we built: you define events, a command handler that returns events, and an apply method that updates state.

[`eventually-rs`](https://github.com/get-eventually/eventually-rs) takes a different approach with an `EventStore` trait and an `AggregateRootRepository`. It has Postgres support and is under active development (v0.5.0 is in progress with breaking changes expected).

[`eventstore`](https://crates.io/crates/eventstore) is the official Rust client for [EventStoreDB](https://www.eventstore.com/) (now Kurrent) - a purpose-built database for event sourcing. If you need persistent subscriptions, projections in the database, and a battle-tested event store, this is the production-grade option.

For most projects starting out, the hand-rolled approach we built above is fine. It's straightforward, you understand every line, and you can swap in a library later when you need features like persistent projections or multi-node replication. The aggregate pattern and the event store trait don't change.

## When event sourcing is worth it

**Finance and payments.** Any system where you need to explain how you arrived at a number. "Why is this balance $47.23?" becomes a trivial query against the event log instead of a forensic investigation across multiple tables.

**Audit-heavy domains.** Healthcare, legal, compliance. Regulators want to know who changed what, when, and what the state was before and after. Events give you that for free.

**Complex domain logic with temporal dependencies.** Insurance claims that go through multiple stages, orders that can be partially fulfilled, subscriptions with upgrades, downgrades, pauses, and prorations. When the state machine is complex and the transitions need to be traceable, events make it explicit.

**Systems that need replay.** If you might need to reprocess historical data - fix a calculation bug, build a new report, migrate to a new schema - event sourcing lets you replay history through corrected logic. With traditional state storage, you'd need a backup from before the bug was introduced (if you have one).

## When event sourcing is overkill

**CRUD applications.** A blog, a todo app, a settings page. If your entities are created, maybe updated a few times, and the history doesn't matter, event sourcing adds complexity for no benefit. Just UPDATE the row.

**Systems where the current state is all that matters.** A cache, a feature flag store, a user preferences table. Nobody will ever ask "what was this feature flag set to three weeks ago?" If they do, a `last_modified_at` column is enough.

**Small teams without event sourcing experience.** The pattern has a learning curve. Eventual consistency between the event store and read models is a source of bugs that are hard to reproduce and harder to debug. If your team is new to this, the operational complexity can eat the productivity gains. Start with CQRS (separate read/write models, same database) and add event sourcing only when you need the audit trail or replay capability.

**High-frequency updates to the same entity.** If an entity gets thousands of updates per second, even with snapshots, the append-only log grows fast and concurrency conflicts become frequent. Real-time game state, high-frequency sensor data, live location tracking - these are better served by state-based storage with change data capture if you need history.

## Storage and operational concerns

Event stores grow indefinitely. Unlike traditional databases where an UPDATE reuses space, every state change adds a new row. For a bank account with 1000 transactions per year, that's 1000 rows per year per account, forever. Multiply by millions of accounts and you're looking at serious storage.

Strategies for managing growth:

**Archival.** Move events older than N years to cold storage (S3, archive tier). Keep snapshots in the hot store so you don't need the old events for normal operations.

**Compaction.** For some domains, you can replace a long event stream with a single "state established" event. Account had 50,000 events? Replace them with `BalanceEstablished { balance_cents: 47230, as_of_version: 50000 }` and delete the originals. You lose history but gain space. Only do this when regulations allow it.

**Partitioning.** Shard event streams by aggregate ID or date range. SQLite per aggregate, Postgres partitions by month, or a purpose-built event store like EventStoreDB that handles this natively.

Event versioning is the other operational headache. When you rename a field in an event, add a new field, or change the structure, old events in the store still have the old shape. You need a strategy:

```rust
/// Upcasting: transform old event formats to new ones during replay.
fn upcast(event_type: &str, version: u32, payload: serde_json::Value) -> serde_json::Value {
    match (event_type, version) {
        ("Deposited", 1) => {
            // v1 had "amount" in dollars, v2 uses "amount_cents"
            let mut p = payload;
            if let Some(amount) = p.get("amount").and_then(|v| v.as_f64()) {
                p["amount_cents"] = serde_json::json!((amount * 100.0) as i64);
                p.as_object_mut().unwrap().remove("amount");
            }
            p
        }
        _ => payload,
    }
}
```

Upcasting transforms old events into the current schema during replay. The stored events are never modified - you keep the original bytes and transform on read. This is similar to database migrations but applied at the application level rather than the storage level.

## The decision framework

Before reaching for event sourcing, ask:

1. Do you need an audit trail of every state change? Not "it would be nice" but "regulators require it" or "customers dispute transactions."
2. Do you need to rebuild state from history? Fix calculation bugs retroactively, build new reports from historical data, replay events through updated business logic.
3. Is your domain inherently event-driven? Orders, payments, claims, bookings - things that go through stages where each transition is meaningful.
4. Can your team handle eventual consistency? The read model will lag behind the write model. Users might see stale data for a few milliseconds (or longer under load). Is that acceptable?

If the answer to questions 1-3 is "not really" and your team hasn't built event-sourced systems before, CQRS with a traditional database gives you most of the architectural benefits without the operational complexity. You can always add event sourcing to specific aggregates later - it doesn't have to be all or nothing.

Event sourcing is a powerful pattern when the domain demands it. The append-only event log, the aggregate that rebuilds from history, the projections that derive read models - they're elegant solutions to real problems in finance, healthcare, logistics, and any domain where "what happened" matters as much as "what is." But they come with real costs: storage growth, event versioning, eventual consistency, and a steeper learning curve. Use it where it earns its keep. Use CRUD everywhere else.
