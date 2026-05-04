+++
title = "Builder pattern in Rust - when and how to use it"
date = 2025-06-13
description = "Why Rust leans on the builder pattern harder than most languages, three ways to implement it, and when it becomes unnecessary overhead."

[taxonomies]
tags = ["rust", "design-patterns", "api-design"]
+++

If you've written any non-trivial Rust, you've already used the builder pattern. `reqwest::Client::builder()`, `Command::new("ls").arg("-la")`, `tokio::runtime::Builder::new_multi_thread()` - it's everywhere. More present in Rust than in Java, where the Gang of Four originally popularized it. That's not a coincidence. Rust's type system forces you toward builders in situations where other languages have simpler alternatives.

This post covers why the pattern fits Rust so well, three ways to implement it (manual, `derive_builder`, and `bon`), the typestate trick that turns runtime checks into compile-time errors, and when you should skip the builder entirely.

## Why Rust needs builders more than other languages

In Python you write `connect(host="localhost", port=5432, ssl=True, timeout=30)`. In C++ you write overloads. In Java you use telescoping constructors or just pass nulls. Rust has none of these escape hatches:

**No function overloading.** You can't define two functions with the same name that differ by argument count or type. One name, one signature, period.

**No default parameter values.** Every argument must be supplied at every call site. There's no `fn connect(port: u16 = 5432)`.

**No named arguments.** You can't write `connect(port: 5432, host: "localhost")` at the call site to clarify what each value means.

So what happens when you have a struct with 8 fields, 5 of them optional? Without builders, your options are grim:

```rust
// Option A: monster constructor
let cfg = Config::new("localhost", 5432, true, None, Some(30), None, None, false);
// Which bool is SSL? Which None is the password? Good luck.

// Option B: struct literal with defaults
let cfg = Config {
    host: "localhost".into(),
    port: 5432,
    ssl: true,
    password: None,
    timeout: Some(30),
    max_retries: None,
    pool_size: None,
    verbose: false,
};
// Better. But every caller must know every field, even optional ones.
```

The struct literal approach (Option B) is actually fine for internal code. But once you're building a public API, adding a new field is a breaking change unless you `#[non_exhaustive]` the struct - and then callers can't construct it directly at all. The builder gives you a stable API surface where adding new fields with defaults never breaks anyone.

## The ecosystem is built on this

Look at how the most downloaded crates use builders:

**reqwest** - HTTP client with dozens of configuration knobs:
```rust
let client = reqwest::Client::builder()
    .timeout(Duration::from_secs(10))
    .redirect(Policy::limited(5))
    .cookie_store(true)
    .user_agent("my-app/1.0")
    .build()?;
```

**std::process::Command** - a builder right in the standard library:
```rust
let output = Command::new("cargo")
    .arg("build")
    .arg("--release")
    .env("RUSTFLAGS", "-C target-cpu=native")
    .current_dir("/my/project")
    .output()?;
```

**tokio::runtime::Builder**:
```rust
let rt = tokio::runtime::Builder::new_multi_thread()
    .worker_threads(4)
    .enable_all()
    .build()?;
```

The pattern is the same every time: start with a constructor, chain configuration methods, finish with `.build()`. Callers only specify what they care about. Defaults handle the rest.

## Implementing a builder by hand

Before reaching for a macro, you should understand what happens underneath. Let's build a real example - an HTTP client config:

```rust
pub struct ClientConfig {
    base_url: String,
    timeout_ms: u64,
    max_retries: u32,
    bearer_token: Option<String>,
    follow_redirects: bool,
}
```

The builder is a separate struct where every field is `Option<T>`:

```rust
pub struct ClientConfigBuilder {
    base_url: Option<String>,
    timeout_ms: Option<u64>,
    max_retries: Option<u32>,
    bearer_token: Option<String>,
    follow_redirects: Option<bool>,
}
```

Each setter takes `self` by value (consuming builder) and returns `Self`:

```rust
impl ClientConfigBuilder {
    pub fn new() -> Self {
        Self {
            base_url: None,
            timeout_ms: None,
            max_retries: None,
            bearer_token: None,
            follow_redirects: None,
        }
    }

    pub fn base_url(mut self, url: impl Into<String>) -> Self {
        self.base_url = Some(url.into());
        self
    }

    pub fn timeout_ms(mut self, ms: u64) -> Self {
        self.timeout_ms = Some(ms);
        self
    }

    pub fn max_retries(mut self, n: u32) -> Self {
        self.max_retries = Some(n);
        self
    }

    pub fn bearer_token(mut self, token: impl Into<String>) -> Self {
        self.bearer_token = Some(token.into());
        self
    }

    pub fn follow_redirects(mut self, follow: bool) -> Self {
        self.follow_redirects = Some(follow);
        self
    }

    pub fn build(self) -> Result<ClientConfig, String> {
        Ok(ClientConfig {
            base_url: self.base_url.ok_or("base_url is required")?,
            timeout_ms: self.timeout_ms.unwrap_or(5000),
            max_retries: self.max_retries.unwrap_or(3),
            bearer_token: self.bearer_token,
            follow_redirects: self.follow_redirects.unwrap_or(true),
        })
    }
}

impl ClientConfig {
    pub fn builder() -> ClientConfigBuilder {
        ClientConfigBuilder::new()
    }
}
```

Usage:
```rust
let config = ClientConfig::builder()
    .base_url("https://api.example.com")
    .timeout_ms(10_000)
    .bearer_token("secret-token")
    .build()?;
```

This is about 50 lines for 5 fields. It's mechanical, repetitive code. The setter bodies are identical except for the field name. That's exactly the kind of boilerplate that macros exist to eliminate.

### Consuming vs borrowing setters

Notice the setters above take `self` by value (consuming). There's another style where setters take `&mut self`:

```rust
pub fn base_url(&mut self, url: impl Into<String>) -> &mut Self {
    self.base_url = Some(url.into());
    self
}
```

The `&mut self` approach lets you conditionally apply settings:

```rust
let mut builder = ClientConfig::builder();
builder.base_url("https://api.example.com");
if needs_auth {
    builder.bearer_token("secret");
}
let config = builder.build()?;
```

The consuming approach (`self`) enables a cleaner chained style but makes conditional logic awkward because you'd need `let builder = if needs_auth { builder.bearer_token("secret") } else { builder }`. The standard library's `Command` uses `&mut self`. Most derive macros give you the consuming style. Neither is wrong - pick based on how callers will use it.

### What this compiles to

Since the builder is just a struct with `Option` fields, it lives entirely on the stack. Each setter is a trivial field assignment. With optimizations enabled, LLVM inlines the entire builder chain and optimizes it down to direct field initialization - identical to constructing the struct with a literal. If you've read the [zero-cost abstractions post](/blog/zero-cost-abstractions-in-rust---what-it-actually-means/), you'll recognize the pattern: the abstraction disappears at compile time.

You can verify this on [Godbolt](https://godbolt.org). A builder chain like `.base_url("x").timeout_ms(100).build()` produces the same assembly as a direct struct construction once LLVM runs its passes. The `Option` wrappers and the intermediate builder struct get completely erased. Zero cost.

## derive_builder - the runtime approach

[derive_builder](https://crates.io/crates/derive_builder) (v0.20.2) has been around since 2016. It's the most downloaded builder crate. One derive macro generates the entire builder:

```rust
use derive_builder::Builder;

#[derive(Builder)]
#[builder(setter(into))]
pub struct ClientConfig {
    base_url: String,
    #[builder(default = "5000")]
    timeout_ms: u64,
    #[builder(default = "3")]
    max_retries: u32,
    #[builder(setter(strip_option))]
    bearer_token: Option<String>,
    #[builder(default = "true")]
    follow_redirects: bool,
}
```

This generates a `ClientConfigBuilder` with `&mut self` setters and a `build()` method returning `Result<ClientConfig, ClientConfigBuilderError>`:

```rust
let config = ClientConfigBuilder::default()
    .base_url("https://api.example.com")
    .timeout_ms(10_000_u64)
    .bearer_token("secret")
    .build()?;
```

The `setter(into)` attribute adds `impl Into<T>` bounds so you can pass `&str` where `String` is expected. `strip_option` lets you call `.bearer_token("value")` instead of `.bearer_token(Some("value".into()))`. `default` sets fallback values for fields not explicitly set.

### The trade-off

`derive_builder` validates at **runtime**. If you forget to set `base_url`, the code compiles just fine. You get an error at runtime when `.build()` returns `Err`. Nothing stops you from calling the same setter twice either.

You can add custom validation with the `build_fn(validate = "...")` attribute:

```rust
#[derive(Builder)]
#[builder(build_fn(validate = "Self::validate"))]
pub struct ClientConfig {
    base_url: String,
    timeout_ms: u64,
    max_retries: u32,
}

impl ClientConfigBuilder {
    fn validate(&self) -> Result<(), String> {
        if let Some(timeout) = self.timeout_ms {
            if timeout == 0 {
                return Err("timeout must be > 0".into());
            }
        }
        Ok(())
    }
}
```

For many use cases, runtime validation is fine. You're already handling `Result` from `.build()`. The error messages are clear. But if you want the compiler to catch mistakes before your code even runs, you need the typestate approach.

## bon - compile-time guarantees with typestate

[bon](https://crates.io/crates/bon) (v3.9.1) is the modern alternative. It was designed with lessons learned from both `derive_builder` and `typed-builder`. The key difference: bon uses the [typestate pattern](https://docs.rs/bon/latest/bon/) to enforce correctness at compile time.

```rust
use bon::Builder;

#[derive(Builder)]
pub struct ClientConfig {
    #[builder(into)]
    base_url: String,
    #[builder(default = 5000)]
    timeout_ms: u64,
    #[builder(default = 3)]
    max_retries: u32,
    #[builder(into)]
    bearer_token: Option<String>,
    #[builder(default = true)]
    follow_redirects: bool,
}
```

Usage looks the same:
```rust
let config = ClientConfig::builder()
    .base_url("https://api.example.com")
    .timeout_ms(10_000)
    .bearer_token("secret")
    .build();
```

But the behavior is different. `base_url` has no default, so it's required. If you skip it:

```rust
let config = ClientConfig::builder()
    .timeout_ms(10_000)
    .build(); // Compile error: base_url was not set
```

This fails at compile time, not runtime. The `.build()` method simply doesn't exist on the builder type until all required fields have been set.

### bon also works on functions

This is where bon really shines compared to alternatives. You can put `#[builder]` on a free function or method:

```rust
use bon::builder;

#[builder]
fn send_email(
    #[builder(into)] to: String,
    #[builder(into)] subject: String,
    #[builder(into)] body: String,
    #[builder(default)] html: bool,
    #[builder(default = 3)] retries: u32,
) -> Result<(), String> {
    // send logic
    Ok(())
}

// Callers get named parameters:
send_email()
    .to("user@example.com")
    .subject("Hello")
    .body("World")
    .html(true)
    .call()?;
```

This effectively gives Rust named parameters. The function signature stays clean. Callers get an ergonomic API. Required parameters are enforced at compile time. This pattern works on `impl` blocks too - put `#[bon]` on the impl block and `#[builder]` on methods.

## How typestate builders actually work

The typestate pattern encodes state into the type system. Each configuration step changes the type of the builder. I covered how Rust's generics and monomorphization work in the [trait objects vs enums vs generics post](/blog/trait-objects-vs-enums-vs-generics) - typestate builds directly on those mechanics.

Here's a simplified version of what bon generates under the hood:

```rust
use std::marker::PhantomData;

// States - zero-sized types, exist only at compile time
struct Missing;
struct Set;

struct ClientConfigBuilder<BaseUrl, Timeout> {
    base_url: Option<String>,
    timeout_ms: Option<u64>,
    _state: PhantomData<(BaseUrl, Timeout)>,
}

impl ClientConfigBuilder<Missing, Missing> {
    fn new() -> Self {
        Self {
            base_url: None,
            timeout_ms: None,
            _state: PhantomData,
        }
    }
}

// Setting base_url transitions Missing -> Set for that type parameter
impl<Timeout> ClientConfigBuilder<Missing, Timeout> {
    fn base_url(self, url: impl Into<String>) -> ClientConfigBuilder<Set, Timeout> {
        ClientConfigBuilder {
            base_url: Some(url.into()),
            timeout_ms: self.timeout_ms,
            _state: PhantomData,
        }
    }
}

// timeout can be set regardless of base_url's state
impl<BaseUrl> ClientConfigBuilder<BaseUrl, Missing> {
    fn timeout_ms(self, ms: u64) -> ClientConfigBuilder<BaseUrl, Set> {
        ClientConfigBuilder {
            base_url: self.base_url,
            timeout_ms: Some(ms),
            _state: PhantomData,
        }
    }
}

// build() only exists when ALL required fields are Set
impl ClientConfigBuilder<Set, Set> {
    fn build(self) -> ClientConfig {
        ClientConfig {
            base_url: self.base_url.unwrap(), // safe: guaranteed by typestate
            timeout_ms: self.timeout_ms.unwrap(),
        }
    }
}
```

The `PhantomData` and zero-sized type parameters have zero runtime cost. They exist purely in the type system. After monomorphization, LLVM sees concrete structs with no phantom fields, and optimizes them away entirely.

The compiler error you get when forgetting a field looks something like:

```
error[E0599]: no method named `build` found for struct
  `ClientConfigBuilder<Missing, Set>` in the current scope
```

That `Missing` in the type signature tells you exactly which field you forgot.

### The `base_url` setter only exists on `ClientConfigBuilder<Missing, _>`

This means calling `.base_url()` twice is also a compile error. The first call transitions the type parameter from `Missing` to `Set`. The second call would need `Missing` again, but the type is now `Set` - so the method doesn't exist. Double-setting is structurally impossible.

### Trade-off: more generated code

The typestate approach generates more code per struct. For N required fields, you get 2^N possible type states (though the compiler only monomorphizes the paths your code actually uses). With many required fields, compile times can increase. For a struct with 3-4 required fields, you won't notice. With 15 required fields, you might.

## When the builder pattern is overkill

Not every struct needs a builder. Here's when to skip it:

**Small structs with all required fields.** If your struct has 2-3 fields and all of them are mandatory, a plain constructor is clearer:

```rust
struct Point {
    x: f64,
    y: f64,
}

impl Point {
    fn new(x: f64, y: f64) -> Self {
        Self { x, y }
    }
}

// This is perfectly readable. A builder would add noise.
let p = Point::new(1.0, 2.0);
```

**Internal types.** For types that only your crate constructs, struct literal syntax works fine. You control all the call sites. If you add a field, you update every constructor yourself. No backward compatibility concern.

**Types with clear "slots".** If the arguments have distinct types that make the meaning obvious, a constructor is fine:

```rust
// Unambiguous: the first arg is clearly the name, the second is the age
let user = User::new("Alice".into(), 30);
```

Compare with a case where a builder helps:

```rust
// Ambiguous: is 5432 the port or the timeout? Is true for SSL or verbose?
let conn = Connection::new("localhost", 5432, true, 30, false);
// vs
let conn = Connection::builder()
    .host("localhost")
    .port(5432)
    .ssl(true)
    .timeout(30)
    .verbose(false)
    .build()?;
```

**The Default + struct update pattern.** For config-like types with sensible defaults, sometimes `Default` plus struct update syntax is enough:

```rust
#[derive(Default)]
struct Config {
    verbose: bool,
    max_retries: u32,
    timeout_ms: u64,
}

let cfg = Config {
    timeout_ms: 10_000,
    ..Config::default()
};
```

This doesn't work for public APIs (adding fields is breaking), but for internal use it's the simplest option.

## Decision framework

Here's how I think about it:

| Situation | Approach |
|---|---|
| 2-3 fields, all required, types are distinct | Plain `new()` constructor |
| Internal config, sensible defaults | `#[derive(Default)]` + struct update |
| Public API, many optional fields | Builder (manual or derived) |
| Need compile-time required field checks | `bon` or manual typestate |
| Want runtime validation (cross-field checks) | `derive_builder` with `validate` |
| Want named params on a function | `bon` with `#[builder]` on fn |

## Picking a crate

If you've decided you want a derive macro, here's the landscape:

**[derive_builder](https://crates.io/crates/derive_builder)** - The veteran. Runtime validation. `&mut self` setters by default. Great if you need conditional builder logic or want to pass builders around as a single concrete type without generics. The downside: forgotten required fields are runtime errors.

**[bon](https://crates.io/crates/bon)** - The modern pick. Compile-time typestate validation. Works on structs, functions, and methods. More attributes for fine-grained control (`#[builder(into)]`, `#[builder(default)]`, custom finish function names). The downside: more complex generated types can make error messages harder to read.

**[typed-builder](https://crates.io/crates/typed-builder)** - Similar to bon's compile-time approach. Also uses typestate. Less feature-rich than bon (no function builders). Was the go-to before bon existed. Still actively maintained and perfectly fine if you're already using it.

For new projects, I'd reach for `bon`. It covers the most ground (structs + functions + methods), has the best compile-time safety, and is actively developed. But any of these three are solid choices - the worst option is spending an hour deciding between them.

## A complete example: HTTP client with layered config

Let's tie it together with a more realistic scenario. An HTTP client that reads defaults from environment variables, lets users override via builder, and validates the final config:

```rust
use bon::Builder;
use std::time::Duration;

#[derive(Builder, Debug)]
pub struct HttpClient {
    #[builder(into)]
    base_url: String,

    #[builder(default = default_timeout())]
    timeout: Duration,

    #[builder(default = 3)]
    max_retries: u32,

    #[builder(into)]
    bearer_token: Option<String>,

    #[builder(default)]
    headers: Vec<(String, String)>,

    #[builder(default = true)]
    follow_redirects: bool,
}

fn default_timeout() -> Duration {
    let ms: u64 = std::env::var("HTTP_TIMEOUT_MS")
        .ok()
        .and_then(|v| v.parse().ok())
        .unwrap_or(5000);
    Duration::from_millis(ms)
}

impl HttpClient {
    pub fn get(&self, path: &str) -> String {
        format!("GET {}{}", self.base_url, path)
    }
}
```

Usage:

```rust
// Minimal - only required field
let client = HttpClient::builder()
    .base_url("https://api.example.com")
    .build();

// Full config
let client = HttpClient::builder()
    .base_url("https://api.example.com")
    .timeout(Duration::from_secs(30))
    .max_retries(5)
    .bearer_token("my-token")
    .headers(vec![
        ("X-Request-Id".into(), "abc-123".into()),
    ])
    .follow_redirects(false)
    .build();
```

The `base_url` field has no default, so `bon` makes it required at compile time. Everything else has defaults. The timeout reads from an environment variable with a fallback. The builder gives callers a clean API while keeping the internals flexible.

## What I'd want you to take away

The builder pattern in Rust isn't just an OOP holdover - it solves real problems that come from Rust's deliberate lack of overloading, default arguments, and named parameters. The ecosystem has standardized on it for good reason.

For most cases, `bon` with `#[derive(Builder)]` gives you the best trade-off: minimal boilerplate, compile-time safety, and support for functions and methods too. But don't reach for it reflexively. A struct with two required fields doesn't need a builder. `Default` with struct update syntax handles plenty of internal config types. The right tool depends on whether you're building a public API or wiring up internal plumbing.

If you write a builder by hand once, you'll understand what the macros generate, and you'll have better intuition for when the pattern helps versus when it's ceremony for ceremony's sake.
