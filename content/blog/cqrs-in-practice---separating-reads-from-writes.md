+++
title = "CQRS in practice - separating reads from writes"
date = 2025-03-20
description = "Command Query Responsibility Segregation without the event sourcing baggage - when it helps, when it's overkill, and how to implement it in Rust with separate read and write models."

[taxonomies]
tags = ["rust", "architecture", "design-patterns", "databases"]
+++

You have an order management system. The write path is straightforward: validate the order, check inventory, persist it. The read path is a nightmare. The dashboard needs order counts grouped by status. The customer portal needs order history with product names and thumbnails. The admin panel needs a searchable list with filters on date range, amount, customer, and fulfillment state. The export endpoint needs a flat CSV row per order line item.

All of these reads hit the same `orders` and `order_items` tables, but they need wildly different shapes. So you end up with one `Order` struct that tries to serve all of them. It has 30 fields, half of which are `Option` because they're only populated for certain views. Your `find_all` method takes 8 parameters. Your SQL queries join 4 tables and the WHERE clause is built dynamically based on which fields are `Some`.

This is the problem CQRS solves.

<!-- more -->

## What CQRS actually is

Command Query Responsibility Segregation. The name sounds academic but the idea is simple: use different types for reading data than for writing data. Commands change state. Queries return state. They go through different code paths, use different structs, and can be optimized independently.

That's it. Not "use Kafka." Not "build an event store." Not "deploy two databases." Just: the struct you use to create an order is not the same struct you use to display an order on a dashboard.

Bertrand Meyer introduced the underlying principle as CQS (Command Query Separation) back in the 1980s - every method should either change state or return data, never both. Greg Young took that further with CQRS: separate the *models* themselves, not just the methods.

If you've read [the repository pattern post](/blog/the-repository-pattern-abstracting-data-access-in-rust/), you already have the building blocks. A `Repository<T>` trait with `create`, `update`, `delete` on the write side, and `find_by_id`, `find_all` on the read side. CQRS takes the next step: those two sides don't have to share the same `T`.

## What CQRS is not

The biggest misconception: CQRS requires event sourcing. It doesn't. Event sourcing stores state as a sequence of events and replays them to reconstruct current state. CQRS just means separate models for reads and writes. You can combine them, but they're independent ideas. Most systems that benefit from CQRS don't need event sourcing at all.

Second misconception: CQRS means two databases. It can, but the simplest form uses one database with different structs and different queries for the read path vs the write path. No synchronization, no eventual consistency, no message queues. Same Postgres instance, different code paths.

Third misconception: CQRS is an architecture. It's a pattern you apply to specific parts of your system. Your user registration flow is plain CRUD? Keep it that way. Your analytics dashboard reads denormalized data from 5 tables? That's where CQRS helps. You don't need to go all-in.

## The spectrum of CQRS

There's no single "CQRS implementation." It's a spectrum from lightweight type separation to fully distributed systems:

**Level 1 - Different types, same database, same code path.** Your `CreateOrder` command struct has 6 fields. Your `OrderSummary` query result has 4 different fields (including computed ones like `total_amount`). Both hit the same `orders` table. This is where most projects should start.

**Level 2 - Different types, same database, different code paths.** Commands go through a `WriteRepository` that does validation and business logic. Queries go through a `ReadRepository` that runs optimized SQL with JOINs and aggregations. Still one database, but the read queries bypass the domain model entirely - they just project data into view-specific structs.

**Level 3 - Different databases.** Writes go to a normalized relational database. Reads come from a denormalized read store - maybe the same Postgres with materialized views, maybe Redis, maybe Elasticsearch. Now you need a synchronization mechanism: database triggers, change data capture, or domain events that update the read store after each write.

Level 1 gets you 80% of the benefit with almost zero cost. Level 3 is for systems where reads and writes have genuinely different scaling requirements - thousands of complex queries per second but only a handful of writes. Most web applications live at Level 1 or Level 2 and never need to go further.

## The write side: commands and handlers

The write model cares about correctness. Validation, business rules, invariants. Here's what a command-oriented write path looks like in Rust:

```rust
/// Commands - what the caller wants to happen.
/// These are intentional: "place this order," not "insert a row."
pub struct PlaceOrder {
    pub customer_id: String,
    pub items: Vec<OrderLineItem>,
    pub shipping_address: String,
}

pub struct OrderLineItem {
    pub product_id: String,
    pub quantity: i32,
    pub unit_price_cents: i64,
}

pub struct CancelOrder {
    pub order_id: String,
    pub reason: String,
}

pub struct MarkShipped {
    pub order_id: String,
    pub tracking_number: String,
}
```

Notice these aren't generic `UpdateOrder { field1: Option<...>, field2: Option<...> }` partial-update structs. Each command represents a specific business action with exactly the data that action requires. `CancelOrder` needs a reason. `MarkShipped` needs a tracking number. There's no way to accidentally set a tracking number when canceling.

The write repository only has methods that change state:

```rust
use async_trait::async_trait;

/// The entity as stored - all fields, normalized, source of truth.
#[derive(Debug, Clone)]
pub struct Order {
    pub id: String,
    pub customer_id: String,
    pub status: OrderStatus,
    pub shipping_address: String,
    pub tracking_number: Option<String>,
    pub cancel_reason: Option<String>,
    pub created_at: String,
    pub updated_at: String,
}

#[derive(Debug, Clone, PartialEq)]
pub enum OrderStatus {
    Pending,
    Confirmed,
    Shipped,
    Delivered,
    Cancelled,
}

#[async_trait]
pub trait OrderWriteRepo: Send + Sync {
    async fn insert(&self, order: &Order, items: &[OrderItem]) -> Result<(), RepoError>;
    async fn update_status(
        &self,
        id: &str,
        status: OrderStatus,
        updated_at: &str,
    ) -> Result<(), RepoError>;
    async fn set_tracking(
        &self,
        id: &str,
        tracking: &str,
        updated_at: &str,
    ) -> Result<(), RepoError>;
    async fn set_cancelled(
        &self,
        id: &str,
        reason: &str,
        updated_at: &str,
    ) -> Result<(), RepoError>;
}
```

No `find_by_id` here. No `find_all`. The write repo doesn't serve queries. Each method maps to a specific state transition. `set_tracking` writes exactly two fields: `tracking_number` and `updated_at`. It doesn't load the entire entity, modify it in memory, and write it back. It runs a targeted UPDATE.

The command handler contains the business logic:

```rust
pub struct OrderCommandHandler {
    write_repo: Arc<dyn OrderWriteRepo>,
    product_repo: Arc<dyn ProductReadRepo>,
}

impl OrderCommandHandler {
    pub async fn handle_place_order(
        &self,
        cmd: PlaceOrder,
    ) -> Result<String, AppError> {
        // Validate: at least one item
        if cmd.items.is_empty() {
            return Err(AppError::Validation("order must have at least one item".into()));
        }

        // Validate: all products exist and prices match
        for item in &cmd.items {
            let product = self.product_repo
                .get_product(&item.product_id)
                .await?
                .ok_or_else(|| AppError::NotFound(
                    format!("product {}", item.product_id)
                ))?;

            if product.price_cents != item.unit_price_cents {
                return Err(AppError::Validation(format!(
                    "price mismatch for {}: expected {}, got {}",
                    item.product_id, product.price_cents, item.unit_price_cents
                )));
            }
        }

        let now = chrono::Utc::now().to_rfc3339();
        let order_id = uuid::Uuid::new_v4().to_string();

        let order = Order {
            id: order_id.clone(),
            customer_id: cmd.customer_id,
            status: OrderStatus::Pending,
            shipping_address: cmd.shipping_address,
            tracking_number: None,
            cancel_reason: None,
            created_at: now.clone(),
            updated_at: now,
        };

        let items: Vec<OrderItem> = cmd.items
            .into_iter()
            .map(|li| OrderItem {
                id: uuid::Uuid::new_v4().to_string(),
                order_id: order_id.clone(),
                product_id: li.product_id,
                quantity: li.quantity,
                unit_price_cents: li.unit_price_cents,
            })
            .collect();

        self.write_repo.insert(&order, &items).await?;
        Ok(order_id)
    }

    pub async fn handle_mark_shipped(
        &self,
        cmd: MarkShipped,
    ) -> Result<(), AppError> {
        let now = chrono::Utc::now().to_rfc3339();
        self.write_repo.set_tracking(&cmd.order_id, &cmd.tracking_number, &now).await?;
        self.write_repo.update_status(&cmd.order_id, OrderStatus::Shipped, &now).await?;
        Ok(())
    }

    pub async fn handle_cancel(
        &self,
        cmd: CancelOrder,
    ) -> Result<(), AppError> {
        let now = chrono::Utc::now().to_rfc3339();
        self.write_repo.set_cancelled(&cmd.order_id, &cmd.reason, &now).await?;
        self.write_repo.update_status(&cmd.order_id, OrderStatus::Cancelled, &now).await?;
        Ok(())
    }
}
```

The handler orchestrates the business rules. The repo executes the storage. Each command handler method is small and focused. `handle_place_order` validates, then inserts. `handle_cancel` sets the reason, then updates the status. There's no temptation to add query logic here because the write repo doesn't support queries.

## The read side: queries and view models

The read model cares about speed and shape. No business logic, no validation - just projecting data into the exact shape the consumer needs. This is where CQRS pays off: each query returns a purpose-built struct.

```rust
/// What the dashboard needs: counts and totals, no line items.
#[derive(Debug, Clone, serde::Serialize)]
pub struct OrderSummary {
    pub id: String,
    pub customer_name: String,
    pub status: String,
    pub item_count: i32,
    pub total_cents: i64,
    pub created_at: String,
}

/// What the customer portal needs: full detail with product names.
#[derive(Debug, Clone, serde::Serialize)]
pub struct OrderDetail {
    pub id: String,
    pub status: String,
    pub shipping_address: String,
    pub tracking_number: Option<String>,
    pub items: Vec<OrderDetailItem>,
    pub total_cents: i64,
    pub created_at: String,
}

#[derive(Debug, Clone, serde::Serialize)]
pub struct OrderDetailItem {
    pub product_name: String,
    pub quantity: i32,
    pub unit_price_cents: i64,
    pub line_total_cents: i64,
}

/// What the CSV export needs: one flat row per line item.
#[derive(Debug, Clone, serde::Serialize)]
pub struct OrderExportRow {
    pub order_id: String,
    pub customer_email: String,
    pub product_name: String,
    pub quantity: i32,
    pub unit_price_cents: i64,
    pub order_status: String,
    pub order_date: String,
}

/// What the analytics endpoint needs.
#[derive(Debug, Clone, serde::Serialize)]
pub struct OrderStats {
    pub total_orders: i64,
    pub total_revenue_cents: i64,
    pub orders_by_status: Vec<StatusCount>,
    pub avg_order_value_cents: i64,
}

#[derive(Debug, Clone, serde::Serialize)]
pub struct StatusCount {
    pub status: String,
    pub count: i64,
}
```

Five different shapes, all from the same underlying data. In a traditional single-model approach, you'd either cram all of these into one bloated `Order` struct or write ad-hoc SQL in each handler. CQRS makes each view model a first-class type with its own read path.

The read repository has purpose-built query methods:

```rust
#[async_trait]
pub trait OrderReadRepo: Send + Sync {
    /// Dashboard: paginated summaries with optional status filter.
    async fn list_summaries(
        &self,
        opts: &QueryOpts,
        status_filter: Option<&str>,
    ) -> Result<Vec<OrderSummary>, RepoError>;

    /// Customer portal: full order with line items and product names.
    async fn get_detail(&self, order_id: &str) -> Result<Option<OrderDetail>, RepoError>;

    /// CSV export: flat rows, one per line item.
    async fn export_rows(
        &self,
        date_from: &str,
        date_to: &str,
    ) -> Result<Vec<OrderExportRow>, RepoError>;

    /// Analytics: aggregated stats.
    async fn get_stats(&self) -> Result<OrderStats, RepoError>;
}
```

Each method returns exactly what the consumer needs. `list_summaries` does a COUNT and SUM in SQL so you never load individual items just to count them. `get_detail` does a JOIN to get product names. `export_rows` does a different JOIN to flatten everything into rows. The SQL for each is optimized for that specific shape - no compromise needed.

Here's what the SQL implementation of `list_summaries` looks like:

```rust
use sqlx::{SqlitePool, Row};

pub struct SqliteOrderReadRepo {
    pool: SqlitePool,
}

#[async_trait]
impl OrderReadRepo for SqliteOrderReadRepo {
    async fn list_summaries(
        &self,
        opts: &QueryOpts,
        status_filter: Option<&str>,
    ) -> Result<Vec<OrderSummary>, RepoError> {
        let limit = opts.limit.unwrap_or(50);
        let offset = opts.offset.unwrap_or(0);

        let (where_clause, bind_status) = match status_filter {
            Some(s) => ("WHERE o.status = ?", Some(s)),
            None => ("", None),
        };

        let sql = format!(
            "SELECT o.id, c.name AS customer_name, o.status,
                    COUNT(oi.id) AS item_count,
                    COALESCE(SUM(oi.quantity * oi.unit_price_cents), 0) AS total_cents,
                    o.created_at
             FROM orders o
             JOIN customers c ON c.id = o.customer_id
             LEFT JOIN order_items oi ON oi.order_id = o.id
             {}
             GROUP BY o.id
             ORDER BY o.created_at DESC
             LIMIT ? OFFSET ?",
            where_clause
        );

        let mut query = sqlx::query(&sql);
        if let Some(status) = bind_status {
            query = query.bind(status);
        }
        query = query.bind(limit).bind(offset);

        let rows = query
            .fetch_all(&self.pool)
            .await
            .map_err(|e| RepoError::Storage(e.to_string()))?;

        Ok(rows
            .iter()
            .map(|r| OrderSummary {
                id: r.get("id"),
                customer_name: r.get("customer_name"),
                status: r.get("status"),
                item_count: r.get("item_count"),
                total_cents: r.get("total_cents"),
                created_at: r.get("created_at"),
            })
            .collect())
    }

    async fn get_detail(&self, order_id: &str) -> Result<Option<OrderDetail>, RepoError> {
        // First, fetch the order header
        let order_row = sqlx::query(
            "SELECT id, status, shipping_address, tracking_number, created_at
             FROM orders WHERE id = ?"
        )
        .bind(order_id)
        .fetch_optional(&self.pool)
        .await
        .map_err(|e| RepoError::Storage(e.to_string()))?;

        let order_row = match order_row {
            Some(r) => r,
            None => return Ok(None),
        };

        // Then, fetch line items with product names in one query
        let item_rows = sqlx::query(
            "SELECT p.name AS product_name, oi.quantity,
                    oi.unit_price_cents,
                    (oi.quantity * oi.unit_price_cents) AS line_total_cents
             FROM order_items oi
             JOIN products p ON p.id = oi.product_id
             WHERE oi.order_id = ?
             ORDER BY p.name"
        )
        .bind(order_id)
        .fetch_all(&self.pool)
        .await
        .map_err(|e| RepoError::Storage(e.to_string()))?;

        let items: Vec<OrderDetailItem> = item_rows
            .iter()
            .map(|r| OrderDetailItem {
                product_name: r.get("product_name"),
                quantity: r.get("quantity"),
                unit_price_cents: r.get("unit_price_cents"),
                line_total_cents: r.get("line_total_cents"),
            })
            .collect();

        let total_cents: i64 = items.iter().map(|i| i.line_total_cents).sum();

        Ok(Some(OrderDetail {
            id: order_row.get("id"),
            status: order_row.get("status"),
            shipping_address: order_row.get("shipping_address"),
            tracking_number: order_row.get("tracking_number"),
            items,
            total_cents,
            created_at: order_row.get("created_at"),
        }))
    }

    async fn export_rows(
        &self,
        date_from: &str,
        date_to: &str,
    ) -> Result<Vec<OrderExportRow>, RepoError> {
        let rows = sqlx::query(
            "SELECT o.id AS order_id, c.email AS customer_email,
                    p.name AS product_name, oi.quantity,
                    oi.unit_price_cents, o.status AS order_status,
                    o.created_at AS order_date
             FROM orders o
             JOIN customers c ON c.id = o.customer_id
             JOIN order_items oi ON oi.order_id = o.id
             JOIN products p ON p.id = oi.product_id
             WHERE o.created_at >= ? AND o.created_at < ?
             ORDER BY o.created_at, o.id, p.name"
        )
        .bind(date_from)
        .bind(date_to)
        .fetch_all(&self.pool)
        .await
        .map_err(|e| RepoError::Storage(e.to_string()))?;

        Ok(rows
            .iter()
            .map(|r| OrderExportRow {
                order_id: r.get("order_id"),
                customer_email: r.get("customer_email"),
                product_name: r.get("product_name"),
                quantity: r.get("quantity"),
                unit_price_cents: r.get("unit_price_cents"),
                order_status: r.get("order_status"),
                order_date: r.get("order_date"),
            })
            .collect())
    }

    async fn get_stats(&self) -> Result<OrderStats, RepoError> {
        let row = sqlx::query(
            "SELECT COUNT(*) AS total_orders,
                    COALESCE(SUM(oi_totals.order_total), 0) AS total_revenue_cents,
                    COALESCE(AVG(oi_totals.order_total), 0) AS avg_order_value_cents
             FROM orders o
             LEFT JOIN (
                 SELECT order_id, SUM(quantity * unit_price_cents) AS order_total
                 FROM order_items
                 GROUP BY order_id
             ) oi_totals ON oi_totals.order_id = o.id"
        )
        .fetch_one(&self.pool)
        .await
        .map_err(|e| RepoError::Storage(e.to_string()))?;

        let status_rows = sqlx::query(
            "SELECT status, COUNT(*) AS count FROM orders GROUP BY status ORDER BY count DESC"
        )
        .fetch_all(&self.pool)
        .await
        .map_err(|e| RepoError::Storage(e.to_string()))?;

        Ok(OrderStats {
            total_orders: row.get("total_orders"),
            total_revenue_cents: row.get("total_revenue_cents"),
            avg_order_value_cents: row.get("avg_order_value_cents"),
            orders_by_status: status_rows
                .iter()
                .map(|r| StatusCount {
                    status: r.get("status"),
                    count: r.get("count"),
                })
                .collect(),
        })
    }
}
```

Compare this to a unified `Repository<Order>` where `find_all` returns `Vec<Order>` and every consumer has to do its own transformation. The read repo runs the exact SQL each consumer needs. The dashboard gets aggregated data in one query - it never loads thousands of order items just to count them. The export gets flat rows without the application having to flatten nested structures in memory.

## Wiring it up

The API layer routes commands and queries to their respective handlers:

```rust
use std::sync::Arc;
use axum::{Router, Json, extract::{State, Path, Query}};

pub struct AppState {
    pub commands: OrderCommandHandler,
    pub queries: Arc<dyn OrderReadRepo>,
}

async fn place_order(
    State(state): State<Arc<AppState>>,
    Json(cmd): Json<PlaceOrder>,
) -> Result<Json<serde_json::Value>, AppError> {
    let id = state.commands.handle_place_order(cmd).await?;
    Ok(Json(serde_json::json!({ "order_id": id })))
}

async fn cancel_order(
    State(state): State<Arc<AppState>>,
    Json(cmd): Json<CancelOrder>,
) -> Result<Json<serde_json::Value>, AppError> {
    state.commands.handle_cancel(cmd).await?;
    Ok(Json(serde_json::json!({ "status": "cancelled" })))
}

async fn list_orders(
    State(state): State<Arc<AppState>>,
    Query(opts): Query<QueryOpts>,
    Query(filter): Query<StatusFilter>,
) -> Result<Json<Vec<OrderSummary>>, AppError> {
    let summaries = state.queries
        .list_summaries(&opts, filter.status.as_deref())
        .await?;
    Ok(Json(summaries))
}

async fn get_order(
    State(state): State<Arc<AppState>>,
    Path(id): Path<String>,
) -> Result<Json<OrderDetail>, AppError> {
    let detail = state.queries.get_detail(&id).await?
        .ok_or(AppError::NotFound(format!("order {}", id)))?;
    Ok(Json(detail))
}

async fn order_stats(
    State(state): State<Arc<AppState>>,
) -> Result<Json<OrderStats>, AppError> {
    let stats = state.queries.get_stats().await?;
    Ok(Json(stats))
}

fn router() -> Router<Arc<AppState>> {
    Router::new()
        // Commands (writes)
        .route("/orders", axum::routing::post(place_order))
        .route("/orders/:id/cancel", axum::routing::post(cancel_order))
        // Queries (reads)
        .route("/orders", axum::routing::get(list_orders))
        .route("/orders/:id", axum::routing::get(get_order))
        .route("/orders/stats", axum::routing::get(order_stats))
}
```

POST endpoints go through the command handler. GET endpoints go through the read repo. They share the same database but different code paths, different types, different optimization strategies. That's Level 2 CQRS - practical, testable, zero infrastructure overhead.

## Testing both sides independently

Because reads and writes are separate traits, you test them separately. The write side tests business rules without caring about query shapes. The read side tests data projection without caring about validation.

```rust
#[cfg(test)]
mod tests {
    use super::*;

    struct MockOrderWriteRepo {
        inserted: std::sync::Mutex<Vec<(Order, Vec<OrderItem>)>>,
    }

    impl MockOrderWriteRepo {
        fn new() -> Self {
            Self { inserted: std::sync::Mutex::new(vec![]) }
        }

        fn inserted_count(&self) -> usize {
            self.inserted.lock().unwrap().len()
        }
    }

    #[async_trait]
    impl OrderWriteRepo for MockOrderWriteRepo {
        async fn insert(&self, order: &Order, items: &[OrderItem]) -> Result<(), RepoError> {
            self.inserted.lock().unwrap().push((order.clone(), items.to_vec()));
            Ok(())
        }
        async fn update_status(&self, _: &str, _: OrderStatus, _: &str) -> Result<(), RepoError> {
            Ok(())
        }
        async fn set_tracking(&self, _: &str, _: &str, _: &str) -> Result<(), RepoError> {
            Ok(())
        }
        async fn set_cancelled(&self, _: &str, _: &str, _: &str) -> Result<(), RepoError> {
            Ok(())
        }
    }

    #[tokio::test]
    async fn place_order_rejects_empty_items() {
        let handler = OrderCommandHandler {
            write_repo: Arc::new(MockOrderWriteRepo::new()),
            product_repo: Arc::new(mock_product_repo()),
        };

        let cmd = PlaceOrder {
            customer_id: "cust-1".into(),
            items: vec![],
            shipping_address: "123 Main St".into(),
        };

        let result = handler.handle_place_order(cmd).await;
        assert!(result.is_err());
    }

    #[tokio::test]
    async fn place_order_validates_price_mismatch() {
        let handler = OrderCommandHandler {
            write_repo: Arc::new(MockOrderWriteRepo::new()),
            product_repo: Arc::new(mock_product_repo()), // returns price 999
        };

        let cmd = PlaceOrder {
            customer_id: "cust-1".into(),
            items: vec![OrderLineItem {
                product_id: "prod-1".into(),
                quantity: 1,
                unit_price_cents: 500, // wrong price
            }],
            shipping_address: "123 Main St".into(),
        };

        let result = handler.handle_place_order(cmd).await;
        assert!(matches!(result, Err(AppError::Validation(_))));
    }
}
```

Write-side tests use a mock write repo. They verify commands are validated, rejected when invalid, and persisted when valid. No database, no SQL, no query optimization concerns.

Read-side tests use a real SQLite database seeded with known data, verifying that queries return the right shape with the right aggregations. You could also use a mock read repo for handler tests, but for the read SQL itself, an integration test against SQLite is worth the small overhead - you're testing the query logic, not business rules.

## When CQRS helps

**Read models that differ from write models.** The example above: `PlaceOrder` has nested line items with product IDs. `OrderSummary` has a flat row with a customer name, item count, and total. These shapes are fundamentally different. Forcing them into one struct means either the write side carries display-only fields or the read side loads unnecessary data.

**Multiple read shapes for the same data.** Dashboard, customer portal, admin panel, export, analytics - each needs a different projection. Without CQRS, every new view means another set of parameters on `find_all` or a new ad-hoc query buried in a handler. With CQRS, each view gets its own method on the read repo and its own return type.

**Read-heavy systems.** If your system does 100 reads for every write, optimizing the read path independently is worth the effort. The read repo can use [database-level optimizations](https://www.sqlite.org/wal.html) - covering indexes, materialized views, denormalized tables, read replicas - without affecting the write model. If you read [the SQLite WAL post](/blog/why-sqlite-with-wal-mode-is-good-enough-for-most-web-apps/), you already know that WAL mode lets readers run concurrently with writers. CQRS at the application level complements that at the storage level.

**Complex domain logic on writes.** When your write path involves validation across multiple entities, inventory checks, pricing calculations, and state machine transitions, isolating that logic from query concerns keeps handlers clean. The write side is a command handler with focused business logic. The read side is a data projector with optimized SQL.

## When CQRS is overkill

**Simple CRUD.** If your read model and write model are the same shape - a blog post is a blog post, whether you're creating it or displaying it - CQRS adds types and traits for no benefit. A single `Repository<Post>` with `create`, `find_by_id`, and `find_all` is enough.

**Small data, few consumers.** If you have one list view and one detail view, and the data fits in a single SELECT with no JOINs, splitting into read and write repos doubles your code for zero gain.

**Prototype or MVP stage.** Get the product working first. When you start feeling the pain - too many `Option` fields, queries getting complex, multiple consumers fighting over the same struct - that's when you introduce CQRS. Not before.

The test is simple: are you writing `#[serde(skip_serializing_if = "Option::is_none")]` on half the fields of your domain entity? Are you passing 6 boolean flags to control which fields get populated? Are different API endpoints returning the same type but with different fields filled in? Those are signs you need separate read models.

## The pragmatic middle ground

Full CQRS with separate databases, event-driven synchronization, and eventual consistency is a distributed systems problem. It adds operational complexity that most teams don't need. The pragmatic approach:

**Start with separate types.** Use one `CreateOrder` struct for writes and one `OrderSummary` struct for reads. Same database, same codebase. This costs almost nothing and gives you the type-safety benefit immediately.

**Separate the traits when queries get complex.** When your read methods start needing JOINs, aggregations, and custom SQL that doesn't fit the generic `Repository<T>` shape, split into `OrderWriteRepo` and `OrderReadRepo`. This is Level 2 and where most projects should land.

**Go to separate stores only when measured.** If your read traffic is crushing the database and write latency is suffering, consider a read replica or a materialized view. But measure first. [SQLite in WAL mode handles thousands of mixed operations per second](/blog/why-sqlite-with-wal-mode-is-good-enough-for-most-web-apps/). Postgres with proper indexes handles orders of magnitude more. You might be surprised how far a single database goes.

**Keep the read repo as a trait.** Even if you only have one implementation today, the trait boundary lets you swap in a caching layer, a read replica, or an in-memory implementation for tests without changing any consumer code. This is the same principle from [the adapter pattern](/blog/the-adapter-pattern-in-rust-wrapping-external-apis/) - depend on the trait, not the implementation.

## CQRS and Rust's type system

Rust makes CQRS natural in ways that dynamic languages don't. In Python or JavaScript, your "separate models" are just dicts with different keys - there's nothing stopping you from accidentally reading write-model fields in a query handler or vice versa. In Rust, `OrderSummary` and `PlaceOrder` are different types. You can't pass one where the other is expected. The compiler enforces the separation.

The ownership model helps too. Your `OrderCommandHandler` owns an `Arc<dyn OrderWriteRepo>`. Your query handlers own an `Arc<dyn OrderReadRepo>`. There's no shared mutable state between them. The write repo can't accidentally be used for reads because the query handlers don't have access to it. The borrow checker enforces what most languages enforce through documentation and discipline.

This is also where Rust's zero-cost abstractions matter. Each view model struct is exactly as big as its fields - no hidden pointers, no runtime type information, no base class overhead. An `OrderSummary` with 6 fields takes exactly as much memory as those 6 fields. When you're projecting thousands of rows into view models, that efficiency adds up. No garbage collector pause in the middle of building a dashboard response.

## Recap

CQRS is not an architecture. It's a pattern you apply where read and write models diverge. In Rust, it's natural: define command structs and a write repo trait for mutations, define view-model structs and a read repo trait for queries. Start simple - same database, different types. Split the repos when complexity demands it. Go to separate stores only when you've measured the need.

The type system does the heavy lifting. Different types for different purposes, trait boundaries between components, ownership rules that prevent accidental cross-contamination. You get the benefits of CQRS with compile-time guarantees that the separation is maintained.

Don't add CQRS because someone wrote a blog post about it (even this one). Add it when your single model is struggling to serve multiple consumers with different needs. You'll know when that happens - the `Option` fields and the query parameters will tell you.
