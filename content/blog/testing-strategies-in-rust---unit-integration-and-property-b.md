+++
title = "Testing strategies in Rust - unit, integration, and property-based"
date = 2026-02-25
description = "A practical guide to Rust testing - from built-in unit and integration tests to property-based testing, mocking strategies, cargo-nextest, and coverage with llvm-cov."

[taxonomies]
tags = ["rust", "testing", "tools", "developer-experience"]
+++

Rust's compiler catches a lot of bugs. Ownership, lifetimes, type safety - they eliminate entire categories of defects that plague C++ or Go codebases. But the compiler can't verify that your business logic is correct. It can't confirm that your parser handles edge cases. It can't tell you that your API returns a 404 when an item doesn't exist instead of panicking. That's what tests are for.

The good news: Rust has one of the best built-in testing stories of any systems language. `cargo test` works out of the box with zero config. The bad news: most teams stop at `#[test]` functions with a few assertions and never explore what else is available. Property-based testing, structured fixtures, mocking strategies, process-per-test execution, source-based coverage - the ecosystem goes deep.

<!-- more -->

## The three built-in test types

Rust ships three kinds of tests, each with different visibility rules and use cases. Understanding the boundaries matters because it affects what you can actually test.

### Unit tests

Unit tests live inside the module they test, guarded by `#[cfg(test)]`:

```rust
pub struct Money {
    cents: i64,
    currency: &'static str,
}

impl Money {
    pub fn new(cents: i64, currency: &'static str) -> Self {
        Self { cents, currency }
    }

    pub fn add(&self, other: &Money) -> Result<Money, String> {
        if self.currency != other.currency {
            return Err(format!(
                "Cannot add {} to {}",
                self.currency, other.currency
            ));
        }
        Ok(Money::new(self.cents + other.cents, self.currency))
    }

    fn normalize_cents(&self) -> i64 {
        self.cents.abs()
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn add_same_currency() {
        let a = Money::new(100, "USD");
        let b = Money::new(250, "USD");
        let result = a.add(&b).unwrap();
        assert_eq!(result.cents, 350);
    }

    #[test]
    fn add_different_currency_fails() {
        let usd = Money::new(100, "USD");
        let eur = Money::new(200, "EUR");
        assert!(usd.add(&eur).is_err());
    }

    #[test]
    fn normalize_is_always_positive() {
        let negative = Money::new(-500, "USD");
        assert_eq!(negative.normalize_cents(), 500);
    }
}
```

The `super::*` import is the key detail. Because unit tests are a nested module inside the source file, they can access private functions and fields. That `normalize_cents` method is `fn`, not `pub fn` - external code can't call it, but the test module can. This is deliberate. Unit tests are meant to test implementation details.

The `#[cfg(test)]` attribute strips the entire module from release builds. No binary bloat, no dead code warnings from test helpers.

One thing that trips people up: each `#[test]` function runs in its own thread by default, but they share the same process. If a test panics, `cargo test`'s harness catches it. If a test segfaults (unlikely in safe Rust, possible in unsafe code), it takes down the entire test binary. We'll see later how `cargo-nextest` solves this with process isolation.

### Integration tests

Integration tests live in the `tests/` directory at your crate root, next to `src/`:

```
my_crate/
  src/
    lib.rs
    parser.rs
  tests/
    parse_full_document.rs
    error_recovery.rs
```

Each file in `tests/` compiles as a separate crate that depends on your library. This means integration tests can only use your public API - exactly what external consumers would see:

```rust
// tests/parse_full_document.rs
use my_crate::Parser;

#[test]
fn parse_nested_structure() {
    let input = r#"
        [section]
        key = "value"
        nested.key = 42
    "#;

    let doc = Parser::new().parse(input).unwrap();
    assert_eq!(doc.get("section.key"), Some(&Value::String("value".into())));
    assert_eq!(doc.get("section.nested.key"), Some(&Value::Integer(42)));
}
```

Integration tests catch a different class of bugs than unit tests. Your internal modules might work fine individually, but the public API might wire them together incorrectly. Maybe your parser module is correct but `lib.rs` re-exports the wrong constructor. Integration tests catch that.

A practical note on organization: if you have shared helpers for integration tests, put them in `tests/common/mod.rs`. Using `tests/common.rs` instead will make `cargo test` treat it as its own test suite (producing a "0 tests" binary). The `mod.rs` convention tells Cargo it's a module, not a test file.

```
tests/
  common/
    mod.rs        # shared setup functions
  api_tests.rs    # uses `mod common;`
  db_tests.rs     # uses `mod common;`
```

### Doc tests

Doc tests are the third kind, and arguably the most underrated. Any Rust code block in a doc comment gets compiled and executed during `cargo test`:

```rust
/// Splits a string into words, filtering out empty segments.
///
/// ```
/// use my_crate::split_words;
///
/// let words = split_words("hello  world  ");
/// assert_eq!(words, vec!["hello", "world"]);
/// ```
///
/// Handles empty input:
///
/// ```
/// use my_crate::split_words;
///
/// let words = split_words("");
/// assert!(words.is_empty());
/// ```
pub fn split_words(input: &str) -> Vec<&str> {
    input.split_whitespace().collect()
}
```

Doc tests serve double duty: they're documentation and regression tests simultaneously. When your API changes and the example breaks, `cargo test` catches it. No more stale examples in your README.

A few doc test tricks worth knowing. Lines starting with `#` are compiled but hidden from the rendered docs - useful for imports and setup code that would clutter the example:

```rust
/// ```
/// # use std::collections::HashMap;
/// # let mut map = HashMap::new();
/// # map.insert("key", 42);
/// // This is the part the reader sees:
/// assert_eq!(map["key"], 42);
/// ```
```

Mark code blocks that should compile but not run with `no_run`. Mark code that should fail to compile with `compile_fail`. Mark code that's not Rust with `text` or `ignore`:

```rust
/// ```no_run
/// // This would block forever, so we don't run it in tests
/// let listener = std::net::TcpListener::bind("0.0.0.0:8080").unwrap();
/// for stream in listener.incoming() { /* ... */ }
/// ```
///
/// ```compile_fail
/// let x: i32 = "not a number"; // proves this doesn't compile
/// ```
```

Doc tests are slower than unit tests because each one compiles as its own mini-binary. For a crate with hundreds of doc examples, this adds up. But the guarantee that your docs stay correct is worth it.

## Property-based testing

Writing individual test cases works until it doesn't. You pick a few inputs, check the output, and call it done. But you're testing the cases you thought of, which means you're missing the cases you didn't. Property-based testing flips this: you describe *properties* that should hold for all inputs, and the framework generates hundreds of random inputs to verify them.

Two crates dominate this space: [proptest](https://github.com/proptest-rs/proptest) (75M+ downloads, currently at v1.10.0) and [quickcheck](https://github.com/BurntSushi/quickcheck) (v1.0.3, by BurntSushi). Both work. proptest is more powerful for complex scenarios; quickcheck is simpler when you just need the basics.

### proptest

proptest's core idea is *strategies* - objects that describe how to generate values. You compose them to build complex inputs:

```rust
use proptest::prelude::*;

fn reverse<T: Clone>(xs: &[T]) -> Vec<T> {
    xs.iter().rev().cloned().collect()
}

proptest! {
    #[test]
    fn reverse_twice_is_identity(ref xs in prop::collection::vec(any::<i32>(), 0..100)) {
        let reversed_twice = reverse(&reverse(xs));
        prop_assert_eq!(&reversed_twice, xs);
    }

    #[test]
    fn reverse_preserves_length(ref xs in prop::collection::vec(any::<String>(), 0..50)) {
        prop_assert_eq!(reverse(xs).len(), xs.len());
    }
}
```

When a test fails, proptest *shrinks* the input to the smallest value that still triggers the failure. If your sort function breaks on a 47-element vector, proptest will try to reduce it to a 2 or 3-element case that demonstrates the same bug. This is where proptest really shines over quickcheck - shrinking is strategy-aware. If your strategy says "generate integers between 10 and 100," proptest won't shrink below 10. quickcheck's shrinking is type-level, so it might shrink to values that violate your constraints, producing confusing test rejections.

Custom strategies let you generate domain-specific values:

```rust
use proptest::prelude::*;

#[derive(Debug, Clone, PartialEq)]
struct Email {
    local: String,
    domain: String,
}

fn email_strategy() -> impl Strategy<Value = Email> {
    (
        "[a-z][a-z0-9_.]{1,20}",        // local part
        "[a-z]{2,10}\\.[a-z]{2,4}",      // domain
    )
        .prop_map(|(local, domain)| Email { local, domain })
}

proptest! {
    #[test]
    fn email_local_part_is_never_empty(email in email_strategy()) {
        prop_assert!(!email.local.is_empty());
    }
}
```

### quickcheck

quickcheck takes a different approach. Instead of explicit strategies, you implement the `Arbitrary` trait for your types:

```rust
use quickcheck::{quickcheck, Arbitrary, Gen};

#[derive(Debug, Clone)]
struct Positive(u32);

impl Arbitrary for Positive {
    fn arbitrary(g: &mut Gen) -> Self {
        Positive(u32::arbitrary(g).max(1))
    }
}

quickcheck! {
    fn division_is_inverse_of_multiplication(a: Positive, b: Positive) -> bool {
        let product = a.0 as u64 * b.0 as u64;
        product / a.0 as u64 == b.0 as u64
    }
}
```

quickcheck is simpler - fewer concepts, less API surface. If your properties involve standard types (integers, strings, vectors), it's a fast way to get property tests running. For anything involving constrained domains (valid emails, balanced trees, state machines), proptest's strategy system is significantly more ergonomic.

### What to property-test

Property-based testing isn't about replacing example-based tests. It's about finding the cases your examples miss. Good candidates:

- **Encode/decode roundtrips**: `decode(encode(x)) == x` for all `x`
- **Invariant preservation**: sorting produces sorted output, a balanced tree stays balanced after insertion
- **Idempotency**: applying an operation twice gives the same result as once
- **Commutativity**: `a + b == b + a`, `merge(x, y) == merge(y, x)`
- **No panics**: the function handles all inputs without crashing

## Mocking strategies

Testing code that talks to databases, APIs, or the filesystem requires some way to replace those dependencies with controlled substitutes. Rust offers three approaches, each with trade-offs.

### Trait-based testing (hand-written mocks)

If you followed the patterns from [dependency injection without a framework](/blog/dependency-injection-patterns-without-a-framework), your code already takes dependencies as trait objects. Testing is just providing a different implementation:

```rust
#[async_trait::async_trait]
pub trait UserStore {
    async fn find_by_id(&self, id: &str) -> Result<Option<User>, DbError>;
    async fn save(&self, user: &User) -> Result<(), DbError>;
}

// Production implementation
pub struct PostgresUserStore { /* ... */ }

#[async_trait::async_trait]
impl UserStore for PostgresUserStore {
    async fn find_by_id(&self, id: &str) -> Result<Option<User>, DbError> {
        // actual database query
        todo!()
    }
    async fn save(&self, user: &User) -> Result<(), DbError> {
        todo!()
    }
}

// Test implementation - no macros, no frameworks
struct FakeUserStore {
    users: std::sync::Mutex<Vec<User>>,
}

impl FakeUserStore {
    fn new() -> Self {
        Self {
            users: std::sync::Mutex::new(Vec::new()),
        }
    }

    fn with_users(users: Vec<User>) -> Self {
        Self {
            users: std::sync::Mutex::new(users),
        }
    }
}

#[async_trait::async_trait]
impl UserStore for FakeUserStore {
    async fn find_by_id(&self, id: &str) -> Result<Option<User>, DbError> {
        Ok(self.users.lock().unwrap().iter().find(|u| u.id == id).cloned())
    }

    async fn save(&self, user: &User) -> Result<(), DbError> {
        self.users.lock().unwrap().push(user.clone());
        Ok(())
    }
}

#[tokio::test]
async fn registration_saves_user() {
    let store = FakeUserStore::new();
    let service = RegistrationService::new(Box::new(store));

    service.register("alice@example.com", "Alice").await.unwrap();

    // ... assert user was saved
}
```

This approach is verbose but completely transparent. No macros to debug, no hidden behavior, no compile-time overhead from proc macros. The fake implementation is just code you can step through in a debugger. For traits with 2-3 methods, hand-written mocks are often the right choice.

This connects directly to the [adapter pattern](/blog/the-adapter-pattern-in-rust---wrapping-external-apis/) - if you've wrapped an external API behind a trait, you already have the boundary where a fake implementation slots in.

### mockall - generated mocks

When your trait has 10+ methods and you only care about 2 of them in a given test, hand-writing fakes gets tedious. [mockall](https://github.com/asomers/mockall) (v0.13.1, 107M+ downloads) generates mock structs from traits using proc macros:

```rust
use mockall::automock;

#[automock]
#[async_trait::async_trait]
pub trait PaymentGateway {
    async fn charge(&self, amount: i64, token: &str) -> Result<String, PaymentError>;
    async fn refund(&self, charge_id: &str) -> Result<(), PaymentError>;
    async fn get_balance(&self) -> Result<i64, PaymentError>;
}

#[tokio::test]
async fn checkout_charges_correct_amount() {
    let mut mock = MockPaymentGateway::new();

    mock.expect_charge()
        .with(mockall::predicate::eq(2500), mockall::predicate::always())
        .times(1)
        .returning(|_, _| Ok("ch_123".to_string()));

    let service = CheckoutService::new(Box::new(mock));
    let result = service.checkout(cart_with_total(2500)).await;

    assert!(result.is_ok());
}
```

mockall handles expectations (how many times a method should be called), argument matchers, return value sequences, and call ordering. It is powerful. But it has sharp edges:

- `#[automock]` must appear *before* `#[async_trait]` on the trait definition
- Generic methods with non-`'static` type parameters can't be mocked (they can't be downcast internally)
- `impl Trait` return types get silently boxed (allocation overhead)
- Traits with multiple `impl` blocks require the `mock!` macro instead of `#[automock]`

The biggest risk with mockall is over-specification. If your test asserts that `charge` is called exactly once with exactly these arguments in this order, you've coupled the test to the implementation sequence. Refactor the internals and the test breaks, even though behavior is identical.

### The middle ground: trait-based with generics

You can avoid both the verbosity of hand-written mocks and the complexity of mockall by making your functions generic over the trait:

```rust
async fn process_order<S: UserStore, P: PaymentGateway>(
    store: &S,
    payment: &P,
    order: Order,
) -> Result<Receipt, OrderError> {
    let user = store
        .find_by_id(&order.user_id)
        .await?
        .ok_or(OrderError::UserNotFound)?;
    let charge_id = payment.charge(order.total, &user.payment_token).await?;
    Ok(Receipt { charge_id, order })
}
```

In tests, you pass simple structs that implement the trait. In production, you pass the real implementations. No `dyn`, no boxing, no virtual dispatch overhead in production builds. The compiler monomorphizes each variant separately.

## Test fixtures with rstest

[rstest](https://github.com/la10736/rstest) (v0.26.1) brings two things Rust's built-in test harness lacks: reusable fixtures and parametrized tests.

Fixtures are functions marked with `#[fixture]` that produce test data or initialized state:

```rust
use rstest::*;

#[fixture]
fn db() -> TestDatabase {
    TestDatabase::new_in_memory()
}

#[fixture]
fn seeded_db(db: TestDatabase) -> TestDatabase {
    db.insert_user("alice", "alice@test.com");
    db.insert_user("bob", "bob@test.com");
    db
}

#[rstest]
fn find_existing_user(seeded_db: TestDatabase) {
    let user = seeded_db.find_user("alice");
    assert!(user.is_some());
}

#[rstest]
fn count_users(seeded_db: TestDatabase) {
    assert_eq!(seeded_db.user_count(), 2);
}
```

Fixtures compose - `seeded_db` depends on `db`, and rstest resolves the chain automatically. If you add `#[once]` to a fixture, it runs once and returns a `&'static` reference shared across all tests that use it. Useful for expensive setup like spawning a test server.

Parametrized tests generate one test case per argument set:

```rust
#[rstest]
#[case("hello", 5)]
#[case("", 0)]
#[case("rust testing", 12)]
#[case("  spaces  ", 10)]
fn string_length(#[case] input: &str, #[case] expected: usize) {
    assert_eq!(input.len(), expected);
}
```

This generates four independent test functions (`string_length::case_1` through `case_4`), each showing up separately in test output. Much cleaner than four copy-pasted test functions that differ by one line.

## cargo-nextest: a better test runner

[cargo-nextest](https://nexte.st/) (v0.9.132) is a drop-in replacement for `cargo test` that changes one fundamental thing: each test runs in its own process.

Why does this matter? With `cargo test`, all tests in a binary share a process. A test that corrupts global state (environment variables, static mutables, current directory) affects every subsequent test. A test that segfaults kills the entire suite. With nextest, each test is isolated. A segfault in one test means one test fails - the rest keep running.

The speed difference is significant. nextest parallelizes across all tests from all binaries simultaneously, while `cargo test` only parallelizes within a single binary, running binaries sequentially. Official benchmarks on real projects:

| Project | Tests | Speedup |
|---------|-------|---------|
| tokio | 1,138 | 2.09x |
| crucible | 483 | 3.38x |
| reqwest | 113 | 2.48x |
| meilisearch | 721 | 1.96x |

Install and use:

```bash
# Install
cargo install cargo-nextest --locked

# Run all tests
cargo nextest run

# Run with a filter
cargo nextest run -- test_name_pattern

# Retry flaky tests automatically (up to 2 retries)
cargo nextest run --retries 2
```

Flaky test detection is a standout feature. If a test fails on the first run but passes on retry, nextest marks it as flaky in the output rather than silently passing. This surfaces intermittent issues that `cargo test` hides.

One limitation: nextest doesn't run doc tests (yet). You'll still need `cargo test --doc` for those.

## Coverage with cargo-llvm-cov

[cargo-llvm-cov](https://github.com/taiki-e/cargo-llvm-cov) (v0.8.5) uses LLVM's source-based code coverage instrumentation. Unlike region-based tools, it instruments at the LLVM IR level, giving you line, region, and (on nightly) branch coverage with high accuracy.

```bash
# Install
cargo install cargo-llvm-cov --locked

# Terminal summary
cargo llvm-cov

# HTML report - opens in browser
cargo llvm-cov --open

# Use with nextest
cargo llvm-cov nextest

# LCOV format for CI (Codecov, Coveralls)
cargo llvm-cov --lcov --output-path lcov.info

# Only measure coverage for your crate, not dependencies
cargo llvm-cov --no-default-features
```

The HTML report is the most useful. It shows each source file with line-by-line highlighting - green for covered, red for uncovered. You can quickly spot untested branches.

A typical CI setup:

```yaml
# .github/workflows/coverage.yml
- name: Install cargo-llvm-cov
  uses: taiki-e/install-action@cargo-llvm-cov

- name: Generate coverage
  run: cargo llvm-cov --lcov --output-path lcov.info

- name: Upload to Codecov
  uses: codecov/codecov-action@v4
  with:
    files: lcov.info
```

Coverage numbers are a useful signal but a terrible goal. 90% line coverage doesn't mean your code is well-tested - it means 90% of lines executed during tests, which says nothing about whether the assertions were meaningful. A test that calls every function but never checks return values gives you high coverage and zero confidence. Use coverage to find code you forgot to test, not as a metric to maximize.

## What to test, what not to test

After setting up all these tools, you still need to decide where to spend your testing effort. Not all code benefits equally from tests.

**Worth testing thoroughly:**

- **Parsing and serialization logic** - edge cases are abundant, property tests shine here
- **Business rules and domain logic** - the "if this then that" code where bugs cost money
- **Error paths** - the `Err` branches, the `None` cases, the malformed input handling
- **State machines** - valid transitions, invalid transitions, edge states
- **Public API contracts** - if external users depend on it, it needs integration tests

**Probably not worth unit testing:**

- **Trivial getters/setters** - a function that returns `self.name.clone()` doesn't need a test
- **Direct delegation** - if your method just calls another method with the same arguments, the test adds no value
- **Framework glue code** - route registrations, middleware wiring, DI setup. These are better covered by a single integration test that boots the app and hits an endpoint
- **Type-system-enforced properties** - if the compiler prevents the bug, a test is redundant

**The gray area:**

- **Database queries** - unit testing with a fake store catches logic bugs but misses SQL issues. An integration test with a real (test) database catches both but is slower. Most teams do both
- **External API calls** - the [adapter pattern](/blog/the-adapter-pattern-in-rust---wrapping-external-apis/) lets you test your logic with fakes while a few integration tests verify the real adapter against a sandbox
- **Concurrent code** - property-based tests with concurrent strategies can surface race conditions, but some timing-dependent bugs require specialized tools like loom

A reasonable starting point: unit tests for logic, integration tests for boundaries, property tests for parsers and encoders, and coverage reports to find the blind spots. Add more where bugs actually show up - your bug tracker is the best guide to where tests are missing.
