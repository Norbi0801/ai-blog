+++
title = "Dependency injection patterns without a framework"
date = 2025-01-07
description = "Four ways to wire dependencies in Rust - from plain struct fields to TypeId-keyed registries - without reaching for a DI framework."

[taxonomies]
tags = ["rust", "architecture", "design-patterns", "testing"]
+++

Coming from Spring or NestJS, the first thing you notice in Rust is the absence. No `@Injectable()`. No classpath scanning. No IoC container that magically resolves your dependency graph at startup. You define a service class, annotate its constructor, and the framework figures out what to pass. In Rust, there's no reflection to inspect constructor parameters, no decorator metadata to scan, no runtime proxy generation. The language doesn't support any of it.

And yet, Rust codebases wire dependencies cleanly. Often more cleanly than their Java or TypeScript counterparts, because the type system does at compile time what Spring does at runtime. The question isn't whether DI works in Rust - it's which pattern to use.

<!-- more -->

## What dependency injection actually is

Strip away the frameworks, the annotations, the XML configs. DI is one idea: a component receives its dependencies from the outside instead of creating them internally. That's it. If your function calls `Database::connect()` inside its body, it owns that decision. If it receives a `db: &Database` parameter, the caller owns the decision. The second version is dependency injection. No framework required.

In Spring, the container reads `@Autowired` annotations, matches types to beans, and calls constructors with the right arguments. In Rust, *you* call the constructor with the right arguments. The "container" is your `main` function (or your framework's startup code). The "autowiring" is the type checker refusing to compile if you pass the wrong type.

Four patterns cover almost every Rust project I've seen. They sit on a spectrum from simple-and-explicit to flexible-and-dynamic.

## Pattern 1: Constructor injection

The simplest form. Your struct takes its dependencies as constructor arguments and stores them in fields:

```rust
use std::sync::Arc;

pub struct UserService {
    db: Arc<DatabasePool>,
    mailer: Arc<EmailClient>,
    config: AppConfig,
}

impl UserService {
    pub fn new(
        db: Arc<DatabasePool>,
        mailer: Arc<EmailClient>,
        config: AppConfig,
    ) -> Self {
        Self { db, mailer, config }
    }

    pub async fn register(&self, email: String, name: String) -> Result<User, AppError> {
        let user = self.db.insert_user(&email, &name).await?;
        self.mailer.send_welcome(&email, &name).await?;
        Ok(user)
    }
}
```

Wiring happens in `main`:

```rust
#[tokio::main]
async fn main() {
    let config = AppConfig::from_env();
    let db = Arc::new(DatabasePool::connect(&config.database_url).await.unwrap());
    let mailer = Arc::new(EmailClient::new(&config.smtp_host));

    let user_service = UserService::new(
        Arc::clone(&db),
        Arc::clone(&mailer),
        config,
    );

    // pass user_service to your router, CLI handler, etc.
}
```

This is what Java developers would call "constructor injection" - except there's no container. You wrote the wiring by hand. The compiler checks that every dependency is provided and has the correct type. If `UserService` needs a `DatabasePool` and you pass an `EmailClient`, the error shows up at compile time, not as a `NoSuchBeanDefinitionException` at startup.

**Why this works for most projects:** The dependency graph of a typical web service is shallow. You have 3-5 infrastructure pieces (database, cache, HTTP client, message queue, config) and 5-15 services that consume them. Writing out the wiring for 15 constructors takes about 40 lines in `main`. That's not boilerplate - that's your application's architecture, stated explicitly in one place.

**Where it breaks down:** When service A depends on B, B depends on C, C depends on D, and D depends on A's config... the wiring order starts to matter, and `main` grows into a 200-line dependency resolution function. You can manage this with helper functions (`fn build_services(config: &AppConfig) -> Services`), but at some point you're hand-rolling a container.

## Pattern 2: Trait-based injection

Instead of depending on concrete types, accept a trait. If you read the [adapter pattern post](/blog/the-adapter-pattern-in-rust-wrapping-external-apis/), this is the same idea - define a trait for what you need, implement it for the real thing and for test mocks. The DI angle is about *how* the trait implementation gets to the consumer.

### Static dispatch (generics)

```rust
pub struct OrderService<R: OrderRepository, N: Notifier> {
    repo: R,
    notifier: N,
}

impl<R: OrderRepository, N: Notifier> OrderService<R, N> {
    pub fn new(repo: R, notifier: N) -> Self {
        Self { repo, notifier }
    }

    pub async fn place_order(&self, items: Vec<LineItem>) -> Result<Order, AppError> {
        let order = self.repo.create(items).await?;
        self.notifier.order_placed(&order).await?;
        Ok(order)
    }
}
```

The compiler monomorphizes this - you get `OrderService<PgOrderRepo, SmtpNotifier>` in production and `OrderService<InMemoryOrderRepo, NoOpNotifier>` in tests. Zero-cost dispatch. I covered the performance implications in [the strategy pattern post](/blog/the-strategy-pattern-in-rust-polymorphism-done-right/) - for anything that does real work (database queries, network calls), the dispatch mechanism doesn't matter.

The downside is type parameter proliferation. If `OrderService` depends on 4 traits, you have `OrderService<R, N, P, C>`. Every function that touches the service needs those bounds. You can mitigate with a supertrait:

```rust
pub trait OrderDeps: OrderRepository + Notifier + PaymentGateway + Cache {}
impl<T> OrderDeps for T where T: OrderRepository + Notifier + PaymentGateway + Cache {}
```

But this is getting exotic. If you find yourself writing supertrait blanket impls to tame generics, it's a signal to switch to dynamic dispatch.

### Dynamic dispatch (dyn Trait)

```rust
pub struct OrderService {
    repo: Arc<dyn OrderRepository>,
    notifier: Arc<dyn Notifier>,
}

impl OrderService {
    pub fn new(
        repo: Arc<dyn OrderRepository>,
        notifier: Arc<dyn Notifier>,
    ) -> Self {
        Self { repo, notifier }
    }
}
```

No type parameters. The cost is one vtable lookup per method call - [the strategy pattern post](/blog/the-strategy-pattern-in-rust-polymorphism-done-right/) walked through the fat pointer layout and why it's negligible for anything that isn't a tight numeric loop.

This is the workhorse pattern for service-oriented Rust code. The [repository pattern post](/blog/the-repository-pattern-abstracting-data-access-in-rust/) used `Arc<dyn DynRepository<User>>` extensively - same idea applied to data access specifically.

**When to use static vs dynamic:** Static when you're writing a library and users provide the implementation (like `HashMap<K, V, S: BuildHasher>` - you don't want to force dynamic dispatch on every library consumer). Dynamic when you're writing an application and the implementations are known at the binary level. Application code almost always wants `dyn Trait`.

## Pattern 3: The registry (TypeMap)

Sometimes you don't want every consumer to name every dependency in its constructor. You want a bag of services keyed by type: put things in, pull things out. This is how Axum's `State<T>`, Actix-web's `web::Data<T>`, and Bevy's resource system all work under the hood.

The core data structure is a `HashMap<TypeId, Box<dyn Any>>`:

```rust
use std::any::{Any, TypeId};
use std::collections::HashMap;

pub struct Registry {
    services: HashMap<TypeId, Box<dyn Any + Send + Sync>>,
}

impl Registry {
    pub fn new() -> Self {
        Self {
            services: HashMap::new(),
        }
    }

    pub fn register<T: Send + Sync + 'static>(&mut self, service: T) {
        self.services.insert(TypeId::of::<T>(), Box::new(service));
    }

    pub fn get<T: Send + Sync + 'static>(&self) -> Option<&T> {
        self.services
            .get(&TypeId::of::<T>())
            .and_then(|boxed| boxed.downcast_ref::<T>())
    }

    pub fn get_or_panic<T: Send + Sync + 'static>(&self) -> &T {
        self.get::<T>().unwrap_or_else(|| {
            panic!(
                "service not registered: {}",
                std::any::type_name::<T>()
            )
        })
    }
}
```

Usage:

```rust
let mut registry = Registry::new();
registry.register(DatabasePool::connect("postgres://...").await?);
registry.register(EmailClient::new("smtp://..."));
registry.register(AppConfig::from_env());

// Later, anywhere that has &Registry:
let db = registry.get_or_panic::<DatabasePool>();
let mailer = registry.get::<EmailClient>();
```

You call `register::<T>()` and retrieve with `get::<T>()`. The type *is* the key. No string names, no ambiguity. Will Crichton wrote about this pattern in [Types Over Strings](https://willcrichton.net/notes/types-over-strings/) - using `TypeId` as a map key is fundamentally safer than string-keyed registries because the compiler guarantees type identity.

### What happens inside TypeId

`TypeId` is a 128-bit value computed at compile time via a compiler intrinsic. The compiler hashes the full type path using SipHash-1-3 with a zeroed key. The 128-bit width gives a collision probability of roughly 1 in 2^64 under the birthday bound - for practical purposes, no two distinct types will ever produce the same `TypeId`.

The `downcast_ref::<T>()` on `dyn Any` does one comparison: `self.type_id() == TypeId::of::<T>()`. If the 128-bit values match, it performs an unsafe pointer cast from `*const dyn Any` to `*const T`. If they don't match, you get `None`. That's the full runtime cost: one 128-bit comparison and a pointer cast. No reflection, no name lookup, no hash table inside `Any` itself.

You can verify this yourself. The [`Any` trait](https://doc.rust-lang.org/std/any/trait.Any.html) has exactly one method: `fn type_id(&self) -> TypeId`. Everything else - `downcast_ref`, `downcast_mut`, `downcast` on `Box<dyn Any>` - is implemented as inherent methods on the `dyn Any` type, not as trait methods.

### The real-world version: http::Extensions

The `http` crate (which Axum, Hyper, and Tower all use) ships a production-quality TypeMap called [`Extensions`](https://docs.rs/http/latest/http/struct.Extensions.html). When you write this in an Axum handler:

```rust
async fn handler(State(db): State<DatabasePool>) -> impl IntoResponse {
    // ...
}
```

What happens underneath: Axum stores your `DatabasePool` in the request's `Extensions` (a TypeMap). The `State<T>` extractor calls `extensions.get::<T>()`, which does the `TypeId` lookup and downcast. If the type wasn't registered, you get a 500 error. This is the one trade-off: errors are at runtime, not compile time.

Actix-web's `web::Data<T>` works identically - it wraps the value in `Arc<T>`, stores it by `TypeId`, and extracts it in handlers. Bevy's ECS resources use the same mechanism for storing world-level singletons.

### Trait upcasting makes registries cleaner

Before Rust 1.86 (April 2025), storing a `Box<dyn MyService>` in a registry required the infamous `as_any` hack:

```rust
// The old way - every trait needed this boilerplate
trait MyService: Send + Sync {
    fn do_work(&self) -> String;
    fn as_any(&self) -> &dyn Any; // hack to enable downcasting
}

impl dyn MyService {
    fn downcast_ref<T: MyService + 'static>(&self) -> Option<&T> {
        self.as_any().downcast_ref::<T>()
    }
}
```

Since 1.86, [trait upcasting](https://blog.rust-lang.org/2025/04/03/Rust-1.86.0.html) is stable. If `MyService: Any`, you can coerce `&dyn MyService` to `&dyn Any` directly. No helper method, no per-trait boilerplate:

```rust
trait MyService: Any + Send + Sync {
    fn do_work(&self) -> String;
}

fn use_service(service: &dyn MyService) {
    // Direct upcast to &dyn Any - works since 1.86
    let any_ref: &dyn Any = service;
    if let Some(concrete) = any_ref.downcast_ref::<ConcreteService>() {
        println!("Got concrete: {}", concrete.name);
    }
}
```

This matters for DI containers that store trait objects and need to downcast them to specific implementations later - a common pattern when you want to register services by trait but occasionally need the concrete type for testing or introspection.

## Pattern 4: The Context object (Ctx)

Instead of injecting individual dependencies, you inject a single context that holds everything. The handler or service receives one argument - the Ctx - and pulls what it needs:

```rust
pub struct Ctx {
    registry: Registry,
}

impl Ctx {
    pub fn repo<T: Entity + 'static>(&self) -> &dyn Repository<T>
    where
        dyn Repository<T>: Any,
    {
        self.registry.get_or_panic::<Box<dyn Repository<T>>>()
    }

    pub fn service<T: Send + Sync + 'static>(&self) -> &T {
        self.registry.get_or_panic::<T>()
    }

    pub fn config(&self) -> &AppConfig {
        self.registry.get_or_panic::<AppConfig>()
    }
}
```

Usage in a handler:

```rust
async fn create_order(ctx: &Ctx, input: CreateOrderInput) -> Result<Order, AppError> {
    let order_repo = ctx.repo::<Order>();
    let product_repo = ctx.repo::<Product>();
    let mailer = ctx.service::<EmailClient>();
    let config = ctx.config();

    // validate products exist
    for item in &input.items {
        product_repo
            .find_by_id(&item.product_id)
            .await?
            .ok_or_else(|| AppError::NotFound(format!("product {}", item.product_id)))?;
    }

    let order = order_repo.create(input.into()).await?;

    if config.send_order_emails {
        mailer.send_order_confirmation(&order).await?;
    }

    Ok(order)
}
```

The Ctx pattern is the registry pattern with a typed facade on top. Instead of `registry.get::<EmailClient>()` everywhere (stringly-typed in spirit, even if type-keyed in practice), you get `ctx.service::<EmailClient>()` with domain-specific convenience methods like `ctx.repo::<Order>()`.

**The trade-off is hidden dependencies.** With constructor injection, the function signature tells you exactly what a service needs. With Ctx, every function takes `&Ctx` and you have to read the body to know which services it actually uses. This is the same trade-off as Spring's `ApplicationContext` - convenience at the cost of explicitness.

Bevy's ECS system pushes the Ctx idea to its logical extreme. Each system function declares its dependencies as parameters - `Query<&Position, With<Velocity>>`, `Res<GameConfig>`, `ResMut<Score>` - and the scheduler reads the function signature to determine what resources the system needs. The "Ctx" is the entire `World`, but the type system constrains which parts each system can access.

## Comparison

| | Constructor | Trait-based | Registry | Ctx |
|---|---|---|---|---|
| **Dependencies visible in signature** | Yes | Yes | No | No |
| **Compile-time safety** | Full | Full | Runtime panics | Runtime panics |
| **Adding a dependency** | Change constructor + all call sites | Change constructor + all call sites | Call `register()` once | Call `register()` once |
| **Type parameter proliferation** | None | Can grow (generics) or none (dyn) | None | None |
| **Testing** | Pass mocks to constructor | Pass mock impl | Register mocks in registry | Register mocks in Ctx |
| **Typical project size** | Small-medium | Medium-large | Large | Large / frameworks |

## When each pattern fits

**Constructor injection** covers 80% of cases. Your web service has a database pool, a config, maybe an HTTP client and a cache. Five dependencies, ten services. Write the constructors, wire them in `main`, done. The explicit wiring is documentation - a new developer reads `main` and understands the entire dependency graph in 30 seconds.

**Trait-based injection** is constructor injection with an abstraction layer. Use it when you need swappable implementations - the [repository pattern](/blog/the-repository-pattern-abstracting-data-access-in-rust/) for database backends, the [adapter pattern](/blog/the-adapter-pattern-in-rust-wrapping-external-apis/) for external APIs. The trait gives you the seam for testing and the boundary for future changes. When someone says "dependency injection in Rust," this is usually what they mean.

**The registry** (TypeMap) makes sense when the number of dependencies is large or dynamic. Plugin systems, where plugins register their own services. Framework internals, where the set of services isn't known until startup. If you find yourself passing 12 parameters to a constructor, a registry simplifies the wiring. But you lose compile-time guarantees - a missing registration is a runtime panic.

**The Ctx object** is the registry with a domain-aware API. Frameworks use it (Axum's `State`, Actix's `Data`, Bevy's `World`) because they need to support arbitrary user types without knowing them at framework-compile-time. If you're building a framework or a plugin-heavy application, this is the pattern. If you're building a regular service, the Ctx is probably overkill.

## The testing payoff

Regardless of which pattern you pick, the point of DI is testability. Without DI:

```rust
// Untestable - hardcoded database connection inside
pub async fn get_user(id: &str) -> Result<User, AppError> {
    let pool = DatabasePool::connect("postgres://prod:5432/mydb").await?;
    let user = sqlx::query_as("SELECT * FROM users WHERE id = $1")
        .bind(id)
        .fetch_one(&pool)
        .await?;
    Ok(user)
}
```

With DI (trait-based, dynamic dispatch):

```rust
pub async fn get_user(
    repo: &dyn UserRepository,
    id: &str,
) -> Result<User, AppError> {
    repo.find_by_id(id)
        .await?
        .ok_or_else(|| AppError::NotFound(format!("user {}", id)))
}
```

The second version is testable in isolation. No database, no network, no flaky CI. The test runs in microseconds:

```rust
#[tokio::test]
async fn returns_not_found_for_missing_user() {
    let repo = InMemoryUserRepo::new();
    let result = get_user(&repo, "nonexistent").await;
    assert!(matches!(result, Err(AppError::NotFound(_))));
}

#[tokio::test]
async fn returns_user_when_exists() {
    let repo = InMemoryUserRepo::new();
    repo.seed(User {
        id: "u1".into(),
        name: "Alice".into(),
        email: "alice@example.com".into(),
    });

    let user = get_user(&repo, "u1").await.unwrap();
    assert_eq!(user.name, "Alice");
}
```

The DI pattern doesn't just help you swap databases. It helps you test error paths that are hard to trigger with a real database - what happens when `find_by_id` returns an error? When the connection pool is exhausted? When the query times out? With an in-memory mock, you control every response:

```rust
struct FailingRepo;

#[async_trait]
impl UserRepository for FailingRepo {
    async fn find_by_id(&self, _id: &str) -> Result<Option<User>, RepoError> {
        Err(RepoError::Storage("connection refused".into()))
    }
    // ... other methods
}

#[tokio::test]
async fn handles_database_failure_gracefully() {
    let repo = FailingRepo;
    let result = get_user(&repo, "u1").await;
    assert!(matches!(result, Err(AppError::Storage(_))));
}
```

Try triggering "connection refused" reliably in an integration test against a real Postgres instance. You'd need to stop the database mid-test. With DI, you return an error from a struct. Five lines. Deterministic every time.

## The DI crate landscape

You might be wondering if there are crates that bring Spring-style DI to Rust. There are, though adoption is modest:

- [**shaku**](https://crates.io/crates/shaku) (~133K downloads) - Compile-time DI with `#[derive(Component)]` and `#[derive(Interface)]`. Has integrations for Axum, Actix, and Rocket. The most mature option.
- [**dill**](https://crates.io/crates/dill) (~48K downloads) - Runtime DI container used in the [kamu-data](https://github.com/kamu-data/kamu-cli) project, a large real-world Rust codebase using onion architecture.
- [**nject**](https://github.com/nicoscotton/nject) - Zero-cost compile-time DI with `#[injectable]` and `#[module]` macros.
- [**lockjaw**](https://github.com/nicoscotton/lockjaw) - Dagger-inspired, fully static, detects circular dependencies at compile time.

For comparison, `serde` has over 300 million downloads. `tokio` has over 200 million. The most popular DI crate has 133 thousand. That ratio tells you something: the Rust ecosystem overwhelmingly solves DI with language features (traits, generics, constructors) rather than frameworks. The patterns in this post aren't a compromise - they're the preferred approach.

[Pavex](https://github.com/LukeMathWalker/pavex), Luca Palmieri's web framework, represents an interesting frontier. It uses rustdoc JSON output as a compile-time reflection API to build dependency graphs and generate zero-cost DI wiring. No runtime TypeMap, no dynamic dispatch - the generated code calls constructors directly in the right order. It's still early, but it shows where compile-time DI in Rust could go.

## The bottom line

Don't port your Spring mental model to Rust. Spring needs a container because Java's type system can't express "this function needs a UserRepository" at compile time without annotations and reflection. Rust's type system does exactly that. The function signature *is* the dependency declaration:

```rust
async fn handler(
    State(db): State<Arc<DatabasePool>>,
    Json(body): Json<CreateUserRequest>,
) -> Result<Json<User>, AppError> {
    // ...
}
```

That Axum handler declares two dependencies: a `DatabasePool` and a `CreateUserRequest` body. The framework reads the type parameters and provides them. No scanning, no annotation processing, no proxy generation. Types in, values out.

Start with constructor injection. When you need swappable implementations, add traits. When the dependency count gets unwieldy, consider a registry. And when you're building a framework that needs to support arbitrary user types, reach for the Ctx pattern.

Each step trades compile-time safety for runtime flexibility. Stay as far left on that spectrum as your project allows.
