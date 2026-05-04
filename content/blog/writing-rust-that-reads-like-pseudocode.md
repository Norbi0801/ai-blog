+++
title = "Writing Rust that reads like pseudocode"
date = 2025-05-18
description = "How method chaining, let-else, if-let chains, pattern matching, and the ? operator can turn verbose Rust into code that reads almost like pseudocode - and when you should keep the verbosity."

[taxonomies]
tags = ["rust", "code-style", "readability", "ergonomics"]
+++

Rust has a reputation for verbosity. Lifetime annotations, explicit error handling, pattern matching on every variant, type conversions everywhere. Newcomers look at a 30-line function and think: this would be 8 lines in Python.

They're not wrong about the line count. But they're wrong about the conclusion. Rust has accumulated a surprisingly rich set of features that let you write code so terse and expressive it reads like pseudocode. The catch is that most of these features were stabilized across different releases over the past four years, so a lot of tutorials and Stack Overflow answers still show the verbose style. People learn the hard way first and never discover the shortcuts.

This post is about those shortcuts. Not toy examples - real patterns you use daily that collapse 15-line blocks into 2-line expressions without sacrificing clarity. And just as important: the cases where the verbose version is actually better.

<!-- more -->

## The ? operator - errors as control flow

If you've covered the [From and Into traits](/blog/from-and-into-traits---rusts-conversion-magic/), you already know that `?` is built on top of `From`. But let's look at it purely from a readability angle.

The verbose version, circa Rust 1.0:

```rust
fn load_config(path: &str) -> Result<Config, AppError> {
    let contents = match std::fs::read_to_string(path) {
        Ok(c) => c,
        Err(e) => return Err(AppError::from(e)),
    };
    let parsed = match toml::from_str(&contents) {
        Ok(p) => p,
        Err(e) => return Err(AppError::from(e)),
    };
    Ok(parsed)
}
```

Ten lines. Six of them are boilerplate that says "if this failed, convert the error and bail." The actual logic - read a file, parse TOML - is buried.

With `?`:

```rust
fn load_config(path: &str) -> Result<Config, AppError> {
    let parsed = toml::from_str(&std::fs::read_to_string(path)?)?;
    Ok(parsed)
}
```

Three lines. The logic reads top-to-bottom: read the file, parse it, return it. The error conversion is implicit (via your `impl From<io::Error> for AppError` and `impl From<toml::de::Error> for AppError`). The control flow - "bail on failure" - is a single character.

You can even go further:

```rust
fn load_config(path: &str) -> Result<Config, AppError> {
    Ok(toml::from_str(&std::fs::read_to_string(path)?)?)
}
```

One expression. Whether this is "better" depends on your team. Some people find nested `?` hard to scan. I find it perfectly clear when there are only two or three operations. Past that, I break it into `let` bindings with meaningful names.

The point isn't to play code golf. The point is that `?` turns error handling from structural noise into punctuation. You read the happy path. The `?` tells you "this can fail, and if it does, we leave." That's pseudocode-level clarity.

### What ? compiles to

If you're curious whether `?` adds overhead, check [Compiler Explorer](https://rust.godbolt.org/). The generated assembly for `?` is identical to the manual `match` version. The compiler sees through the `From` conversion and the `Try` trait implementation and emits the same branching code. This is a true zero-cost abstraction - syntactic sugar that disappears at the machine level.

## let-else - the early return pattern

Stabilized in [Rust 1.65](https://blog.rust-lang.org/2022/11/03/Rust-1.65.0.html) via [RFC 3137](https://rust-lang.github.io/rfcs/3137-let-else.html), `let-else` handles a pattern that shows up constantly: "extract this value or bail."

Before `let-else`, you'd write:

```rust
fn process_user(users: &HashMap<String, User>, id: &str) -> Result<(), AppError> {
    let user = match users.get(id) {
        Some(u) => u,
        None => return Err(AppError::NotFound(format!("user {id}"))),
    };
    
    let email = match &user.email {
        Some(e) => e,
        None => return Err(AppError::Validation("email required".into())),
    };
    
    let domain = match email.split('@').nth(1) {
        Some(d) => d,
        None => return Err(AppError::Validation("invalid email format".into())),
    };
    
    println!("Processing user {} with domain {}", user.name, domain);
    Ok(())
}
```

Eighteen lines. The structure is repetitive: try something, unwrap it, or return an error. The actual logic - get user, get email, extract domain - is scattered across match arms.

With `let-else`:

```rust
fn process_user(users: &HashMap<String, User>, id: &str) -> Result<(), AppError> {
    let Some(user) = users.get(id) else {
        return Err(AppError::NotFound(format!("user {id}")));
    };
    let Some(email) = &user.email else {
        return Err(AppError::Validation("email required".into()));
    };
    let Some(domain) = email.split('@').nth(1) else {
        return Err(AppError::Validation("invalid email format".into()));
    };
    
    println!("Processing user {} with domain {}", user.name, domain);
    Ok(())
}
```

Same logic, but the structure is flat. No nesting. Each statement reads as: "let this be X, or else bail." The binding (`user`, `email`, `domain`) enters scope for the rest of the function. The else block must diverge - `return`, `break`, `continue`, or `panic!`.

Read that code aloud: "Let some user equal users-get-id, else return not-found. Let some email equal user-email, else return validation error." That is pseudocode.

### Why the else block must diverge

The compiler enforces that the `else` block never falls through. This isn't arbitrary - it's necessary for soundness. If the pattern doesn't match, the binding on the left side has no value. Allowing the `else` block to complete normally would mean `user` is uninitialized after it. Rust doesn't have null, so the only options are: give it a value (use regular `if let`), or leave the current scope (`return`, `break`, `continue`, `panic!`).

This is the same reasoning behind [RFC 3137](https://rust-lang.github.io/rfcs/3137-let-else.html). The RFC discussion also considered allowing `let ... else { default_value }` syntax but rejected it - that's what `unwrap_or` and `if let` are for.

## if-let chains - flattening nested conditions

This one took years. [RFC 2497](https://rust-lang.github.io/rfcs/2497-if-let-chains.html) was proposed in 2018 and finally stabilized in [Rust 1.88](https://blog.rust-lang.org/2025/06/26/Rust-1.88.0/) in June 2025, requiring the 2024 edition.

The problem it solves:

```rust
fn authorize_action(
    session: &Option<Session>,
    action: &str,
    permissions: &HashMap<String, Vec<String>>,
) -> bool {
    if let Some(session) = session {
        if session.is_active() {
            if let Some(perms) = permissions.get(&session.user_id) {
                if perms.contains(&action.to_string()) {
                    return true;
                }
            }
        }
    }
    false
}
```

Four levels of nesting. The "rightward drift" makes it hard to see what the function actually checks. You have to mentally track which indentation level you're at.

With if-let chains:

```rust
fn authorize_action(
    session: &Option<Session>,
    action: &str,
    permissions: &HashMap<String, Vec<String>>,
) -> bool {
    if let Some(session) = session
        && session.is_active()
        && let Some(perms) = permissions.get(&session.user_id)
        && perms.contains(&action.to_string())
    {
        return true;
    }
    false
}
```

Flat. One `if` with four conditions chained by `&&`. You can mix `let` patterns and boolean expressions freely. Each condition can use bindings from previous ones - `session` from the first `let` is available in `session.is_active()`, and `perms` from the third is available in the fourth.

Read it as: "if there's a session AND it's active AND the user has permissions AND those permissions include this action, allow it." That sentence structure maps directly to the code.

### Why it needed the 2024 edition

This feature has a subtle interaction with temporary lifetimes. In the 2021 edition, temporaries created in `if let` conditions had their scope extended to the end of the `if` block. With chained conditions, this could lead to surprising drop order - a temporary from the first condition would outlive a temporary from the third condition even though the third was created later. The [2024 edition changed temporary scoping rules](https://doc.rust-lang.org/edition-guide/rust-2024/temporary-tail-expr-scope.html) to make drop order consistent, which was a prerequisite for landing if-let chains without footguns.

## Method chaining - iterator pipelines as sentences

Rust iterators are lazy. Nothing happens until you call a consuming method like `collect()`, `sum()`, `count()`, or `for_each()`. This lets you build pipelines that describe transformations step by step.

The imperative version:

```rust
fn active_admin_emails(users: &[User]) -> Vec<String> {
    let mut result = Vec::new();
    for user in users {
        if user.role == Role::Admin {
            if user.is_active {
                if let Some(ref email) = user.email {
                    result.push(email.to_lowercase());
                }
            }
        }
    }
    result.sort();
    result.dedup();
    result
}
```

Twelve lines. Nested conditionals. Mutable state. You have to read the whole thing to understand what the output looks like.

The iterator version:

```rust
fn active_admin_emails(users: &[User]) -> Vec<String> {
    users.iter()
        .filter(|u| u.role == Role::Admin && u.is_active)
        .filter_map(|u| u.email.as_ref())
        .map(|e| e.to_lowercase())
        .sorted()
        .dedup()
        .collect()
}
```

Seven lines, no nesting, no mutable state. Each line is a verb: filter, extract, transform, sort, deduplicate, collect. The pipeline reads like a description of the result.

`sorted()` and `dedup()` here come from the [itertools crate](https://docs.rs/itertools/latest/itertools/trait.Itertools.html) (the standard library's `Iterator` has `dedup` only on consecutive elements via a slightly different pattern). Add `use itertools::Itertools;` and you get dozens of extra combinators: `unique()`, `chunk_by()`, `tuple_windows()`, `intersperse()`, `join()`.

### When chains get too long

There is a point where a chain stops being readable. I usually draw the line at around 7-8 combinators. Past that, extract intermediate steps into named `let` bindings:

```rust
fn process_orders(orders: &[Order]) -> Summary {
    let valid_orders: Vec<_> = orders.iter()
        .filter(|o| o.status != Status::Cancelled)
        .filter(|o| o.total_cents > 0)
        .collect();

    let revenue: i64 = valid_orders.iter()
        .map(|o| o.total_cents)
        .sum();

    let top_products: Vec<String> = valid_orders.iter()
        .flat_map(|o| o.items.iter())
        .counts_by(|item| item.product_id.clone())
        .into_iter()
        .sorted_by_key(|&(_, count)| std::cmp::Reverse(count))
        .take(10)
        .map(|(id, _)| id)
        .collect();

    Summary { order_count: valid_orders.len(), revenue, top_products }
}
```

The names `valid_orders`, `revenue`, `top_products` are doing real work here. They tell the reader what each pipeline produces without having to trace through the combinators. This is verbose, and it's better that way.

### Rayon: parallelism as a one-word change

One of the best demonstrations of how method chains read like pseudocode is [Rayon](https://docs.rs/rayon/latest/rayon/). You swap `.iter()` for `.par_iter()` and the entire pipeline runs in parallel:

```rust
use rayon::prelude::*;

let total: i64 = records.par_iter()
    .filter(|r| r.region == "EU")
    .map(|r| r.amount)
    .sum();
```

Same chain. One word changed. The code still reads identically, but now it's work-stealing across all CPU cores. The [Rayon docs](https://docs.rs/rayon/latest/rayon/iter/trait.ParallelIterator.html) explain the work-stealing scheduler, but from a readability standpoint the beauty is that parallelism doesn't change the shape of the code at all.

## Pattern matching - exhaustive case analysis

If you've read [the post on Rust enums](/blog/rust-enums-are-not-what-you-think-algebraic-data-types-explained), you know that `match` with sum types is how Rust replaces `instanceof` chains, null checks, and type switches. But there are specific patterns that push match expressions toward pseudocode territory.

### Destructuring in match arms

```rust
fn describe_event(event: &Event) -> String {
    match event {
        Event::Click { x, y, button: MouseButton::Left } => {
            format!("left click at ({x}, {y})")
        }
        Event::Click { button: MouseButton::Right, .. } => {
            "right click (position ignored)".into()
        }
        Event::KeyPress { key, modifiers } if modifiers.contains(&Modifier::Ctrl) => {
            format!("ctrl+{key}")
        }
        Event::KeyPress { key, .. } => format!("key: {key}"),
        Event::Resize { width, height } => format!("resize to {width}x{height}"),
    }
}
```

Every arm is a sentence: "If it's a left click at (x, y), say left click at (x, y)." The destructuring pulls values directly into scope. The `..` means "I don't care about the other fields." Match guards (`if modifiers.contains(...)`) add conditions without nesting. And the compiler ensures every variant is covered.

### Matches! macro for boolean checks

When you just need "does this match a pattern?" and don't need to extract values:

```rust
// Before
let is_error = match &response.status {
    Status::ServerError(_) => true,
    Status::ClientError(_) => true,
    _ => false,
};

// After
let is_error = matches!(response.status, Status::ServerError(_) | Status::ClientError(_));
```

One line. `matches!` was stabilized in [Rust 1.42](https://blog.rust-lang.org/2020/03/12/Rust-1.42.html) and it's one of those small things that eliminates an annoying amount of boilerplate. It also supports guards:

```rust
let is_critical = matches!(event, Event::Error { severity, .. } if severity > 9);
```

## Naming - the part nobody wants to talk about

All the syntax tricks in the world don't help if your names are bad. This is language-agnostic, but Rust has some conventions worth calling out.

### Follow the std patterns

The standard library is remarkably consistent with naming. These patterns are worth internalizing:

| Convention | Examples | Meaning |
|---|---|---|
| `into_*` | `into_bytes()`, `into_inner()` | Consumes self, returns owned value |
| `as_*` | `as_str()`, `as_ref()`, `as_bytes()` | Cheap reference conversion, borrows |
| `to_*` | `to_string()`, `to_lowercase()` | Expensive conversion, allocates |
| `try_*` | `try_from()`, `try_recv()` | Can fail, returns Result |
| `*_mut` | `iter_mut()`, `get_mut()` | Mutable variant of an existing method |
| `with_*` | `with_capacity()` | Constructor variant with config |
| `is_*` / `has_*` | `is_empty()`, `has_remaining()` | Returns bool |

These conventions are documented in the [Rust API Guidelines](https://rust-lang.github.io/api-guidelines/naming.html). When your code follows them, readers don't have to check the signature - the name tells them whether the call moves, borrows, allocates, or can fail. That's pseudocode-level intent.

### Name intermediate bindings for what they represent

Bad:

```rust
let x = users.iter().filter(|u| u.is_active).collect::<Vec<_>>();
let y = x.len();
```

Good:

```rust
let active_users: Vec<_> = users.iter().filter(|u| u.is_active).collect();
let active_count = active_users.len();
```

The second version doesn't need comments. The names are the comments.

### One-letter closures are fine in short chains

This might be controversial. In a 2-combinator chain, `|u|` is perfectly clear:

```rust
users.iter().filter(|u| u.is_active).count()
```

Nobody is confused about what `u` is. The context makes it obvious. But in a 6-combinator chain where `u` transforms into different shapes, use descriptive names:

```rust
orders.iter()
    .filter(|order| order.is_fulfilled())
    .flat_map(|order| order.line_items.iter())
    .filter(|item| item.needs_restock())
    .map(|item| item.product_id.clone())
    .collect::<HashSet<_>>()
```

The shift from `order` to `item` signals that the type changed after `flat_map`. That's information a single-letter variable would hide.

## Combining features - where it all comes together

The real power shows up when you combine these features. Here's a function that handles an HTTP request to update a user's profile:

**The verbose version:**

```rust
fn handle_update_profile(
    db: &Database,
    session: &Option<Session>,
    request: &UpdateProfileRequest,
) -> Result<ProfileResponse, ApiError> {
    // Check session
    let session = match session {
        Some(s) => s,
        None => return Err(ApiError::Unauthorized),
    };
    if !session.is_active() {
        return Err(ApiError::Unauthorized);
    }
    
    // Load user
    let user = match db.find_user(&session.user_id) {
        Ok(Some(u)) => u,
        Ok(None) => return Err(ApiError::NotFound("user".into())),
        Err(e) => return Err(ApiError::Internal(e.to_string())),
    };
    
    // Validate email if provided
    let validated_email = match &request.email {
        Some(email) => {
            if !email.contains('@') {
                return Err(ApiError::Validation("invalid email".into()));
            }
            Some(email.clone())
        }
        None => None,
    };
    
    // Build update
    let update = UserUpdate {
        name: request.name.clone(),
        email: validated_email,
    };
    
    // Apply
    let updated = match db.update_user(&user.id, update) {
        Ok(u) => u,
        Err(e) => return Err(ApiError::Internal(e.to_string())),
    };
    
    Ok(ProfileResponse::from(updated))
}
```

Thirty-seven lines. Clear, but repetitive. The match-on-Result, match-on-Option pattern dominates the visual structure.

**The same logic, using everything in this post:**

```rust
fn handle_update_profile(
    db: &Database,
    session: &Option<Session>,
    request: &UpdateProfileRequest,
) -> Result<ProfileResponse, ApiError> {
    let Some(session) = session else {
        return Err(ApiError::Unauthorized);
    };
    if !session.is_active() {
        return Err(ApiError::Unauthorized);
    }

    let user = db.find_user(&session.user_id)?
        .ok_or_else(|| ApiError::NotFound("user".into()))?;

    let validated_email = match &request.email {
        Some(email) if !email.contains('@') => {
            return Err(ApiError::Validation("invalid email".into()));
        }
        Some(email) => Some(email.clone()),
        None => None,
    };

    let updated = db.update_user(&user.id, UserUpdate {
        name: request.name.clone(),
        email: validated_email,
    })?;

    Ok(updated.into())
}
```

Twenty-two lines, down from thirty-seven. The session check uses `let-else`. The user lookup chains `?` with `ok_or_else`. The database update uses `?` directly. But notice the email validation - it stayed as a `match`.

I tried writing the email validation as a combinator chain first. Something like `request.email.as_ref().map(|e| e.contains('@').then(|| e.clone())).flatten()`. It's "functional" and "concise" but it's incomprehensible. You'd stare at it for 30 seconds trying to figure out what it does, and it doesn't even handle the error case (returning `Err`) properly because `map`/`flatten` can only produce `None`, not an error.

The match version is five lines, and every arm is a clear English statement: "if there's an email and it's invalid, error. If there's an email and it's valid, keep it. If there's no email, that's fine." Conciseness lost. Clarity won.

This brings us to the most important section.

## When verbose is better

There's a seductive trap in chasing conciseness. You start chaining everything, collapsing every match into a combinator, and the code becomes a puzzle instead of a narrative. Here are the cases where I deliberately choose the longer version.

### Error handling with context

```rust
// Too terse - which operation failed?
let result = fetch_data(url)?.parse()?.validate()?;

// Better - you can tell exactly where it broke
let response = fetch_data(url)
    .map_err(|e| AppError::fetch_failed(url, e))?;
let parsed = response.parse()
    .map_err(|e| AppError::parse_failed(url, e))?;
let validated = parsed.validate()
    .map_err(|e| AppError::validation_failed(&parsed.id, e))?;
```

The terse version gives you a `?` on every operation but the error that reaches the caller has no context. Which step failed? What URL? What data was being parsed? The verbose version adds `map_err` at each step, and each error carries enough context to debug the issue from the log alone. In production code, this matters more than saving lines.

Libraries like [anyhow](https://docs.rs/anyhow/latest/anyhow/) offer a middle ground with `.context()`:

```rust
use anyhow::Context;

let response = fetch_data(url)
    .context(format!("fetching {url}"))?;
let parsed = response.parse()
    .context("parsing response body")?;
```

Still explicit about what each step does, but less boilerplate than custom `map_err` closures.

### Complex boolean logic

```rust
// Clever but opaque
let access = matches!(role, Role::Admin)
    || (matches!(role, Role::Editor) && resource.is_published())
    || (matches!(role, Role::Author) && resource.author_id == user_id);

// Clear
let access = match role {
    Role::Admin => true,
    Role::Editor => resource.is_published(),
    Role::Author => resource.author_id == user_id,
    Role::Viewer => false,
};
```

The match version is two lines longer but immediately obvious. Each role maps to a condition. You can scan it in one pass. The `matches!` chain requires you to mentally parse operator precedence and three separate conditions.

There's another advantage: the match version is exhaustive. If someone adds a `Role::Moderator` variant next month, the compiler forces them to decide what access level it gets. The `matches!` chain would silently give moderators no access.

### Side effects in pipelines

```rust
// Don't
users.iter()
    .filter(|u| u.needs_notification())
    .for_each(|u| {
        let msg = build_message(u);
        email_service.send(&u.email, &msg).unwrap();
        audit_log.record(Action::Notified, &u.id);
        metrics.increment("notifications_sent");
    });

// Do
let users_to_notify: Vec<_> = users.iter()
    .filter(|u| u.needs_notification())
    .collect();

for user in &users_to_notify {
    let msg = build_message(user);
    email_service.send(&user.email, &msg)?;
    audit_log.record(Action::Notified, &user.id);
    metrics.increment("notifications_sent");
}
```

The `for_each` version hides side effects inside a closure, swallows errors with `unwrap()`, and makes it impossible to short-circuit on failure. The `for` loop makes the side effects visible, lets you use `?` for error propagation, and lets you add `break`/`continue` logic if needed.

A good rule: use iterator chains for transforming data, use `for` loops for performing actions.

## The readability checklist

When I write Rust and want it to read as clearly as possible, I run through these questions:

1. **Can I use `?` instead of `match` on Result/Option?** Almost always yes. Add `map_err` for context when needed.
2. **Can `let-else` replace `if let` + early return?** If you're just extracting a value or bailing, yes.
3. **Is the iterator chain under 7 combinators?** If not, break it into named intermediate bindings.
4. **Do the variable names describe what the value represents?** `active_users` beats `filtered`. `email_domain` beats `parts[1]`.
5. **Would a `match` be clearer than chained `if` conditions?** Especially when there are more than 3 cases, or when exhaustiveness matters.
6. **Are side effects in `for` loops, not in `for_each`?** Keep the type system helping you.
7. **Would a reader understand this without reading the function signature?** If not, the names need work.

## What's coming next

The Rust readability story keeps getting better. A few things on the horizon:

**Gen blocks** ([RFC 3513](https://rust-lang.github.io/rfcs/3513-gen-blocks.html)) - currently nightly-only, the `gen` keyword is reserved in the 2024 edition. Gen blocks will let you write iterators as sequential code with `yield`, similar to Python generators. This eliminates the need for complex state-machine implementations when you want a custom iterator.

**If-let guards in match arms** ([PR #141295](https://github.com/rust-lang/rust/pull/141295)) - currently being stabilized, this would allow `if let` patterns inside match guards, combining pattern matching with more complex destructuring in guard position.

Each of these chips away at another piece of accidental complexity. The trend line is clear: Rust is getting more expressive with each edition, not less. The language that people call verbose in 2024 will read very differently by 2027.

The goal was never conciseness for its own sake. It's clarity - making the code express intent so directly that reading it feels like reading the specification. When `let-else`, `?`, method chains, and pattern matching combine well, Rust gets surprisingly close to that ideal.

Just don't chain seven combinators with a `for_each` at the end and call it clean.
