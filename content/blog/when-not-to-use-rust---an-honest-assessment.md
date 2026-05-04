+++
title = "When not to use Rust - an honest assessment"
date = 2026-04-20
description = "Rust is excellent, but not for everything. A practical framework for deciding when Rust helps and when it actively slows you down."

[taxonomies]
tags = ["rust", "architecture", "python", "decision-making"]
+++

I write a lot of Rust. Most of the posts on this blog are about Rust. I genuinely think it's one of the best systems programming languages ever designed. And I also think reaching for it on every new project is a mistake.

The Rust community has a bit of an enthusiasm problem. Someone asks "what should I use for X?" and the answer is always Rust. Rewrite it in Rust. Rust all the things. That energy built a phenomenal ecosystem, but it also leads to poor technology decisions when the tradeoffs don't line up.

This post is about those tradeoffs. Not "Rust bad" - that would be a lazy take. More like: here's a framework for thinking about when Rust's costs outweigh its benefits, based on real project characteristics.

<!-- more -->

## The costs nobody talks about

Before getting into specific scenarios, it's worth being explicit about what Rust costs you. Not in dollars - in time, iteration speed, and team bandwidth.

### Compile times

The 2025 [Rust Compiler Performance Survey](https://blog.rust-lang.org/2025/09/10/rust-compiler-performance-survey-2025-results/) found that **55% of respondents wait more than ten seconds for a rebuild**. That's a rebuild - not a clean build. Clean builds on non-trivial projects routinely hit 2-5 minutes. Large projects with many proc-macro-heavy dependencies can push past 10.

Compare that to Python (zero compile time), Go (typically under 2 seconds for rebuilds), or JavaScript (instant with hot module replacement). Those seconds add up. If you're iterating on business logic and rebuilding 200 times a day, the difference between 0.5 seconds and 15 seconds is an hour of staring at your terminal.

The compiler has gotten faster - Nicholas Nethercote's [work throughout 2025](https://nnethercote.github.io/2025/12/05/how-to-speed-up-the-rust-compiler-in-december-2025.html) brought measurable improvements, and parallel frontend compilation is progressing. But "faster than it was" is different from "fast." The feedback loop in Rust is still meaningfully slower than in most languages you'd compare it against.

### Learning curve

This isn't just a "read the book" problem. The [2025 Rust survey](https://blog.jetbrains.com/rust/2026/02/11/state-of-rust-2025/) reports that **41.6% of current Rust developers worry the language is becoming too complex**. Teams report 3-6 months before developers trained in other languages reach full productivity in Rust.

If you've been writing Rust for years, it's easy to forget how brutal the early weeks are. Ownership, borrowing, lifetimes, trait bounds, async Pin/Unpin, turbofish syntax - the concepts stack up fast. I covered lifetimes in [a previous post](/blog/lifetimes-in-rust-the-mental-model-that-finally-clicked/) and even with a solid mental model, it took real time to internalize.

The 2025 Stack Overflow survey has Rust at **72% "most admired"** among developers who use it, but actual adoption remains modest. People love it in theory. Using it daily is a different conversation.

### Fighting the borrow checker on the wrong problem

Some problem domains map naturally to Rust's ownership model. Some don't. If your code is fundamentally about shared mutable state - UI frameworks, game entity systems, graph algorithms - you'll spend a disproportionate amount of time satisfying the borrow checker with `Rc<RefCell<T>>`, `Arc<Mutex<T>>`, or index-based patterns that work around the ownership rules rather than benefiting from them.

That time isn't free. It's time you could spend on the actual problem.

## When not to use Rust

### Rapid prototyping and throwaway code

You're validating a product idea. You don't know if the feature will survive next sprint. You need to get something in front of users this week, not this quarter.

Python and JavaScript let you move at a fundamentally different speed for this kind of work. Not because the languages are better - because the feedback loop is shorter and the type system won't stop you from doing something sloppy that's perfectly fine for a prototype.

Here's a concrete example. Say you need to hit three APIs, merge the results, and display them. In Python:

```python
import httpx
import asyncio

async def fetch_all():
    async with httpx.AsyncClient() as client:
        users, orders, products = await asyncio.gather(
            client.get("https://api.example.com/users"),
            client.get("https://api.example.com/orders"),
            client.get("https://api.example.com/products"),
        )
    return {
        "users": users.json(),
        "orders": orders.json(),
        "products": products.json(),
    }
```

That's 12 lines. You wrote it in two minutes. The return type is a dict of whatever the APIs give you. You don't need to define structs, derive traits, or handle every possible error variant.

The Rust equivalent:

```rust
use reqwest::Client;
use serde::Deserialize;

#[derive(Deserialize)]
struct User { /* fields */ }
#[derive(Deserialize)]
struct Order { /* fields */ }
#[derive(Deserialize)]
struct Product { /* fields */ }

struct MergedData {
    users: Vec<User>,
    orders: Vec<Order>,
    products: Vec<Product>,
}

async fn fetch_all() -> Result<MergedData, reqwest::Error> {
    let client = Client::new();
    let (users, orders, products) = tokio::join!(
        client.get("https://api.example.com/users").send(),
        client.get("https://api.example.com/orders").send(),
        client.get("https://api.example.com/products").send(),
    );
    Ok(MergedData {
        users: users?.json().await?,
        orders: orders?.json().await?,
        products: products?.json().await?,
    })
}
```

More lines. More ceremony. You had to define struct shapes up front. If the API changes a field name, you get a compile error instead of silently propagating wrong data - which is *great* for production code and *annoying* when you're exploring.

The Rust version is more correct. The Python version shipped three days ago and is already generating user feedback. Correctness you don't need yet is a cost, not a feature.

### Small scripts and automation

You need to rename 500 files based on a CSV mapping. Parse some logs and count error rates. Glue two CLI tools together. Monitor a directory for changes and trigger a webhook.

This is bash and Python territory. Not because Rust can't do it - it absolutely can - but because the overhead of `Cargo.toml`, compilation, struct definitions, and error handling for a script you'll run twice and then forget about is not a good trade.

```bash
# Rename files from CSV mapping
while IFS=, read -r old new; do
    mv "$old" "$new"
done < mapping.csv
```

That's the whole thing. It runs right now. No dependencies, no compilation, no type definitions. Will it handle edge cases with spaces in filenames? Probably not. Does it need to for a one-off task? Probably not.

A rule of thumb: if the script is under 100 lines and you'll run it fewer than 10 times, a compiled language adds friction without benefit. If it grows into something long-lived or needs to be reliable, then the calculus changes.

### Data science and machine learning

Python isn't dominant in data science by accident. The ecosystem is enormous and deeply integrated:

- **pandas** and **polars** for dataframes (and yes, polars is written in Rust, but you use it through Python)
- **NumPy** for numerical computing, backed by decades of optimized BLAS/LAPACK routines
- **scikit-learn** for ML with 200+ algorithms ready to use
- **matplotlib**, **seaborn**, **plotly** for visualization
- **Jupyter notebooks** for interactive exploration
- **PyTorch** and **TensorFlow** for deep learning, with GPU support, pretrained models, and massive community resources

Rust's data science story is improving. [Polars](https://pola.rs/) is excellent for dataframe operations. [Linfa](https://github.com/rust-ml/linfa) provides classical ML algorithms. [Burn](https://burn.dev/) is a promising deep learning framework. But "promising" is not "production-ready ecosystem with 15 years of community knowledge."

The practical gap shows up in the workflow, not just the libraries. A data scientist exploring a dataset needs to:

1. Load data, look at it, filter some rows
2. Try three different transformations
3. Plot intermediate results
4. Fit a model, examine coefficients
5. Iterate on feature engineering
6. Generate a report with inline visualizations

Every step of that workflow is optimized for Python. Jupyter notebooks let you execute cells interactively and see results inline. There's no Rust equivalent that's even close. You *could* do exploratory data analysis in Rust, but you'd be fighting the tooling the entire time.

The smart pattern here is the one the ecosystem is already converging on: Python as the interface, Rust as the engine. Polars, [pydantic](https://github.com/pydantic/pydantic) (v2 core is Rust), [ruff](https://github.com/astral-sh/ruff) (Python linter written in Rust), [uv](https://github.com/astral-sh/uv) (Python package manager written in Rust) - all of these use Rust where raw performance matters and expose a Python API where developer experience matters.

### Quick CRUD web applications

You're building a standard web app. Users, authentication, database, REST API, admin panel. The kind of thing startups ship every day.

Rails, Django, and Laravel exist specifically for this. They give you:

- Database migrations out of the box
- ORM with query builders
- Authentication and session management built-in
- Admin interface generators
- Form validation
- Template engines
- A massive ecosystem of plugins for common features

A new Rails app with user authentication, a PostgreSQL-backed model, and a REST API takes about 15 minutes to scaffold:

```bash
rails new myapp --database=postgresql
cd myapp
rails generate scaffold User name:string email:string
rails generate devise:install
rails db:migrate
```

You now have a working app with CRUD endpoints, database schema, validations, and views. In Rust with [Axum](https://github.com/tokio-rs/axum), you'd still be writing your database connection pool configuration.

Rust web frameworks have matured significantly. Axum, Actix-web, and Loco are all solid. [Loco](https://loco.rs/) in particular is inspired by Rails and closes much of the gap. But "closes much of the gap" still means more manual wiring, fewer batteries included, and a smaller pool of tutorials, Stack Overflow answers, and hired developers who already know the framework.

The exception: if your CRUD app needs to handle 50,000 concurrent connections on a single server, or if response latency at the tail matters for your business, Rust's performance advantage is real and measurable. I wrote about companies making this exact tradeoff in [Rust in production](/blog/rust-in-production-what-companies-actually-use-it-for/) - Cloudflare's Pingora serving a trillion requests per day, Discord eliminating GC pauses. But those are infrastructure-scale problems. Most CRUD apps never get there.

### When your team doesn't know Rust

This one is underrated. You have a team of five. Four write Python, one writes Go. You have a deadline in three months. Choosing Rust means:

- 3-6 months of reduced productivity while the team learns
- Review bottleneck (who reviews Rust code if nobody's fluent?)
- Debugging takes longer when you're still learning the idioms
- Hiring is harder - the Rust talent pool is smaller and more expensive (Rust roles pay [10-20% more](https://blog.jetbrains.com/rust/2026/02/11/state-of-rust-2025/) than equivalent Go or Python positions)
- One person leaving creates a bus factor problem

The learning curve of Rust isn't just about syntax. It's about learning to think differently about data ownership. Experienced developers from garbage-collected languages consistently report that the hardest part isn't the syntax or the borrow checker errors - it's rewiring their mental model about who owns what and when things get dropped.

This doesn't mean "never adopt Rust." It means budget for the transition honestly. Start with a non-critical internal tool. Let the team build intuition before betting the product roadmap on a language nobody knows yet.

## When Rust IS the right choice

I don't want this post to read like I'm down on Rust. There are problem domains where it's genuinely the best tool available. Most of these map to scenarios where Rust's costs (compilation time, learning curve, verbosity) are small relative to the value of its guarantees (memory safety without GC, fearless concurrency, zero-cost abstractions).

**Performance-critical services.** If you're building infrastructure that processes millions of requests per second, where tail latency matters, where memory usage directly maps to server cost - Rust pays for itself. The [companies using Rust in production](/blog/rust-in-production-what-companies-actually-use-it-for/) are overwhelmingly in this category.

**Long-running services with large heaps.** Discord's story is the canonical example. If your service holds millions of objects in memory for extended periods, any garbage-collected language will eventually hit pause times. Rust sidesteps the problem entirely.

**CLI tools.** Fast startup, single static binary, cross-compilation to every platform. Rust is arguably the best language for CLI tools right now. [ripgrep](https://github.com/BurntSushi/ripgrep), [fd](https://github.com/sharkdp/fd), [bat](https://github.com/sharkdp/bat), [delta](https://github.com/dandavison/delta), ruff, uv - the pattern is well-established. I covered the cross-compilation story in [a previous post](/blog/cross-compilation-in-rust/).

**WebAssembly.** Rust's WASM story is first-class. No runtime to bundle, predictable performance, small binary sizes. [wasm-bindgen](https://github.com/rustwasm/wasm-bindgen) and [wasm-pack](https://github.com/nickel-org/wasm-pack) make the integration smooth. If you're shipping compute-heavy logic to the browser, Rust is a strong default.

**Safety-critical and embedded systems.** Where correctness is non-negotiable - automotive, aerospace, medical devices - Rust's compile-time guarantees eliminate entire classes of bugs that would require extensive runtime testing in C. The feature flags system I covered in [an earlier post](/blog/feature-flags-in-rust-conditional-compilation-with-cfg/) shows how Rust's `cfg` system handles the conditional compilation that embedded work demands.

**Libraries and engines consumed through other languages.** The "Rust under the hood, Python/JS on top" pattern is proving incredibly effective. Write the performance-critical core in Rust, expose it through FFI or WASM. You get Rust's safety and speed where it matters, and the target language's developer experience where that matters.

## A decision framework

When evaluating Rust for a new project, I run through these questions:

**1. What's the lifespan of this code?**
Throwaway prototype or weekend script? Skip Rust. Service that'll run for five years? Rust's upfront investment amortizes over time. The longer the code lives, the more you benefit from compile-time correctness.

**2. What are the actual performance requirements?**
"It should be fast" is not a requirement. "P99 latency under 5ms at 10k RPS" is. Most applications don't have performance requirements that Python or Go can't meet. Be honest about whether you need Rust's performance or just want it.

**3. Who will maintain this code?**
If the answer is "me, and I know Rust" - great. If the answer is "a team of six, none of whom have written Rust" - factor in the 3-6 month productivity dip and decide if the timeline allows it.

**4. Does the problem map to Rust's strengths?**
Data processing pipelines, network services, parsers, compilers, CLI tools - these map well. GUI apps, data science notebooks, rapid business logic iteration - these don't, at least not today.

**5. What does the ecosystem look like?**
Before committing, check if the libraries you need exist *and are maintained*. A crate with 50 GitHub stars and no commits in 8 months is not a foundation to build on. I covered how to evaluate dependencies in [Build vs buy](/blog/build-vs-buy-when-to-use-a-library-and-when-to-write-your-own/).

**6. What's the cost of a bug?**
A crash in a CLI tool is annoying. A memory safety bug in a proxy handling financial transactions is catastrophic. The higher the cost of failure, the more Rust's compile-time safety guarantees are worth.

## The mature take

Technology choices should be boring. Not "what's the most interesting language" but "what gets this specific thing shipped reliably with the team I have on the timeline I have."

Rust is an exceptional language that's wrong for a significant percentage of projects. Python is a slow language that's right for an enormous percentage of projects. Go is a deliberately limited language that ships production services remarkably fast. None of these statements are contradictions.

The strongest signal that a team has good engineering judgment isn't that they use Rust - it's that they can articulate *why* they use it for some things and not others. The companies I wrote about in the [production post](/blog/rust-in-production-what-companies-actually-use-it-for/) all share this trait: they picked Rust for specific subsystems where its strengths mattered, and they kept everything else in whatever language was already working.

Use Rust when the problem demands it. Use something else when it doesn't. That's not a betrayal of the language - it's the most honest form of advocacy.
