+++
title = "Ports and adapters in Rust - dependency injection the compiler already gives you"
date = 2025-12-06
description = "How hexagonal architecture maps naturally to Rust traits, and why you don't need Spring or NestJS to get clean dependency injection."

[taxonomies]
tags = ["rust", "architecture", "design-patterns", "testing"]
+++

In Spring, you annotate a class with `@Service`, its constructor with `@Autowired`, define an interface, and the framework scans your classpath at startup, builds a dependency graph via reflection, and injects the right implementation. In NestJS, you decorate with `@Injectable()`, register providers in a module, and the container resolves constructor parameter types via `reflect-metadata` at runtime. Both frameworks do real work - classpath scanning, proxy generation, metadata reflection - to solve a problem that Rust solves at compile time for free.

The pattern behind all of this is hexagonal architecture, also called ports and adapters. Alistair Cockburn [introduced it in 2005](https://alistair.cockburn.us/hexagonal-architecture/), and the core idea is simple: your business logic sits at the center, talks to the outside world through interfaces (ports), and concrete implementations (adapters) plug into those interfaces. The application core never knows what's on the other end.

In Rust, a port is a trait. An adapter is an impl. The compiler is your DI container. No annotations, no scanning, no runtime cost.

I wrote about [wrapping external APIs with the adapter pattern](/blog/the-adapter-pattern-in-rust-wrapping-external-apis/) back in April, and later about [four dependency injection patterns](/blog/dependency-injection-patterns-without-a-framework/) in May. This post connects those ideas into the full hexagonal architecture and compares what Rust gives you natively against what Spring and NestJS need frameworks to achieve.

<!-- more -->

## The hexagon in 30 seconds

Picture your application as a hexagon (the shape is arbitrary - Cockburn picked it because it has enough sides to draw multiple ports). The inside is your business logic. The edges are ports - interfaces that define how the outside world talks to you and how you talk to the outside world.

Two kinds of ports:

**Driving ports** (primary) - the outside world calls into your application. An HTTP handler, a CLI command, a test harness. These are the entry points.

**Driven ports** (secondary) - your application calls out. A database, an external API, a file system, a message queue. These are the dependencies.

Adapters sit outside the hexagon and implement the ports. A `PostgresUserRepo` implements the `UserRepository` port. An `AxumRouter` implements the HTTP driving port. A `MockGitHub` implements the `GitHubPort` for tests.

The rule: dependencies point inward. The application core depends on port traits. Adapters depend on the core (they implement its traits). The core never imports an adapter. This is what makes everything swappable and testable.

## Building it: a repo health scorer

Enough theory. Here's a complete example - a tool that scores GitHub repositories by health (stars vs open issues). We'll define a port, build an adapter that shells out to the `gh` CLI, write the scoring logic in a core that doesn't know GitHub exists, and test everything with a mock.

### The port and domain types

The port lives in your domain layer. It depends on nothing external - no `octocrab`, no `reqwest`, no `gh`. Just your types:

```rust
// src/ports.rs
use async_trait::async_trait;

#[derive(Debug, Clone)]
pub struct Repo {
    pub full_name: String,
    pub stars: u64,
    pub open_issues: u64,
    pub last_push: String,
}

#[derive(Debug, thiserror::Error)]
pub enum PortError {
    #[error("not found: {0}")]
    NotFound(String),
    #[error("auth failed - run `gh auth login`")]
    AuthFailed,
    #[error("rate limited, retry later")]
    RateLimited,
    #[error("{0}")]
    Other(String),
}

#[async_trait]
pub trait GitHubPort: Send + Sync {
    async fn list_repos(&self, owner: &str) -> Result<Vec<Repo>, PortError>;
    async fn get_repo(&self, owner: &str, name: &str) -> Result<Repo, PortError>;
}
```

This is the entire contract. The application core will depend on `GitHubPort` and nothing else. Whether the implementation shells out to `gh`, calls the REST API with `reqwest`, uses `octocrab`, or returns hardcoded data - the core doesn't care.

Notice the error type. `PortError` doesn't mention HTTP status codes or process exit codes. It speaks in domain terms: not found, auth failed, rate limited. The adapter is responsible for translating infrastructure errors into these.

### The adapter: shelling out to gh

The [previous adapter pattern post](/blog/the-adapter-pattern-in-rust-wrapping-external-apis/) wrapped `octocrab` - a Rust crate. This adapter wraps a CLI binary. Different infrastructure, same port interface:

```rust
// src/adapters/gh_cli.rs
use serde::Deserialize;
use tokio::process::Command;
use crate::ports::{GitHubPort, PortError, Repo};
use async_trait::async_trait;

pub struct GhCliAdapter;

#[derive(Deserialize)]
struct GhRepoJson {
    full_name: String,
    stargazers_count: u64,
    open_issues_count: u64,
    pushed_at: String,
}

impl GhCliAdapter {
    async fn run_gh(&self, args: &[&str]) -> Result<String, PortError> {
        let output = Command::new("gh")
            .args(args)
            .output()
            .await
            .map_err(|e| PortError::Other(format!("failed to spawn gh: {e}")))?;

        if !output.status.success() {
            let stderr = String::from_utf8_lossy(&output.stderr);
            if stderr.contains("auth") || stderr.contains("login") {
                return Err(PortError::AuthFailed);
            }
            if stderr.contains("rate limit") {
                return Err(PortError::RateLimited);
            }
            return Err(PortError::Other(
                format!("gh exited {}: {}", output.status, stderr.trim()),
            ));
        }

        String::from_utf8(output.stdout)
            .map_err(|e| PortError::Other(format!("invalid utf-8: {e}")))
    }
}

#[async_trait]
impl GitHubPort for GhCliAdapter {
    async fn list_repos(&self, owner: &str) -> Result<Vec<Repo>, PortError> {
        let json = self
            .run_gh(&["api", &format!("/orgs/{owner}/repos?per_page=100")])
            .await?;

        let items: Vec<GhRepoJson> = serde_json::from_str(&json)
            .map_err(|e| PortError::Other(format!("json parse error: {e}")))?;

        Ok(items.into_iter().map(into_repo).collect())
    }

    async fn get_repo(&self, owner: &str, name: &str) -> Result<Repo, PortError> {
        let json = self
            .run_gh(&["api", &format!("/repos/{owner}/{name}")])
            .await?;

        let r: GhRepoJson = serde_json::from_str(&json)
            .map_err(|e| PortError::Other(format!("json parse error: {e}")))?;

        Ok(into_repo(r))
    }
}

fn into_repo(r: GhRepoJson) -> Repo {
    Repo {
        full_name: r.full_name,
        stars: r.stargazers_count,
        open_issues: r.open_issues_count,
        last_push: r.pushed_at,
    }
}
```

A few things worth noting. We use `tokio::process::Command` instead of `std::process::Command` - the tokio version is non-blocking, so we don't stall the async runtime while waiting for `gh` to return. If you're unfamiliar with why blocking inside async is a problem, [the tokio post](/blog/understanding-tokio-the-rust-async-runtime/) covered `spawn_blocking` and the cooperative scheduler.

The error mapping in `run_gh` is the adapter's core job: translate process-level failures (exit codes, stderr output) into domain-level errors (`AuthFailed`, `RateLimited`). The `GhRepoJson` struct mirrors GitHub's REST API response shape - `stargazers_count`, `open_issues_count`, `pushed_at` - and `into_repo` converts it to our domain `Repo`. All the GitHub API knowledge stays in this one file.

### The application core

The scorer depends only on the `GitHubPort` trait. It doesn't import `GhCliAdapter`, doesn't know about processes or HTTP, doesn't use `serde`:

```rust
// src/core.rs
use crate::ports::{GitHubPort, PortError};

#[derive(Debug, Clone, serde::Serialize)]
pub struct RepoScore {
    pub full_name: String,
    pub score: f64,
    pub stars: u64,
    pub open_issues: u64,
}

pub struct RepoScorer<G: GitHubPort> {
    github: G,
}

impl<G: GitHubPort> RepoScorer<G> {
    pub fn new(github: G) -> Self {
        Self { github }
    }

    pub async fn score_org(&self, owner: &str) -> Result<Vec<RepoScore>, PortError> {
        let repos = self.github.list_repos(owner).await?;

        let mut scores: Vec<RepoScore> = repos
            .into_iter()
            .map(|r| {
                let star_signal = (r.stars as f64).ln_1p();
                let issue_penalty = (r.open_issues as f64).sqrt() * 0.5;
                let score = (star_signal - issue_penalty).max(0.0);
                RepoScore {
                    full_name: r.full_name,
                    score,
                    stars: r.stars,
                    open_issues: r.open_issues,
                }
            })
            .collect();

        scores.sort_by(|a, b| {
            b.score.partial_cmp(&a.score).unwrap_or(std::cmp::Ordering::Equal)
        });

        Ok(scores)
    }

    pub async fn top_n(&self, owner: &str, n: usize) -> Result<Vec<RepoScore>, PortError> {
        let mut scores = self.score_org(owner).await?;
        scores.truncate(n);
        Ok(scores)
    }
}
```

`RepoScorer<G: GitHubPort>` is generic over the port. When you instantiate it with `GhCliAdapter`, the compiler monomorphizes - it generates a concrete `RepoScorer<GhCliAdapter>` where every call to `self.github.list_repos()` is a direct function call, zero indirection. When you instantiate it with a mock, you get a different monomorphized version. No vtable, no box, no allocation.

If you prefer dynamic dispatch (fewer type parameters, one binary copy of the scoring logic), swap the generic for a trait object:

```rust
pub struct RepoScorer {
    github: Arc<dyn GitHubPort>,
}

impl RepoScorer {
    pub fn new(github: Arc<dyn GitHubPort>) -> Self {
        Self { github }
    }
    // ... same methods, same logic
}
```

The [DI patterns post](/blog/dependency-injection-patterns-without-a-framework/) walked through this trade-off in detail. Short version: generics for libraries, `dyn` for applications.

### Wiring in main

```rust
// src/main.rs
mod adapters;
mod core;
mod ports;

use adapters::gh_cli::GhCliAdapter;
use core::RepoScorer;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let github = GhCliAdapter;
    let scorer = RepoScorer::new(github);

    let scores = scorer.top_n("rust-lang", 5).await?;
    for s in &scores {
        println!("{:<40} score={:.2}  stars={}  issues={}",
            s.full_name, s.score, s.stars, s.open_issues);
    }

    Ok(())
}
```

Two lines of wiring. `GhCliAdapter` gets created, passed to `RepoScorer::new`, done. The compiler verified at compile time that `GhCliAdapter` implements `GitHubPort`. If it didn't, you'd get a trait bound error pointing at the exact line and the exact missing method. Compare that with Spring's `NoSuchBeanDefinitionException` at startup or NestJS's "Nest can't resolve dependencies of RepoScorer" runtime error.

### Testing with a mock adapter

```rust
// src/adapters/mock.rs
use crate::ports::{GitHubPort, PortError, Repo};
use async_trait::async_trait;

pub struct MockGitHubAdapter {
    repos: Vec<Repo>,
}

impl MockGitHubAdapter {
    pub fn new(repos: Vec<Repo>) -> Self {
        Self { repos }
    }

    pub fn empty() -> Self {
        Self { repos: vec![] }
    }
}

#[async_trait]
impl GitHubPort for MockGitHubAdapter {
    async fn list_repos(&self, _owner: &str) -> Result<Vec<Repo>, PortError> {
        Ok(self.repos.clone())
    }

    async fn get_repo(&self, _owner: &str, name: &str) -> Result<Repo, PortError> {
        self.repos
            .iter()
            .find(|r| r.full_name.ends_with(&format!("/{name}")))
            .cloned()
            .ok_or_else(|| PortError::NotFound(name.to_string()))
    }
}
```

And the tests:

```rust
#[cfg(test)]
mod tests {
    use super::*;
    use crate::adapters::mock::MockGitHubAdapter;
    use crate::core::RepoScorer;
    use crate::ports::Repo;

    fn sample_repos() -> Vec<Repo> {
        vec![
            Repo {
                full_name: "org/popular".into(),
                stars: 5000,
                open_issues: 10,
                last_push: "2026-06-01T00:00:00Z".into(),
            },
            Repo {
                full_name: "org/troubled".into(),
                stars: 100,
                open_issues: 200,
                last_push: "2026-05-01T00:00:00Z".into(),
            },
            Repo {
                full_name: "org/tiny".into(),
                stars: 3,
                open_issues: 0,
                last_push: "2026-06-09T00:00:00Z".into(),
            },
        ]
    }

    #[tokio::test]
    async fn popular_repo_scores_highest() {
        let scorer = RepoScorer::new(MockGitHubAdapter::new(sample_repos()));
        let scores = scorer.score_org("org").await.unwrap();
        assert_eq!(scores[0].full_name, "org/popular");
    }

    #[tokio::test]
    async fn top_n_limits_results() {
        let scorer = RepoScorer::new(MockGitHubAdapter::new(sample_repos()));
        let top = scorer.top_n("org", 2).await.unwrap();
        assert_eq!(top.len(), 2);
    }

    #[tokio::test]
    async fn empty_org_returns_empty() {
        let scorer = RepoScorer::new(MockGitHubAdapter::empty());
        let scores = scorer.score_org("empty").await.unwrap();
        assert!(scores.is_empty());
    }

    #[tokio::test]
    async fn high_issues_penalize_score() {
        let scorer = RepoScorer::new(MockGitHubAdapter::new(sample_repos()));
        let scores = scorer.score_org("org").await.unwrap();
        let troubled = scores.iter().find(|s| s.full_name == "org/troubled").unwrap();
        let tiny = scores.iter().find(|s| s.full_name == "org/tiny").unwrap();
        // 100 stars + 200 issues should score lower than 3 stars + 0 issues
        assert!(troubled.score < tiny.score);
    }
}
```

No network. No `gh` binary required. No GitHub token. These tests run in microseconds and verify the actual scoring algorithm. You can simulate any scenario: empty orgs, repos with extreme star counts, repos with thousands of issues. Try getting reliable test coverage for that against the real GitHub API.

## What Spring does instead

Here's the equivalent structure in Spring Boot:

```java
// Port
public interface GitHubPort {
    List<Repo> listRepos(String owner);
    Repo getRepo(String owner, String name);
}

// Adapter
@Service
public class GhCliAdapter implements GitHubPort {
    @Override
    public List<Repo> listRepos(String owner) {
        ProcessBuilder pb = new ProcessBuilder(
            "gh", "api", "/orgs/" + owner + "/repos?per_page=100"
        );
        Process p = pb.start();
        // ... read stdout, parse JSON, map to Repo
    }
    // ...
}

// Core
@Service
public class RepoScorer {
    private final GitHubPort github;

    @Autowired
    public RepoScorer(GitHubPort github) {
        this.github = github;
    }

    public List<RepoScore> scoreOrg(String owner) {
        // ... scoring logic
    }
}
```

The code structure looks similar. The difference is what happens when you run it.

At startup, Spring Boot:

1. Scans every class on the classpath for `@Component`, `@Service`, `@Repository`, `@Controller` annotations
2. Builds a `BeanDefinition` for each, reading constructor parameters via `java.lang.reflect`
3. Resolves the dependency graph - `RepoScorer` needs a `GitHubPort`, `GhCliAdapter` implements `GitHubPort`, match found
4. Instantiates beans in topological order, calling constructors via reflection
5. Wraps some beans in CGLIB proxies for AOP (transactions, security, caching)
6. Stores everything in the `ApplicationContext`

This takes [real time](https://www.javacodegeeks.com/2025/03/optimize-spring-boot-startup-time-tips-techniques.html). A typical Spring Boot app scans hundreds of classes at startup. Large apps can take 10+ seconds just for DI wiring. Spring Boot 3.5 improved this dramatically with ahead-of-time processing, and GraalVM native images eliminate most of it, but the default path is still runtime reflection.

Errors surface at startup too. Forget to annotate `GhCliAdapter` with `@Service`? You get a `NoSuchBeanDefinitionException` when Spring tries to inject `GitHubPort` into `RepoScorer`. Have two implementations of `GitHubPort`? You get `NoUniqueBeanDefinitionException` unless you add `@Qualifier`. These are runtime errors. Your code compiles fine.

## What NestJS does instead

```typescript
// Port (just a TypeScript interface - erased at runtime)
export interface GitHubPort {
    listRepos(owner: string): Promise<Repo[]>;
    getRepo(owner: string, name: string): Promise<Repo>;
}

// Adapter
@Injectable()
export class GhCliAdapter implements GitHubPort {
    async listRepos(owner: string): Promise<Repo[]> {
        const { stdout } = await exec(`gh api /orgs/${owner}/repos`);
        return JSON.parse(stdout).map(toRepo);
    }
    // ...
}

// Core
@Injectable()
export class RepoScorer {
    constructor(@Inject('GITHUB_PORT') private github: GitHubPort) {}

    async scoreOrg(owner: string): Promise<RepoScore[]> {
        // ... scoring logic
    }
}

// Module wiring
@Module({
    providers: [
        { provide: 'GITHUB_PORT', useClass: GhCliAdapter },
        RepoScorer,
    ],
})
export class AppModule {}
```

Notice the string token `'GITHUB_PORT'`. TypeScript interfaces are erased at runtime - they don't exist in JavaScript. NestJS can't look up "the thing that implements `GitHubPort`" because that type information is gone after compilation. Instead, you use string or symbol tokens and `@Inject()` decorators to tell the container which provider to use.

At startup, `NestFactory.create()`:

1. Reads module metadata attached by `@Module()` decorators
2. For each provider, reads constructor parameter types via `Reflect.getMetadata('design:paramtypes', ...)`
3. Matches `@Inject('GITHUB_PORT')` to the provider registered with `provide: 'GITHUB_PORT'`
4. Recursively resolves dependencies
5. Instantiates and caches in the container

Typo in the string token? Runtime crash. Forgot to register the provider in the module? Runtime crash. Circular dependency? Runtime crash (though NestJS detects this early and gives a decent error message).

## What Rust does instead

```rust
fn main() {
    let github = GhCliAdapter;
    let scorer = RepoScorer::new(github);
}
```

That's it. There is no step 2.

The compiler:

1. Checks that `GhCliAdapter` implements `GitHubPort` (the trait bound on `RepoScorer<G: GitHubPort>`)
2. Monomorphizes `RepoScorer<GhCliAdapter>` - generates a specialized version where `self.github.list_repos()` is a direct call to `<GhCliAdapter as GitHubPort>::list_repos`
3. Inlines if profitable

At runtime: nothing. No scanning. No reflection. No proxy generation. No container. The dependency is resolved, type-checked, and wired at compile time. If `GhCliAdapter` is missing a method from `GitHubPort`, the compiler points at the exact method signature and says what's wrong.

The function signature **is** the dependency declaration. `RepoScorer<G: GitHubPort>` says "I need something that implements `GitHubPort`." The compiler reads that and enforces it. Same job as `@Autowired`, but with zero runtime cost and compile-time error messages.

Spring needs `@Autowired` because Java's type system can't express "this constructor parameter must implement this interface, go find a matching bean." The constructor just takes `GitHubPort github` - Java doesn't do anything special with that signature. The framework uses reflection to read parameter types, search the bean registry, and inject. Rust's trait bounds are the language-level equivalent of that framework machinery.

## The cost side by side

| | Spring | NestJS | Rust |
|---|---|---|---|
| **When errors surface** | Startup (runtime) | Startup (runtime) | Compile time |
| **Startup overhead** | Classpath scan + reflection + proxy gen | Module scan + metadata read | Zero |
| **Error quality** | `NoSuchBeanDefinitionException` | "can't resolve dependencies of X" | "trait bound not satisfied" with exact location |
| **Runtime cost per call** | Proxy indirection (CGLIB) | None (direct reference) | None (monomorphized) or vtable (dyn) |
| **Reflection required** | Yes | Yes (reflect-metadata) | No |
| **Multiple implementations** | `@Qualifier` or `@Primary` | Named tokens | Type parameter or explicit wiring |

## Driving adapters: the other side of the hexagon

Everything above is a *driven* port - the application calls out to GitHub. But hexagonal architecture has driving ports too - the outside world calls into the application.

An HTTP handler is a driving adapter. It receives a request, converts it into a domain call, and returns the result:

```rust
// src/adapters/http.rs
use axum::extract::{Path, State};
use axum::http::StatusCode;
use axum::Json;
use std::sync::Arc;
use crate::core::{RepoScore, RepoScorer};
use crate::ports::GitHubPort;

pub async fn score_org_handler<G: GitHubPort + 'static>(
    Path(owner): Path<String>,
    State(scorer): State<Arc<RepoScorer<G>>>,
) -> Result<Json<Vec<RepoScore>>, (StatusCode, String)> {
    scorer.score_org(&owner).await
        .map(Json)
        .map_err(|e| (StatusCode::INTERNAL_SERVER_ERROR, e.to_string()))
}
```

The CLI is another driving adapter:

```rust
// src/adapters/cli.rs
use crate::core::RepoScorer;
use crate::ports::GitHubPort;

pub async fn run_cli<G: GitHubPort>(scorer: &RepoScorer<G>, owner: &str, top: usize) {
    match scorer.top_n(owner, top).await {
        Ok(scores) => {
            for s in &scores {
                println!("{:<40} score={:.2}  stars={}  issues={}",
                    s.full_name, s.score, s.stars, s.open_issues);
            }
        }
        Err(e) => eprintln!("error: {e}"),
    }
}
```

Both adapters call the same `RepoScorer` methods. The scorer doesn't know if it's being driven by an HTTP request, a CLI command, or a test. That's the hexagonal promise: swap any adapter on any side without touching the core.

## Enforcing boundaries with modules

Here's the full project layout:

```
src/
  ports.rs             # GitHubPort trait, Repo, PortError
  core.rs              # RepoScorer - depends only on ports
  adapters/
    mod.rs
    gh_cli.rs          # GhCliAdapter - depends on ports + tokio + serde
    mock.rs            # MockGitHubAdapter - depends on ports only
    http.rs            # Axum handler - depends on core + ports + axum
    cli.rs             # CLI runner - depends on core + ports
  main.rs              # Wiring - depends on everything
```

The dependency flow:

- `ports.rs` imports nothing from the project. Only `async_trait` and `thiserror`.
- `core.rs` imports from `ports`. Never from `adapters`.
- Each adapter imports from `ports` (and optionally `core`). Never from other adapters.
- `main.rs` imports from everywhere - it's the composition root.

In Java, enforcing this boundary requires tools like [ArchUnit](https://www.archunit.org/) that check import rules at test time. In Rust, the module system does it naturally. If `core.rs` tries to `use crate::adapters::gh_cli::GhCliAdapter`, the dependency is visible in the import. You'd spot it in code review. And if you want hard enforcement, split the hexagon into separate crates in a Cargo workspace:

```toml
# Cargo.toml (workspace)
[workspace]
members = ["ports", "core", "adapters", "app"]

# ports/Cargo.toml - no internal dependencies
[dependencies]
async-trait = "0.1"
thiserror = "2"

# core/Cargo.toml - depends only on ports
[dependencies]
ports = { path = "../ports" }

# adapters/Cargo.toml - depends on ports, external crates
[dependencies]
ports = { path = "../ports" }
serde = { version = "1", features = ["derive"] }
serde_json = "1"
tokio = { version = "1", features = ["process"] }
```

Now it's a compile error for `core` to import anything from `adapters`. Cargo enforces the architecture. No discipline required, no linter rules, no ArchUnit. The build system literally won't let you break the dependency direction.

## When this is overkill

Hexagonal architecture adds structure. Structure has a cost: more files, more types, more indirection to trace through. For some projects that cost isn't worth it.

The same heuristics from the [adapter pattern post](/blog/the-adapter-pattern-in-rust-wrapping-external-apis/) apply:

- **Small CLIs** that do one thing with one external dependency. If you're writing a quick tool that pulls data from GitHub and prints it, the full hexagon is ceremony.
- **Prototypes** where you're still figuring out the domain. Adding ports and adapters locks in interfaces early. Better to explore with direct calls, then extract boundaries once the domain stabilizes.
- **Stable, internal dependencies** you fully control. If the "external" service is your team's own microservice with a stable API, wrapping it behind a port might be unnecessary indirection.

The judgment call: do you have more than one adapter for any port (real + mock counts as two)? Does more than one driving adapter call into the core? If yes to either, hexagonal pays for itself in testability alone. If no to both, plain constructor injection is simpler.

## The bottom line

Spring invented `@Autowired` because Java needed help. The language has no trait bounds, no generics-based monomorphization, no way to express "this constructor requires something that implements this interface" in a way the compiler can enforce. The framework fills that gap with reflection and runtime resolution.

NestJS invented `@Injectable()` for the same reason, compounded by TypeScript's erasure of interfaces at compile time. The framework uses `reflect-metadata` to recover type information that TypeScript threw away.

Rust doesn't have that gap. A trait bound is a compile-time dependency declaration. An impl block is a compile-time adapter registration. The `new()` function is your constructor injection point. The `main()` function is your composition root. No scanning, no reflection, no container, no startup cost.

The hexagonal architecture pattern is the same across all three languages. The difference is that in Rust, the language already provides the machinery that Spring and NestJS need frameworks to deliver. Port = trait. Adapter = impl. DI container = the compiler.
