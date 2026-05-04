+++
title = "The repository pattern - abstracting data access in Rust"
date = 2026-03-20
description = "How to build a repository trait in Rust that decouples your business logic from the database, with InMemory, SQLite, and Postgres implementations."

[taxonomies]
tags = ["rust", "design-patterns", "architecture", "databases"]
+++

Every web application talks to a database. The question is how tightly your business logic is coupled to that database. If your handler directly calls `sqlx::query!("SELECT * FROM users WHERE id = $1", id)`, you've made a decision that ripples through your entire codebase: every test needs a live database, switching from Postgres to SQLite means rewriting queries scattered across dozens of files, and your domain logic is buried under SQL strings.

The repository pattern puts a trait between your application and the storage layer. Your business logic calls `repo.find_by_id(id)` and doesn't care whether that hits Postgres, SQLite, an in-memory `HashMap`, or a flat file. You've seen this idea before - if you read [my post on the adapter pattern](/blog/the-adapter-pattern-in-rust-wrapping-external-apis/), the repository is essentially an adapter specialized for data access. Same concept, more specific shape.

<!-- more -->

## The core trait

A repository represents a collection of entities. Not a database connection, not a query builder - a collection. You put things in, take things out, update them, remove them. The trait reflects that:

```rust
use std::future::Future;

pub trait Repository<T: Entity>: Send + Sync {
    fn find_by_id(&self, id: &str) -> impl Future<Output = Result<Option<T>, RepoError>> + Send;
    fn find_all(&self, opts: &QueryOpts) -> impl Future<Output = Result<Vec<T>, RepoError>> + Send;
    fn create(&self, input: T::Create) -> impl Future<Output = Result<T, RepoError>> + Send;
    fn update(&self, id: &str, input: T::Update) -> impl Future<Output = Result<T, RepoError>> + Send;
    fn delete(&self, id: &str) -> impl Future<Output = Result<bool, RepoError>> + Send;
    fn count(&self) -> impl Future<Output = Result<i64, RepoError>> + Send;
}
```

Since Rust 1.75, `async fn` in traits works natively - no `async_trait` crate needed for static dispatch. The syntax above uses `impl Future` return types, which is equivalent to writing `async fn` but gives us the `+ Send` bound explicitly. If you need dynamic dispatch (`dyn Repository<User>`), you still need [`async_trait`](https://crates.io/crates/async-trait) or manual boxing. More on that later.

The `Entity` trait ties a type to its create and update DTOs:

```rust
pub trait Entity: Send + Sync + Clone + 'static {
    type Create: Send;
    type Update: Send;

    fn id(&self) -> &str;
}
```

And the error type stays minimal:

```rust
#[derive(Debug, thiserror::Error)]
pub enum RepoError {
    #[error("entity not found: {0}")]
    NotFound(String),

    #[error("duplicate entity: {0}")]
    Duplicate(String),

    #[error("storage error: {0}")]
    Storage(String),
}
```

Three variants. That's it. `NotFound` for missing entities, `Duplicate` for unique constraint violations, `Storage` for everything else (connection failures, corrupt data, disk full). Don't model every possible SQL error as a variant - your business logic doesn't care about the difference between a socket timeout and a DNS failure. Both are `Storage`.

## The domain types

Before implementing anything, define what you're storing. A `User` entity with its create and update types:

```rust
#[derive(Debug, Clone)]
pub struct User {
    pub id: String,
    pub email: String,
    pub name: String,
    pub active: bool,
    pub created_at: String,
}

pub struct CreateUser {
    pub email: String,
    pub name: String,
}

#[derive(Default)]
pub struct UpdateUser {
    pub email: Option<String>,
    pub name: Option<String>,
    pub active: Option<bool>,
}

impl Entity for User {
    type Create = CreateUser;
    type Update = UpdateUser;

    fn id(&self) -> &str {
        &self.id
    }
}
```

Notice `UpdateUser` uses `Option` fields. This is the partial update pattern - only the fields that are `Some` get modified. The `Default` derive means you can write `UpdateUser { name: Some("new".into()), ..Default::default() }` without specifying every field. No need for a builder, no need for separate `UpdateUserName` / `UpdateUserEmail` types.

## InMemory implementation: tests without a database

The in-memory implementation is the reason you build the trait in the first place. It turns database-dependent tests into pure logic tests that run in microseconds:

```rust
use std::collections::HashMap;
use std::sync::RwLock;
use uuid::Uuid;

pub struct InMemoryUserRepo {
    store: RwLock<HashMap<String, User>>,
}

impl InMemoryUserRepo {
    pub fn new() -> Self {
        Self {
            store: RwLock::new(HashMap::new()),
        }
    }
}

impl Repository<User> for InMemoryUserRepo {
    async fn find_by_id(&self, id: &str) -> Result<Option<User>, RepoError> {
        let store = self.store.read().map_err(|e| RepoError::Storage(e.to_string()))?;
        Ok(store.get(id).cloned())
    }

    async fn find_all(&self, opts: &QueryOpts) -> Result<Vec<User>, RepoError> {
        let store = self.store.read().map_err(|e| RepoError::Storage(e.to_string()))?;
        let mut users: Vec<User> = store.values().cloned().collect();

        // Apply sorting
        match opts.order_by.as_deref() {
            Some("name") => users.sort_by(|a, b| a.name.cmp(&b.name)),
            Some("created_at") => users.sort_by(|a, b| a.created_at.cmp(&b.created_at)),
            _ => users.sort_by(|a, b| a.id.cmp(&b.id)),
        }

        if opts.order_desc {
            users.reverse();
        }

        // Apply pagination
        let offset = opts.offset.unwrap_or(0) as usize;
        let limit = opts.limit.unwrap_or(100) as usize;
        let users: Vec<User> = users.into_iter().skip(offset).take(limit).collect();

        Ok(users)
    }

    async fn create(&self, input: CreateUser) -> Result<User, RepoError> {
        let mut store = self.store.write().map_err(|e| RepoError::Storage(e.to_string()))?;

        // Check for duplicate email
        if store.values().any(|u| u.email == input.email) {
            return Err(RepoError::Duplicate(format!("email: {}", input.email)));
        }

        let user = User {
            id: Uuid::new_v4().to_string(),
            email: input.email,
            name: input.name,
            active: true,
            created_at: chrono::Utc::now().to_rfc3339(),
        };

        store.insert(user.id.clone(), user.clone());
        Ok(user)
    }

    async fn update(&self, id: &str, input: UpdateUser) -> Result<User, RepoError> {
        let mut store = self.store.write().map_err(|e| RepoError::Storage(e.to_string()))?;

        let user = store
            .get_mut(id)
            .ok_or_else(|| RepoError::NotFound(id.to_string()))?;

        if let Some(email) = input.email {
            user.email = email;
        }
        if let Some(name) = input.name {
            user.name = name;
        }
        if let Some(active) = input.active {
            user.active = active;
        }

        Ok(user.clone())
    }

    async fn delete(&self, id: &str) -> Result<bool, RepoError> {
        let mut store = self.store.write().map_err(|e| RepoError::Storage(e.to_string()))?;
        Ok(store.remove(id).is_some())
    }

    async fn count(&self) -> Result<i64, RepoError> {
        let store = self.store.read().map_err(|e| RepoError::Storage(e.to_string()))?;
        Ok(store.len() as i64)
    }
}
```

A few design decisions worth noting. `RwLock` instead of `Mutex` because reads are far more common than writes, and multiple tests can read concurrently. The `async fn` signatures are technically synchronous here (no `.await` calls), but that's fine - the trait demands the signature, the in-memory impl just returns immediately. The sort/pagination logic in `find_all` mirrors what the database would do, so your tests exercise the same query semantics.

## SQLite implementation: production-ready, single-file storage

For production (or at least for apps that don't need a dedicated database server), SQLite via [`sqlx`](https://crates.io/crates/sqlx) (v0.8) gives you a real SQL engine behind the same trait:

```rust
use sqlx::{SqlitePool, Row};

pub struct SqliteUserRepo {
    pool: SqlitePool,
}

impl SqliteUserRepo {
    pub async fn new(database_url: &str) -> Result<Self, RepoError> {
        let pool = SqlitePool::connect(database_url)
            .await
            .map_err(|e| RepoError::Storage(e.to_string()))?;

        // Run migrations on startup
        sqlx::query(
            "CREATE TABLE IF NOT EXISTS users (
                id TEXT PRIMARY KEY,
                email TEXT UNIQUE NOT NULL,
                name TEXT NOT NULL,
                active BOOLEAN NOT NULL DEFAULT 1,
                created_at TEXT NOT NULL
            )"
        )
        .execute(&pool)
        .await
        .map_err(|e| RepoError::Storage(e.to_string()))?;

        Ok(Self { pool })
    }
}

impl Repository<User> for SqliteUserRepo {
    async fn find_by_id(&self, id: &str) -> Result<Option<User>, RepoError> {
        let row = sqlx::query("SELECT id, email, name, active, created_at FROM users WHERE id = ?")
            .bind(id)
            .fetch_optional(&self.pool)
            .await
            .map_err(|e| RepoError::Storage(e.to_string()))?;

        Ok(row.map(|r| User {
            id: r.get("id"),
            email: r.get("email"),
            name: r.get("name"),
            active: r.get("active"),
            created_at: r.get("created_at"),
        }))
    }

    async fn find_all(&self, opts: &QueryOpts) -> Result<Vec<User>, RepoError> {
        let order_col = match opts.order_by.as_deref() {
            Some("name") => "name",
            Some("created_at") => "created_at",
            Some("email") => "email",
            _ => "id",
        };
        let direction = if opts.order_desc { "DESC" } else { "ASC" };
        let limit = opts.limit.unwrap_or(100);
        let offset = opts.offset.unwrap_or(0);

        let query = format!(
            "SELECT id, email, name, active, created_at FROM users ORDER BY {} {} LIMIT ? OFFSET ?",
            order_col, direction
        );

        let rows = sqlx::query(&query)
            .bind(limit)
            .bind(offset)
            .fetch_all(&self.pool)
            .await
            .map_err(|e| RepoError::Storage(e.to_string()))?;

        Ok(rows
            .iter()
            .map(|r| User {
                id: r.get("id"),
                email: r.get("email"),
                name: r.get("name"),
                active: r.get("active"),
                created_at: r.get("created_at"),
            })
            .collect())
    }

    async fn create(&self, input: CreateUser) -> Result<User, RepoError> {
        let id = Uuid::new_v4().to_string();
        let now = chrono::Utc::now().to_rfc3339();

        sqlx::query(
            "INSERT INTO users (id, email, name, active, created_at) VALUES (?, ?, ?, 1, ?)"
        )
        .bind(&id)
        .bind(&input.email)
        .bind(&input.name)
        .bind(&now)
        .execute(&self.pool)
        .await
        .map_err(|e| {
            if e.to_string().contains("UNIQUE constraint failed") {
                RepoError::Duplicate(format!("email: {}", input.email))
            } else {
                RepoError::Storage(e.to_string())
            }
        })?;

        Ok(User {
            id,
            email: input.email,
            name: input.name,
            active: true,
            created_at: now,
        })
    }

    async fn update(&self, id: &str, input: UpdateUser) -> Result<User, RepoError> {
        let existing = self.find_by_id(id).await?
            .ok_or_else(|| RepoError::NotFound(id.to_string()))?;

        let email = input.email.unwrap_or(existing.email);
        let name = input.name.unwrap_or(existing.name);
        let active = input.active.unwrap_or(existing.active);

        sqlx::query("UPDATE users SET email = ?, name = ?, active = ? WHERE id = ?")
            .bind(&email)
            .bind(&name)
            .bind(active)
            .bind(id)
            .execute(&self.pool)
            .await
            .map_err(|e| RepoError::Storage(e.to_string()))?;

        Ok(User {
            id: id.to_string(),
            email,
            name,
            active,
            created_at: existing.created_at,
        })
    }

    async fn delete(&self, id: &str) -> Result<bool, RepoError> {
        let result = sqlx::query("DELETE FROM users WHERE id = ?")
            .bind(id)
            .execute(&self.pool)
            .await
            .map_err(|e| RepoError::Storage(e.to_string()))?;

        Ok(result.rows_affected() > 0)
    }

    async fn count(&self) -> Result<i64, RepoError> {
        let row = sqlx::query("SELECT COUNT(*) as cnt FROM users")
            .fetch_one(&self.pool)
            .await
            .map_err(|e| RepoError::Storage(e.to_string()))?;

        Ok(row.get::<i64, _>("cnt"))
    }
}
```

The `order_by` column is matched against a whitelist. This is important - you never interpolate user input directly into a SQL string, not even column names. The `format!` call is safe because `order_col` and `direction` are both from controlled match arms, not from user input.

The error mapping in `create` detects SQLite's `UNIQUE constraint failed` message and converts it to `RepoError::Duplicate`. This is the same error-mapping approach from [the adapter pattern post](/blog/the-adapter-pattern-in-rust-wrapping-external-apis/) - translate infrastructure errors into domain errors at the boundary.

## Postgres implementation: same trait, different backend

The Postgres version looks almost identical. That's the point - the trait shape doesn't change, just the SQL dialect and connection type:

```rust
use sqlx::{PgPool, Row};

pub struct PgUserRepo {
    pool: PgPool,
}

impl PgUserRepo {
    pub async fn new(database_url: &str) -> Result<Self, RepoError> {
        let pool = PgPool::connect(database_url)
            .await
            .map_err(|e| RepoError::Storage(e.to_string()))?;

        Ok(Self { pool })
    }
}

impl Repository<User> for PgUserRepo {
    async fn find_by_id(&self, id: &str) -> Result<Option<User>, RepoError> {
        let row = sqlx::query(
            "SELECT id, email, name, active, created_at FROM users WHERE id = $1"
        )
        .bind(id)
        .fetch_optional(&self.pool)
        .await
        .map_err(|e| RepoError::Storage(e.to_string()))?;

        Ok(row.map(|r| User {
            id: r.get("id"),
            email: r.get("email"),
            name: r.get("name"),
            active: r.get("active"),
            created_at: r.get("created_at"),
        }))
    }

    async fn create(&self, input: CreateUser) -> Result<User, RepoError> {
        let id = Uuid::new_v4().to_string();
        let now = chrono::Utc::now().to_rfc3339();

        sqlx::query(
            "INSERT INTO users (id, email, name, active, created_at) VALUES ($1, $2, $3, true, $4)"
        )
        .bind(&id)
        .bind(&input.email)
        .bind(&input.name)
        .bind(&now)
        .execute(&self.pool)
        .await
        .map_err(|e| {
            let msg = e.to_string();
            if msg.contains("duplicate key value") {
                RepoError::Duplicate(format!("email: {}", input.email))
            } else {
                RepoError::Storage(msg)
            }
        })?;

        Ok(User {
            id,
            email: input.email,
            name: input.name,
            active: true,
            created_at: now,
        })
    }

    // find_all, update, delete, count follow the same pattern
    // with $1/$2 parameter syntax instead of ?
    // ...
}
```

The differences between SQLite and Postgres implementations: parameter placeholders (`?` vs `$1`), boolean literals (`1` vs `true`), and unique constraint error messages (`UNIQUE constraint failed` vs `duplicate key value`). Everything else is identical. If that duplication bothers you - and it should - you can extract the row-to-entity mapping into a shared function or use `sqlx::FromRow` derive to eliminate the manual `.get()` calls entirely.

## The QueryOpts pattern

Pagination and sorting show up in every list endpoint. Instead of passing `limit`, `offset`, and `order_by` as separate parameters, bundle them:

```rust
#[derive(Debug, Clone, Default)]
pub struct QueryOpts {
    pub limit: Option<i64>,
    pub offset: Option<i64>,
    pub order_by: Option<String>,
    pub order_desc: bool,
}

impl QueryOpts {
    pub fn paginate(limit: i64, offset: i64) -> Self {
        Self {
            limit: Some(limit),
            offset: Some(offset),
            ..Default::default()
        }
    }

    pub fn order(field: &str, desc: bool) -> Self {
        Self {
            order_by: Some(field.to_string()),
            order_desc: desc,
            ..Default::default()
        }
    }

    pub fn first(n: i64) -> Self {
        Self {
            limit: Some(n),
            ..Default::default()
        }
    }
}
```

Usage reads naturally:

```rust
// Get first 20 users, newest first
let users = repo.find_all(&QueryOpts::order("created_at", true)
    .with_limit(20))
    .await?;

// Page 3, 10 per page
let users = repo.find_all(&QueryOpts::paginate(10, 20)).await?;

// Just give me everything (with default limit)
let users = repo.find_all(&QueryOpts::default()).await?;
```

You could extend `QueryOpts` with filtering (`where_clause`, `filters: HashMap<String, String>`), but be careful. The more query logic you push into the generic `QueryOpts`, the harder it is to implement consistently across backends. Simple pagination and sorting work everywhere. Complex filtering is where you start wanting entity-specific query methods:

```rust
// Instead of trying to make QueryOpts handle this:
pub trait UserRepository: Repository<User> {
    async fn find_by_email(&self, email: &str) -> Result<Option<User>, RepoError>;
    async fn find_active(&self, opts: &QueryOpts) -> Result<Vec<User>, RepoError>;
}
```

This is a sub-trait that adds user-specific queries on top of the generic CRUD. The InMemory implementation filters by `user.active == true`. The SQLite implementation adds `WHERE active = 1`. Same result, different mechanisms, unified interface.

## Dynamic dispatch: when you need dyn Repository

If the implementation is known at compile time (your tests always use `InMemoryUserRepo`, production always uses `SqliteUserRepo`), generics work great. The compiler monomorphizes everything, zero overhead.

But sometimes you need to pick the backend at runtime - maybe based on a config flag, or in a service that accepts any repository. That requires `dyn Repository<User>`, and that's where things get interesting.

Native `async fn` in traits (stable since Rust 1.75) doesn't support `dyn Trait` dispatch. The compiler doesn't know how to vtable a method that returns `impl Future` - each implementation could return a different future type with a different size. You need type erasure.

Two options. The first is [`async_trait`](https://crates.io/crates/async-trait), which rewrites your methods to return `Pin<Box<dyn Future>>`:

```rust
use async_trait::async_trait;

#[async_trait]
pub trait DynRepository<T: Entity>: Send + Sync {
    async fn find_by_id(&self, id: &str) -> Result<Option<T>, RepoError>;
    async fn find_all(&self, opts: &QueryOpts) -> Result<Vec<T>, RepoError>;
    async fn create(&self, input: T::Create) -> Result<T, RepoError>;
    async fn update(&self, id: &str, input: T::Update) -> Result<T, RepoError>;
    async fn delete(&self, id: &str) -> Result<bool, RepoError>;
    async fn count(&self) -> Result<i64, RepoError>;
}
```

Now `Arc<dyn DynRepository<User>>` works, and you can swap backends at runtime. The cost is one heap allocation per method call for the boxed future. In a web app where each request already involves network I/O, serialization, and probably a database round trip, that allocation is noise. Don't optimize it away unless profiling says otherwise.

The second option is to write the boxing yourself:

```rust
pub trait DynRepository<T: Entity>: Send + Sync {
    fn find_by_id(&self, id: &str) -> Pin<Box<dyn Future<Output = Result<Option<T>, RepoError>> + Send + '_>>;
    // ... same for other methods
}
```

This is what `async_trait` generates under the hood. Writing it manually gives you control but buys you nothing unless you need a custom allocation strategy. Stick with `async_trait` for ergonomics.

If you're curious what the `async_trait` expansion looks like, I covered [`cargo-expand`](/blog/debugging-rust-beyond-println/) as a debugging tool in an earlier post - run it on a module with `#[async_trait]` and you'll see exactly the desugaring.

## Wiring it up with Arc

The application layer receives the repository through dependency injection. `Arc<dyn DynRepository<User>>` is the idiomatic way to share a repository across handlers in a web framework:

```rust
use std::sync::Arc;

pub struct UserService {
    repo: Arc<dyn DynRepository<User>>,
}

impl UserService {
    pub fn new(repo: Arc<dyn DynRepository<User>>) -> Self {
        Self { repo }
    }

    pub async fn register(&self, email: String, name: String) -> Result<User, RepoError> {
        // Business logic: validate, then persist
        if email.is_empty() || !email.contains('@') {
            return Err(RepoError::Storage("invalid email".into()));
        }

        self.repo.create(CreateUser { email, name }).await
    }

    pub async fn deactivate(&self, user_id: &str) -> Result<User, RepoError> {
        self.repo.update(user_id, UpdateUser {
            active: Some(false),
            ..Default::default()
        }).await
    }
}
```

At the application entry point, you pick the backend:

```rust
#[tokio::main]
async fn main() {
    let repo: Arc<dyn DynRepository<User>> = if std::env::var("DATABASE_URL").is_ok() {
        let url = std::env::var("DATABASE_URL").unwrap();
        Arc::new(PgUserRepo::new(&url).await.expect("db connection failed"))
    } else {
        Arc::new(SqliteUserRepo::new("sqlite:app.db").await.expect("sqlite init failed"))
    };

    let service = UserService::new(repo);
    // Pass service to your web framework router...
}
```

Tests get the in-memory version:

```rust
#[tokio::test]
async fn test_register_validates_email() {
    let repo = Arc::new(InMemoryUserRepo::new());
    let service = UserService::new(repo);

    let result = service.register("not-an-email".into(), "Alice".into()).await;
    assert!(result.is_err());
}

#[tokio::test]
async fn test_register_creates_user() {
    let repo = Arc::new(InMemoryUserRepo::new());
    let service = UserService::new(repo.clone());

    let user = service.register("alice@example.com".into(), "Alice".into()).await.unwrap();
    assert_eq!(user.name, "Alice");
    assert!(user.active);

    // Verify through the repo directly
    let found = repo.find_by_id(&user.id).await.unwrap();
    assert!(found.is_some());
}

#[tokio::test]
async fn test_deactivate_sets_active_false() {
    let repo = Arc::new(InMemoryUserRepo::new());
    let service = UserService::new(repo);

    let user = service.register("bob@example.com".into(), "Bob".into()).await.unwrap();
    let deactivated = service.deactivate(&user.id).await.unwrap();
    assert!(!deactivated.active);
}
```

No database setup, no teardown, no test containers, no flaky CI. Tests run in microseconds and test actual business logic.

## Repository vs ORM: when to use which

Rust has mature ORMs. [Diesel](https://diesel.rs/) (v2.3) gives you compile-time SQL validation and a type-safe query builder. [SeaORM](https://www.sea-ql.org/SeaORM/) (v1.1) offers async-first design with a more ActiveRecord-like API. Both are excellent. So when do you reach for a hand-rolled repository instead?

**Use an ORM when:**

- Your data model maps directly to your API surface. If `User` in the database is roughly `User` in the JSON response, an ORM generates the boring CRUD code for you and does it correctly.
- You have complex queries with joins, aggregations, subqueries. Diesel's query builder catches type errors at compile time. Writing those queries by hand in a repository means testing them manually.
- You want migrations, schema management, and CLI tooling out of the box. `diesel_cli` and `sea-orm-cli` handle migration generation and execution.

**Use a repository when:**

- Your domain model diverges from your storage model. Maybe you store normalized tables but expose denormalized aggregates. The repository does the translation. An ORM wants your domain types and database types to be the same thing.
- You need backend flexibility. ORMs are typically tied to one (or a few) database engines. A repository trait lets you implement `InMemoryRepo` for tests, `SqliteRepo` for dev, and `PgRepo` for production. Try swapping Diesel's Postgres backend for an in-memory `HashMap` in tests - it doesn't work without a real database.
- Your application has a clean architecture boundary. If you practice hexagonal architecture or domain-driven design, the repository trait is your port. ORM types belong in the infrastructure layer, not in your domain.
- Performance hotspots require hand-tuned SQL. ORMs generate SQL that's usually fine but sometimes suboptimal. A repository lets you write exact SQL for the queries that matter while keeping the same interface.

**The hybrid approach** is what I use most often: a repository trait at the boundary, with the implementation using sqlx or even Diesel internally. Your handlers call `repo.find_active_users()`. The repo implementation uses `sqlx::query!()` or Diesel's DSL. You get testability from the trait and query safety from the ORM/query builder:

```rust
impl DynRepository<User> for DieselUserRepo {
    async fn find_by_id(&self, id: &str) -> Result<Option<User>, RepoError> {
        use crate::schema::users::dsl;
        let mut conn = self.pool.get()
            .map_err(|e| RepoError::Storage(e.to_string()))?;

        let result = dsl::users
            .filter(dsl::id.eq(id))
            .first::<UserRow>(&mut conn)
            .optional()
            .map_err(|e| RepoError::Storage(e.to_string()))?;

        Ok(result.map(UserRow::into_domain))
    }
}
```

The `UserRow` is Diesel's `#[derive(Queryable)]` struct. `into_domain()` maps it to your domain `User`. The repository trait hides all of this from the rest of your application. You could swap from Diesel to raw sqlx tomorrow and nothing outside the `infra/` module would change.

## Practical comparison: the same query three ways

To make this concrete, here's a "find active users, sorted by name, paginated" query in each approach:

**Raw repository with sqlx:**

```rust
async fn find_active(&self, opts: &QueryOpts) -> Result<Vec<User>, RepoError> {
    let limit = opts.limit.unwrap_or(100);
    let offset = opts.offset.unwrap_or(0);

    let rows = sqlx::query(
        "SELECT id, email, name, active, created_at FROM users
         WHERE active = true ORDER BY name ASC LIMIT $1 OFFSET $2"
    )
    .bind(limit)
    .bind(offset)
    .fetch_all(&self.pool)
    .await
    .map_err(|e| RepoError::Storage(e.to_string()))?;

    Ok(rows.iter().map(row_to_user).collect())
}
```

**Diesel:**

```rust
fn find_active(conn: &mut PgConnection, limit: i64, offset: i64) -> QueryResult<Vec<UserRow>> {
    use crate::schema::users::dsl;

    dsl::users
        .filter(dsl::active.eq(true))
        .order(dsl::name.asc())
        .limit(limit)
        .offset(offset)
        .load::<UserRow>(conn)
}
```

**SeaORM:**

```rust
async fn find_active(db: &DatabaseConnection, limit: u64, offset: u64) -> Result<Vec<user::Model>, DbErr> {
    User::find()
        .filter(user::Column::Active.eq(true))
        .order_by_asc(user::Column::Name)
        .paginate(db, limit)
        .fetch_page(offset / limit)
        .await
}
```

All three produce the same SQL. Diesel catches type mismatches at compile time (try filtering `active.eq("yes")` - it won't compile). SeaORM gives you async out of the box. The raw sqlx approach is the most flexible but the least safe - misspell a column name and you find out at runtime.

The repository pattern doesn't replace any of these. It wraps whichever one you choose behind a trait so the rest of your application doesn't need to know.

## What I actually recommend

For new Rust projects:

1. **Start with the repository trait.** Even if you only have one implementation. The trait forces you to think about what operations your domain needs, not what your database can do. That's a better starting point.

2. **Use sqlx for the implementation.** It hits the sweet spot between control and convenience. Compile-time checked queries (with the `query!` macro), async native, supports Postgres/MySQL/SQLite. You write SQL directly, which means no ORM abstraction to fight when you need a complex query.

3. **Build the InMemory implementation immediately.** It takes 30 minutes and saves hours in test speed and reliability over the life of the project. If you skip it, you'll end up with either slow integration tests or no tests at all.

4. **Add entity-specific query methods as needed.** Don't pre-build `find_by_email`, `find_by_name`, `find_active` before you need them. Start with the five core methods (`find_by_id`, `find_all`, `create`, `update`, `delete`). Add specific queries when business logic demands them.

5. **Consider Diesel or SeaORM when the schema gets complex.** 15+ tables with foreign keys, joins, and complex aggregations - that's where a proper query builder earns its keep. Wrap it behind the repository trait anyway, but let the ORM handle the SQL generation.

The repository pattern isn't about avoiding SQL. It's about giving your application a stable interface to data that doesn't change when your storage decisions do. Today it's SQLite, next month it's Postgres, and in tests it's a `HashMap`. Your business logic doesn't care, and that's the whole point.
