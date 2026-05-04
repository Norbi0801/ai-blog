+++
title = "Database migrations in Rust - sqlx, diesel, and sea-orm compared"
date = 2025-08-08
description = "Three approaches to database access in Rust: raw compile-time checked SQL, a full ORM with a type-safe DSL, and an async ActiveRecord layer - how their migration workflows, query APIs, and tradeoffs differ in practice."

[taxonomies]
tags = ["rust", "databases", "sqlx", "diesel"]
+++

Rust has three serious options for talking to databases. [sqlx](https://github.com/launchbadge/sqlx) gives you raw SQL with compile-time verification. [Diesel](https://diesel.rs/) gives you a type-safe query DSL and a full ORM. [SeaORM](https://www.sea-ql.org/SeaORM/) builds on top of sqlx to add an async-first ActiveRecord layer. Each one makes fundamentally different tradeoffs around safety, flexibility, and developer experience.

I touched on all three briefly in [the repository pattern post](/blog/the-repository-pattern-abstracting-data-access-in-rust/) when comparing query approaches. This post goes deeper - specifically into how each one handles migrations, what their query APIs actually look like under the hood, and when each approach makes sense.

<!-- more -->

## The migration workflow - three very different experiences

Migrations are where you feel the philosophy difference most. Each library has its own CLI tool, its own file format, and its own opinion about how schema changes should flow through your project.

### sqlx: plain SQL files

sqlx's migration system is minimal by design. You install the CLI, create a migrations directory, and write SQL by hand:

```bash
cargo install sqlx-cli --no-default-features --features sqlite
sqlx database create
sqlx migrate add create_users
```

This generates a file like `migrations/20260510120000_create_users.sql`. You write the SQL:

```sql
CREATE TABLE users (
    id TEXT PRIMARY KEY NOT NULL,
    email TEXT UNIQUE NOT NULL,
    name TEXT NOT NULL,
    active BOOLEAN NOT NULL DEFAULT true,
    created_at TEXT NOT NULL DEFAULT (datetime('now'))
);
```

That's it. No Rust code, no DSL, no code generation. The migration is a SQL file. You run it with `sqlx migrate run`, and sqlx tracks which migrations have been applied in a `_sqlx_migrations` table.

For reversible migrations, use `sqlx migrate add -r create_users`, which generates both an `up.sql` and a `down.sql`. But most teams using sqlx don't bother with down migrations. The reasoning is pragmatic: in production, you almost never roll back a migration. You write a new migration that undoes the change. Down migrations give you a false sense of reversibility that breaks the moment your up migration involved data transformation.

sqlx also supports running migrations from Rust code at startup:

```rust
use sqlx::sqlite::SqlitePoolOptions;
use sqlx::migrate::Migrator;

static MIGRATOR: Migrator = sqlx::migrate!(); // embeds migrations at compile time

#[tokio::main]
async fn main() {
    let pool = SqlitePoolOptions::new()
        .max_connections(5)
        .connect("sqlite:app.db")
        .await
        .expect("failed to connect");

    MIGRATOR.run(&pool).await.expect("migration failed");
}
```

The `sqlx::migrate!()` macro reads the `migrations/` directory at compile time and embeds the SQL into your binary. No filesystem access needed at runtime. This is useful for single-binary deployments where you don't want to ship migration files alongside your executable.

### Diesel: schema.rs and the print_schema workflow

Diesel takes a completely different approach. Migrations are still SQL files, but Diesel adds a generated `schema.rs` file that acts as the bridge between your database schema and Rust's type system.

```bash
cargo install diesel_cli --no-default-features --features sqlite
diesel setup
diesel migration generate create_users
```

This creates `migrations/20260510120000_create_users/up.sql` and `down.sql`. You write both:

```sql
-- up.sql
CREATE TABLE users (
    id TEXT PRIMARY KEY NOT NULL,
    email TEXT UNIQUE NOT NULL,
    name TEXT NOT NULL,
    active BOOLEAN NOT NULL DEFAULT 1,
    created_at TEXT NOT NULL DEFAULT (datetime('now'))
);

-- down.sql
DROP TABLE users;
```

When you run `diesel migration run`, two things happen. First, the migration executes against your database. Second - and this is the key difference - Diesel regenerates `src/schema.rs`:

```rust
// @generated automatically by Diesel CLI.

diesel::table! {
    users (id) {
        id -> Text,
        email -> Text,
        name -> Text,
        active -> Bool,
        created_at -> Text,
    }
}
```

This `table!` macro generates a module with types representing every column and the table itself. These types are what make Diesel's compile-time query checking work. When you write `users::dsl::email.eq("alice@example.com")`, the compiler knows that `email` is a `Text` column and that `.eq()` requires a compatible type. Pass an integer and the code won't compile.

The `diesel.toml` file at your project root controls this behavior:

```toml
[print_schema]
file = "src/schema.rs"
```

Every time you run or revert a migration, Diesel automatically runs `diesel print_schema` and writes the output to that file. This means your schema.rs stays in sync with your actual database schema. You check this file into version control - it's generated code, but it's also the source of truth for your Rust types.

Diesel 2.3 also added `--diff-schema` to `diesel migration generate`, which can look at your current database and your Queryable structs and generate migration SQL automatically. It doesn't cover every case, but for adding or removing columns it saves time.

The downside: Diesel requires a live database connection during development. The `diesel` CLI needs to connect to your database to introspect the schema and generate `schema.rs`. There's no offline mode equivalent to sqlx's `cargo sqlx prepare`.

### SeaORM: Rust-based migrations with entity generation

SeaORM's migration system is the most code-heavy of the three. Migrations are Rust structs, not SQL files:

```bash
cargo install sea-orm-cli
sea-orm-cli migrate init
```

This creates a `migration/` crate (a separate Cargo project) with a `Migrator` struct. Each migration is a Rust file:

```rust
use sea_orm_migration::prelude::*;

#[derive(DeriveMigrationName)]
pub struct Migration;

#[async_trait::async_trait]
impl MigrationTrait for Migration {
    async fn up(&self, manager: &SchemaManager) -> Result<(), DbErr> {
        manager.create_table(
            Table::create()
                .table(Users::Table)
                .if_not_exists()
                .col(ColumnDef::new(Users::Id).string().not_null().primary_key())
                .col(ColumnDef::new(Users::Email).string().not_null().unique_key())
                .col(ColumnDef::new(Users::Name).string().not_null())
                .col(ColumnDef::new(Users::Active).boolean().not_null().default(true))
                .col(ColumnDef::new(Users::CreatedAt).string().not_null())
                .to_owned(),
        ).await
    }

    async fn down(&self, manager: &SchemaManager) -> Result<(), DbErr> {
        manager.drop_table(Table::drop().table(Users::Table).to_owned()).await
    }
}

#[derive(DeriveIden)]
enum Users {
    Table,
    Id,
    Email,
    Name,
    Active,
    CreatedAt,
}
```

That's a lot more code than a SQL file. The `DeriveIden` enum defines identifiers for your table and columns. The `SchemaManager` API provides a database-agnostic way to create tables, add columns, create indexes. The generated SQL differs based on your backend - Postgres gets `CREATE TABLE`, SQLite gets `CREATE TABLE`, MySQL gets `CREATE TABLE`, each with their dialect-specific types.

After running migrations, you generate entity files from the live database:

```bash
sea-orm-cli generate entity -o src/entities
```

This introspects your database and generates Rust files like `src/entities/users.rs`:

```rust
use sea_orm::entity::prelude::*;

#[derive(Clone, Debug, PartialEq, DeriveEntityModel)]
#[sea_orm(table_name = "users")]
pub struct Model {
    #[sea_orm(primary_key, auto_increment = false)]
    pub id: String,
    #[sea_orm(unique)]
    pub email: String,
    pub name: String,
    pub active: bool,
    pub created_at: String,
}

#[derive(Copy, Clone, Debug, EnumIter, DeriveRelation)]
pub enum Relation {}

impl ActiveModelBehavior for ActiveModel {}
```

So the workflow is: write migration in Rust, run it, generate entities from the database. This is the opposite direction from Diesel, where you write the migration SQL and the schema.rs gets generated. SeaORM generates entity Rust code from the database; Diesel generates schema macros from the database. Both need a live database for code generation, but the artifacts look very different.

## The same query, three ways

Theory is nice. Code is better. Here's a realistic query - "find all active users with a given email domain, ordered by creation date, limit 20" - in each library.

### sqlx: raw SQL with compile-time checking

```rust
use sqlx::FromRow;

#[derive(Debug, FromRow)]
struct User {
    id: String,
    email: String,
    name: String,
    active: bool,
    created_at: String,
}

async fn find_active_by_domain(
    pool: &sqlx::SqlitePool,
    domain: &str,
) -> Result<Vec<User>, sqlx::Error> {
    let pattern = format!("%@{domain}");

    sqlx::query_as!(
        User,
        r#"
        SELECT id, email, name, active, created_at
        FROM users
        WHERE active = true AND email LIKE ?
        ORDER BY created_at DESC
        LIMIT 20
        "#,
        pattern
    )
    .fetch_all(pool)
    .await
}
```

The `query_as!` macro does something interesting at compile time. It connects to your development database (via `DATABASE_URL`), sends the query to the database engine's query planner, and verifies that: the SQL is syntactically valid, the referenced tables and columns exist, the bind parameter types match, and the result columns map to the struct fields. If any of that fails, you get a compile error, not a runtime error.

This is worth pausing on. The macro doesn't parse SQL itself. It literally asks your database "is this query valid?" during compilation. That means it catches things no SQL parser could - like referencing a column that was dropped in a recent migration, or using a function that doesn't exist in your SQLite version.

The tradeoff: you need a running database during compilation (or the offline cache). Run `cargo sqlx prepare` and it serializes the query metadata into a `.sqlx/` directory that you commit to your repo. CI can then build without a database by reading from this cache. But every time you change a query or a migration, you need to regenerate the cache.

For queries where you don't need the compile-time check, there's also `sqlx::query()` (without the `!`), which is a plain runtime-checked query builder:

```rust
let rows = sqlx::query("SELECT id, name FROM users WHERE active = ?")
    .bind(true)
    .fetch_all(pool)
    .await?;

for row in rows {
    let id: String = row.get("id");
    let name: String = row.get("name");
}
```

This compiles without a database connection but gives you no safety net. Misspell a column name and you get a runtime panic.

### Diesel: type-safe DSL

```rust
use diesel::prelude::*;

// Generated by diesel in schema.rs
mod schema {
    diesel::table! {
        users (id) {
            id -> Text,
            email -> Text,
            name -> Text,
            active -> Bool,
            created_at -> Text,
        }
    }
}

#[derive(Queryable, Selectable, Debug)]
#[diesel(table_name = schema::users)]
struct User {
    id: String,
    email: String,
    name: String,
    active: bool,
    created_at: String,
}

fn find_active_by_domain(
    conn: &mut SqliteConnection,
    domain: &str,
) -> QueryResult<Vec<User>> {
    use schema::users::dsl::*;

    let pattern = format!("%@{domain}");

    users
        .filter(active.eq(true))
        .filter(email.like(&pattern))
        .order(created_at.desc())
        .limit(20)
        .select(User::as_select())
        .load(conn)
}
```

No SQL strings anywhere. The query is built using Rust method calls and types from the generated `schema.rs`. The compiler verifies everything: `active.eq(true)` works because `active` is `Bool` and `true` is `bool`. Try `active.eq("yes")` and you get a type error. Try `email.eq(42)` and you get a type error. Try referencing a column that doesn't exist in `schema.rs` and you get a compile error.

The important distinction from sqlx's approach: Diesel doesn't need a database at compile time. The type checking comes from the `table!` macro and Rust's type system, not from asking a live database. The `schema.rs` file is the contract. As long as it's in sync with your actual database (which `diesel migration run` ensures), the types are correct.

The function signature is synchronous - `&mut SqliteConnection`, not an async pool. Diesel 2.x is synchronous by default. For async, you need the [`diesel-async`](https://crates.io/crates/diesel-async) crate, which wraps connections in an async pool:

```rust
use diesel_async::RunQueryDsl;
use diesel_async::pooled_connection::deadpool::Pool;

async fn find_active_by_domain(
    pool: &Pool<AsyncSqliteConnection>,
    domain: &str,
) -> QueryResult<Vec<User>> {
    use schema::users::dsl::*;

    let pattern = format!("%@{domain}");
    let mut conn = pool.get().await.map_err(|e| /* ... */)?;

    users
        .filter(active.eq(true))
        .filter(email.like(&pattern))
        .order(created_at.desc())
        .limit(20)
        .select(User::as_select())
        .load(&mut conn)
        .await
}
```

The query builder API stays the same. Only the execution changes - `.load(conn)` becomes `.load(&mut conn).await`.

### SeaORM: ActiveRecord-style query builder

```rust
use sea_orm::*;

// Generated by sea-orm-cli
mod entities {
    pub mod users {
        use sea_orm::entity::prelude::*;

        #[derive(Clone, Debug, PartialEq, DeriveEntityModel)]
        #[sea_orm(table_name = "users")]
        pub struct Model {
            #[sea_orm(primary_key, auto_increment = false)]
            pub id: String,
            pub email: String,
            pub name: String,
            pub active: bool,
            pub created_at: String,
        }

        #[derive(Copy, Clone, Debug, EnumIter, DeriveRelation)]
        pub enum Relation {}

        impl ActiveModelBehavior for ActiveModel {}
    }
}

use entities::users;

async fn find_active_by_domain(
    db: &DatabaseConnection,
    domain: &str,
) -> Result<Vec<users::Model>, DbErr> {
    let pattern = format!("%@{domain}");

    users::Entity::find()
        .filter(users::Column::Active.eq(true))
        .filter(users::Column::Email.like(&pattern))
        .order_by_desc(users::Column::CreatedAt)
        .paginate(db, 20)
        .fetch_page(0)
        .await
}
```

SeaORM's API reads like a fluent query builder, similar to what you'd see in Laravel's Eloquent or Ruby's ActiveRecord. The `Entity::find()` starts a SELECT, `.filter()` adds WHERE clauses, `.order_by_desc()` adds ORDER BY, and `.paginate()` handles LIMIT/OFFSET.

SeaORM is async from the ground up - it uses sqlx internally for the actual database connection and query execution. The `DatabaseConnection` is an async connection pool. No separate `-async` crate needed.

For mutations, SeaORM uses the ActiveModel pattern:

```rust
use sea_orm::ActiveValue::Set;

async fn create_user(db: &DatabaseConnection, email: String, name: String) -> Result<users::Model, DbErr> {
    let user = users::ActiveModel {
        id: Set(uuid::Uuid::new_v4().to_string()),
        email: Set(email),
        name: Set(name),
        active: Set(true),
        created_at: Set(chrono::Utc::now().to_rfc3339()),
    };

    user.insert(db).await
}

async fn deactivate_user(db: &DatabaseConnection, user_id: &str) -> Result<users::Model, DbErr> {
    let mut user: users::ActiveModel = users::Entity::find_by_id(user_id)
        .one(db)
        .await?
        .ok_or(DbErr::RecordNotFound(user_id.to_string()))?
        .into();

    user.active = Set(false);
    user.update(db).await
}
```

`Set(value)` marks a field as "changed, use this value." `NotSet` means "don't touch this field." This partial update pattern is similar to the `Option`-based `UpdateUser` struct I showed in [the repository pattern post](/blog/the-repository-pattern-abstracting-data-access-in-rust/), but baked into the framework.

## What happens under the hood: compile-time checking

The `sqlx::query!` macro is probably the most interesting piece of compile-time machinery in the Rust database ecosystem. Here's what actually happens when you write:

```rust
let user = sqlx::query!("SELECT id, name FROM users WHERE id = ?", user_id)
    .fetch_one(pool)
    .await?;
```

During compilation, the proc macro:

1. Reads `DATABASE_URL` from your environment (or `.env` file)
2. Opens a connection to that database
3. Sends a `PREPARE` statement (or equivalent) to the database with your SQL
4. The database parses the query, resolves table and column references, and reports back: the number and types of bind parameters, and the number, names, and types of result columns
5. The macro maps database types to Rust types (e.g., `TEXT` becomes `String`, `INTEGER` becomes `i32`, `BOOLEAN` becomes `bool`)
6. It generates a struct with fields matching the result columns and wraps the whole thing in type-safe code

The generated code for the query above looks roughly like this (simplified):

```rust
// What sqlx::query! expands to (approximately)
{
    struct QueryResult {
        id: String,
        name: String,
    }

    sqlx::query_scalar_unchecked("SELECT id, name FROM users WHERE id = ?")
        .bind(user_id)
        .map(|row| QueryResult {
            id: row.get(0),
            name: row.get(1),
        })
}
```

You can see this yourself by running `cargo expand` on a module that uses `query!` - I covered that tool in [the debugging post](/blog/debugging-rust-beyond-println/). The actual expansion is more complex (it handles nullability, multiple database backends, and the offline mode cache), but the principle is straightforward: ask the database what the query looks like, generate Rust types to match.

The offline mode (`cargo sqlx prepare`) serializes step 4's output into JSON files in a `.sqlx/` directory. During CI or when `DATABASE_URL` isn't set, the macro reads from these files instead of connecting to a live database. The JSON looks like:

```json
{
  "query": "SELECT id, name FROM users WHERE id = ?",
  "describe": {
    "columns": [
      { "name": "id", "type_info": "TEXT", "nullable": false },
      { "name": "name", "type_info": "TEXT", "nullable": false }
    ],
    "parameters": { "Right": 1 }
  }
}
```

This is clever but has a sharp edge: if you change a migration and forget to re-run `cargo sqlx prepare`, your CI builds against stale metadata. The code compiles, passes tests with the offline cache, and then fails in production because the actual database schema doesn't match. You need CI discipline to either always `prepare` after migration changes or run CI with a real database.

## Performance: where it actually matters

The [Diesel benchmark suite](https://github.com/diesel-rs/diesel/tree/master/diesel_bench) has shown cases where sqlx is significantly slower than Diesel for certain operations. The gap is real but nuanced.

Diesel's synchronous API avoids the overhead of async task scheduling for database operations. When your query takes 200 microseconds and the async runtime's task scheduling adds 5-10 microseconds, that overhead is noise. But for micro-benchmarks doing thousands of trivial queries, it adds up. In the Diesel benchmark suite, this showed up as sqlx being slower by a meaningful factor on simple operations.

For real applications, the bottleneck is almost always the database itself, not the Rust code around it. Network latency to Postgres (even on localhost) dwarfs any difference between these libraries. If you've read [the SQLite WAL post](/blog/why-sqlite-with-wal-mode-is-good-enough-for-most-web-apps/), you know that even SQLite can saturate most applications' throughput needs. The library choice matters far less than your indexes and query plans.

That said, there's one performance-relevant architectural difference: **query pipelining**. `diesel-async` and `tokio-postgres` support sending multiple queries without waiting for each response before sending the next. sqlx and sea-orm don't support this. The crates.io team reported a 20% performance improvement on one endpoint after switching to query pipelining with `diesel-async`. If your application sends sequences of dependent queries in tight loops, this matters.

SeaORM adds overhead on top of sqlx because it builds an abstraction layer. The ActiveModel pattern involves creating intermediate structs, cloning values, and doing runtime checks on which fields are `Set` vs `NotSet`. For CRUD operations this is negligible. For bulk operations processing thousands of rows, it's worth measuring.

## The DX comparison

Developer experience is subjective, but some things are measurable.

**Compile times**: Diesel's proc macros and the `table!` macro expansion add noticeable compile time on large schemas. sqlx's `query!` macro is fast when reading from the offline cache, slow when it needs to hit the database. SeaORM's entity generation is a one-time cost, but the `DeriveEntityModel` macro adds to incremental build times. None of them are fast - database libraries in Rust are some of the heaviest proc macro users in the ecosystem.

**Error messages**: Diesel wins here. Because everything is expressed through Rust types, you get standard Rust type errors. "Expected `diesel::sql_types::Bool`, found `diesel::sql_types::Text`" is clear. sqlx's compile-time errors come from the proc macro and can be cryptic - "error returned from database: no such column: emal" is helpful, but "type mismatch for column 3" less so. SeaORM's errors are runtime by nature (it doesn't do compile-time query checking), so you discover mistakes when your code runs.

**IDE support**: Diesel and SeaORM have an edge because they use standard Rust types and method chains. Your IDE can autocomplete `.filter(users::Column::` and show you every column. sqlx queries are SQL strings - your IDE might syntax-highlight them if you're lucky, but there's no autocomplete for column names inside a `query!` macro.

**Learning curve**: sqlx is the easiest if you already know SQL. You write SQL, you get results. Diesel has the steepest learning curve - the DSL is powerful but unfamiliar, and the type errors from the query builder can be walls of generic bounds. SeaORM sits in the middle - if you've used any ActiveRecord-style ORM before, the patterns are familiar.

## When to choose what

**Choose sqlx when:**

- You know SQL and want to keep writing it
- You need maximum control over the exact queries being executed
- Your team is comfortable with SQL and doesn't want an abstraction layer
- You're working with a database that has vendor-specific features (PostGIS, SQLite JSON1, MySQL full-text search) that ORMs don't expose

**Choose Diesel when:**

- Compile-time safety is your top priority and you don't want to rely on a live database during builds (after initial schema.rs generation)
- You have a complex schema with many joins and aggregations where type errors in SQL would be costly
- You need query pipelining performance (via diesel-async)
- You're okay with a synchronous-first API (or willing to add diesel-async)

**Choose SeaORM when:**

- You want an ActiveRecord-style ORM with async support out of the box
- You prefer code-based migrations over SQL files
- Your application is CRUD-heavy and you want generated entities to minimize boilerplate
- You're coming from frameworks like Django, Rails, or Laravel and want a familiar pattern

**Or choose the hybrid approach** - wrap any of them behind a repository trait, as I described in [the repository pattern post](/blog/the-repository-pattern-abstracting-data-access-in-rust/). Your handlers call `repo.find_active_by_domain("example.com")`. Whether that's implemented with `sqlx::query!`, Diesel's DSL, or SeaORM's entity API is an implementation detail hidden behind the trait.

The one thing I'd push back on is the idea that you need to pick one and stick with it forever. sqlx is a solid default. If you find yourself writing the same pagination/filtering boilerplate across 15 entities, that's the signal to evaluate Diesel or SeaORM. Start simple, add abstraction when it earns its keep.
