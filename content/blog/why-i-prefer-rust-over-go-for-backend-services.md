+++
title = "Why I prefer Rust over Go for backend services"
date = 2026-04-23
description = "A honest comparison of Rust and Go for backend work - where Rust's type system and predictable performance win, and where Go's simplicity is the right call."

[taxonomies]
tags = ["rust", "go", "backend", "architecture"]
+++

I write both Rust and Go. I've shipped production services in both. This isn't a "Go bad, Rust good" post - Go is a genuinely well-designed language that solves real problems. But when I have the choice, I reach for Rust for backend services more often than not. The reasons are specific, and they come down to things I've hit in production, not things I've read in language comparisons.

<!-- more -->

## The type system difference is not cosmetic

People often frame the Rust vs Go type system debate as "Rust has more features." That undersells it. The difference isn't quantity - it's that Rust's type system catches entire categories of bugs that Go lets through to runtime.

The core of it is algebraic types. Rust has `enum` variants that carry data, and the compiler forces you to handle every variant. Go has interfaces and type assertions, which are checked at runtime.

Here's a real scenario. You're building a payment service that processes different transaction types:

```rust
enum Transaction {
    Purchase { amount_cents: i64, merchant_id: String },
    Refund { original_tx: String, amount_cents: i64 },
    Chargeback { original_tx: String, reason: String },
}

fn process(tx: Transaction) -> Result<Receipt, PaymentError> {
    match tx {
        Transaction::Purchase { amount_cents, merchant_id } => {
            // charge the card
            Ok(Receipt::new(amount_cents))
        }
        Transaction::Refund { original_tx, amount_cents } => {
            // reverse the charge
            Ok(Receipt::refund(original_tx, amount_cents))
        }
        Transaction::Chargeback { original_tx, reason } => {
            // flag for review, notify merchant
            Ok(Receipt::chargeback(original_tx, reason))
        }
    }
}
```

If someone adds a `Transaction::Void` variant next month, every `match` in the codebase that doesn't handle it will fail to compile. Not warn - fail. The compiler finds every place in your code that needs updating.

The Go equivalent typically looks like this:

```go
type Transaction interface {
    Type() string
}

func process(tx Transaction) (*Receipt, error) {
    switch t := tx.(type) {
    case *Purchase:
        return newReceipt(t.AmountCents), nil
    case *Refund:
        return refundReceipt(t.OriginalTx, t.AmountCents), nil
    case *Chargeback:
        return chargebackReceipt(t.OriginalTx, t.Reason), nil
    default:
        return nil, fmt.Errorf("unknown transaction type: %T", tx)
    }
}
```

That `default` case is the problem. When someone adds a `Void` type, the compiler is perfectly happy. The code compiles. You find out about the missing case when a `Void` transaction hits the `default` branch in production and returns an error that nobody expected. Maybe your tests catch it. Maybe they don't.

This isn't a toy example. I've seen this exact class of bug in production Go services - a new event type gets added, most switch statements get updated, but one buried in a rarely-exercised code path doesn't. It works for weeks, then fails at 3 AM when that specific event type finally flows through that path.

If you've read my post on [phantom types](/blog/phantom-types-in-rust-compile-time-constraints-with-zero-runtime-cost/), you know Rust can go even further - encoding state machine transitions in the type system so invalid states are literally unrepresentable. Go has nothing equivalent.

## Error handling that the compiler enforces

Go's error handling pattern is famous:

```go
result, err := doSomething()
if err != nil {
    return nil, err
}
```

You see this dozens of times in any Go file. The pattern itself is fine - explicit errors, no exceptions, easy to follow. The problem is that the compiler doesn't force you to check `err`. This compiles without any warning:

```go
result, _ := doSomething()
// using result without checking if doSomething failed
```

And even more subtle:

```go
result, err := doSomething()
// forgot to check err, just kept going
useResult(result) // result might be zero-value garbage
```

Go's linters (`errcheck`, `golangci-lint`) catch some of this, but they're opt-in and imperfect. The language itself doesn't care.

Rust's `Result<T, E>` makes ignoring errors a conscious act. You can't access the success value without handling the error case first:

```rust
// This won't compile - you haven't handled the error
let value = do_something(); // value is Result<T, E>, not T

// You must explicitly handle it
let value = do_something()?; // propagate error up
// or
let value = match do_something() {
    Ok(v) => v,
    Err(e) => return Err(e.into()),
};
// or, if you truly want to ignore it (at least it's explicit)
let value = do_something().unwrap(); // panic on error - visible in code review
```

The `?` operator is worth highlighting. It replaces Go's three-line `if err != nil { return nil, err }` with a single character, while doing the same thing - propagating the error to the caller. It also automatically converts error types if you've implemented `From`, which means error propagation across module boundaries is clean without manual wrapping.

```rust
fn create_order(input: OrderInput) -> Result<Order, AppError> {
    let customer = find_customer(&input.customer_id)?;  // CustomerError -> AppError
    let inventory = check_stock(&input.items)?;          // StockError -> AppError
    let payment = charge_card(&customer, input.total)?;  // PaymentError -> AppError
    Ok(Order::new(customer, inventory, payment))
}
```

Four potential failure points, zero boilerplate. Each `?` propagates the error and converts it. The equivalent Go function has 12 extra lines of `if err != nil`.

## No null, no nil panics

Go has `nil`. It's the zero value for pointers, interfaces, maps, slices, channels, and function types. A nil pointer dereference is a runtime panic - the most common crash in Go production services, and the one you can never fully eliminate through testing because it depends on which code paths get exercised with which data.

```go
func getUser(id string) *User {
    // might return nil if user not found
    return nil
}

user := getUser("abc")
fmt.Println(user.Name) // panic: nil pointer dereference
```

Rust doesn't have null. If a value might be absent, you use `Option<T>`:

```rust
fn get_user(id: &str) -> Option<User> {
    None
}

let user = get_user("abc");
// This won't compile:
// println!("{}", user.name);

// You must handle the None case:
match get_user("abc") {
    Some(user) => println!("{}", user.name),
    None => println!("user not found"),
}

// Or use combinators:
let name = get_user("abc")
    .map(|u| u.name.clone())
    .unwrap_or_else(|| "anonymous".to_string());
```

Tony Hoare called null his "billion-dollar mistake." Rust simply doesn't have it. Every place where a value might be absent is marked in the type signature, and the compiler forces you to handle that absence. You never get a surprise nil dereference at 3 AM because `Option` makes absence explicit and unavoidable.

## Predictable latency under load

This is where the conversation shifts from type theory to production behavior. Go has a garbage collector. Rust doesn't.

Go's GC is good - genuinely good. The Go team has invested enormous effort into making it concurrent and low-pause. Go 1.24 brought [15-25% improvement in GC incremental pause times](https://medium.com/@backendbyeli/go-1-24-release-notes-all-shows-a-favouring-to-gc-runtime-improvements-fcc8609eae07), and Go 1.25 introduced the Green Tea algorithm replacing core parts of the tri-color mark-and-sweep. For most services, GC pauses are under a millisecond and you'll never notice them.

But "most services" has limits. When your service holds millions of long-lived objects in memory - a cache, a connection pool, a session store - the GC has more work to do. And that work shows up as tail latency spikes.

I covered Discord's experience in detail in [Rust in production](/blog/rust-in-production-what-companies-actually-use-it-for/). The short version: their Read States service in Go had latency spikes of 10-40ms every two minutes like clockwork, caused by the GC scanning millions of LRU cache entries. The Rust rewrite eliminated those spikes entirely - average latency dropped to microseconds, and the cache capacity went up to 8 million entries with lower memory usage.

That's not a knock on Go's GC. It's a statement about the fundamental trade-off: any tracing garbage collector must periodically walk live objects. When you have millions of them, that walk takes time. Rust doesn't have this problem because there's no GC - memory is freed deterministically when values go out of scope.

The impact on p99 latency in production is real. I've seen Go services where p50 is 2ms and p99 is 45ms, with the gap almost entirely attributable to GC pauses during high-allocation periods. The equivalent Rust service had p50 at 1.5ms and p99 at 4ms. Flat, predictable, no spikes.

For services where tail latency matters - anything user-facing with tight SLOs, anything in the request path of a latency-sensitive system - this predictability is worth a lot.

## Memory efficiency at scale

Rust services typically use 2-4x less memory than equivalent Go services. This isn't a controversial claim - it shows up consistently in benchmarks and production reports.

Recent [2026 benchmarks](https://byteiota.com/rust-vs-go-2026-backend-performance-benchmarks/) show Rust servers consuming 50-80 MB of RAM where equivalent Go services sit at 100-320 MB. The difference comes from three sources:

**No runtime overhead.** Go ships a runtime with every binary - the garbage collector, the goroutine scheduler, the memory allocator. Rust's runtime is essentially zero. A Rust binary starts with `main()` and nothing else running in the background.

**No GC headroom.** Go's garbage collector needs headroom to work efficiently. The default `GOGC=100` means Go will use roughly 2x the live heap before triggering collection. You're paying for memory you're not using because the GC needs room to breathe. Tuning `GOGC` lower reduces memory but increases GC frequency (and CPU usage). It's a trade-off you don't have in Rust.

**Precise control over layout.** In Rust, a `Vec<(u32, u32)>` is a contiguous block of packed `(u32, u32)` pairs. No indirection, no object headers, no pointer chasing. In Go, the equivalent `[]struct{ A, B uint32 }` is similar for value types, but the moment you use interfaces or pointers, you're adding heap allocations and GC pressure. Rust lets you choose the exact memory layout - and if you want to understand how far that control goes, the [flyweight pattern post](/blog/the-flyweight-pattern-sharing-data-efficiently-in-rust/) covers memory-efficient data sharing in detail.

At scale, 2-4x memory difference translates directly to infrastructure costs. If your Kubernetes cluster runs 200 Go service pods, the equivalent Rust deployment might need 60-100 pods for the same throughput. That's real money.

Cloudflare's [Pingora](https://blog.cloudflare.com/how-we-built-pingora-the-proxy-that-connects-cloudflare-to-the-internet/) - their Rust-based proxy that replaced NGINX - uses 70% less CPU and 67% less memory than the old stack at the same traffic levels. At a trillion requests per day, that's not a rounding error.

## Where Go wins and I pick it anyway

Compile times. This is Go's strongest practical advantage and it's not close. A medium Go project compiles in 1-3 seconds. The same scope in Rust takes 30 seconds to 2 minutes for a clean build. Incremental builds are better (Rust has improved compile times by roughly 30% since 2023, and `cargo check` is fast), but the gap is still large.

For services where I'm iterating rapidly on business logic - changing field names, adjusting validation rules, tweaking response formats - Go's fast feedback loop genuinely matters. When every change takes 5 seconds to verify instead of 40, you move faster. Over a day of development, that compounds.

**Goroutines are simpler than async Rust.** Go's concurrency model is beautiful. `go func()` and channels. No pinning, no `Send + Sync` bounds, no colored function problem. You can teach goroutines to a junior developer in an afternoon. Teaching async Rust takes weeks, and they'll still hit confusing compiler errors around lifetimes in async contexts.

If you've read the post on [understanding Tokio](/blog/understanding-tokio-the-rust-async-runtime-under-the-hood/), you know there's a lot happening under the hood in async Rust - work-stealing schedulers, I/O drivers, cooperative yielding. It's powerful, but it's complex. Go's goroutine scheduler is equally sophisticated under the hood, but the developer-facing API is trivially simple.

**Team velocity with mixed experience levels.** On a team of 8 developers where 2 know Rust well and 6 are learning, productivity craters for the first few months. The borrow checker teaches you to think differently about ownership and lifetimes, and that learning curve is real. On the same team writing Go, everyone is productive in a week.

This isn't a permanent gap - Rust developers become fast once they internalize ownership patterns. But the ramp-up cost is a genuine business consideration.

**Quick CRUD services.** If the service is a thin REST layer over a database with straightforward business rules, Go is usually the right call. [Gin](https://github.com/gin-gonic/gin) or [Echo](https://echo.labstack.com/) get you from zero to deployed in an afternoon. The type system guarantees that Rust provides matter less when the service is simple enough that there aren't many states to get wrong.

## My decision framework

After enough projects in both languages, my heuristic is straightforward:

**I pick Rust when:**
- The service is long-running and holds significant state in memory (caches, connection pools, session stores)
- Tail latency matters - p99 SLOs below 10ms, real-time user-facing systems
- The service processes untrusted input and sits at a security boundary
- Memory efficiency directly impacts infrastructure costs (high-density deployments, edge computing)
- The domain has complex state transitions that benefit from exhaustive type checking
- The service will run for years with infrequent changes - Rust's upfront investment in correctness pays off over a long lifecycle

**I pick Go when:**
- Rapid prototyping - I need something running this week, not this month
- The team is mostly unfamiliar with Rust and there's no time budget for learning
- The service is a straightforward API layer with simple CRUD operations
- Business logic changes frequently and fast iteration matters more than runtime guarantees
- The service is a glue layer orchestrating calls to other services with minimal local state
- I need a quick CLI tool or automation script (though Rust's [clap](https://crates.io/crates/clap) is excellent for CLIs too)

The overlap in the middle is where it gets interesting. A moderately complex backend service with some in-memory state and moderate latency requirements? Either language works fine. In that gray zone, I tend toward Rust because the type system catches bugs that I'd otherwise find through testing or production incidents. But I wouldn't argue with someone who picks Go for the same service.

## The honest trade-off

Every time I choose Rust over Go, I'm trading development speed for runtime guarantees. The first version ships slower. The compiler argues with me more. New team members take longer to become productive.

But the service runs with lower latency, uses less memory, never panics from nil dereferences, and every error path is handled. When something does go wrong, the type system has already eliminated the most common categories of bugs - null pointer issues, unhandled error cases, missed state transitions, data races.

For backend services that I'll operate for years, that trade is worth it. The debugging time saved, the 3 AM pages avoided, the infrastructure costs reduced - they compound over the service's lifetime. Rust's upfront cost is high, but the ongoing cost is low.

Go is a fine language. I use it. I recommend it. For certain services, it's the better choice. But when I'm designing a new backend service and I have the luxury of choosing, I reach for `cargo new` more often than `go mod init`. The compiler is annoying, the build times are slow, and the learning curve is steep. But every time it catches a bug at compile time that would have been a production incident in Go, I'm glad I picked it.
