+++
title = "The adapter pattern in Rust - wrapping external APIs"
date = 2025-01-29
description = "Wrap external crates and HTTP APIs behind your own traits so the world can change without your code changing. Domain error mapping, mockable tests, and the line between a thin wrapper and a rich adapter."

[taxonomies]
tags = ["rust", "architecture", "design-patterns", "testing"]
+++

External APIs change. Your code should not. That is the entire pitch for the adapter pattern, and it is the single most useful piece of architectural advice I can give to anyone past their first Rust project. The day GitHub deprecates a v3 endpoint, the day `octocrab` releases a 0.x version that renames half of its types, the day Stripe moves a field from the top level of a webhook into a nested object - on those days, you want to edit one file in your codebase, not seventeen.

The adapter pattern in Rust is a trait that defines the contract you wish the outside world had, plus an implementation that translates between that contract and the contract you actually got. It is not a framework. It is not a crate. It is two pages of code that pay for themselves the first time a dependency releases a breaking change.

<!-- more -->

## The shape of the problem

Pick any Rust application that talks to a third party. Watch where the third party's types leak into your code. A typical first pass looks like this:

```rust
use octocrab::Octocrab;
use octocrab::models::issues::Issue;

pub async fn close_stale_issues(client: &Octocrab, repo: &str) -> octocrab::Result<()> {
    let (owner, name) = repo.split_once('/').unwrap();
    let issues: Vec<Issue> = client
        .issues(owner, name)
        .list()
        .state(octocrab::params::State::Open)
        .send()
        .await?
        .items;

    for issue in issues {
        if is_stale(&issue) {
            client.issues(owner, name).update(issue.number).state(octocrab::params::State::Closed).send().await?;
        }
    }
    Ok(())
}
```

That function does its job. It also bolts your business logic to `octocrab` in three places: the parameter type, the return type, and the `is_stale` helper that almost certainly takes `&Issue` somewhere up the call stack. Every module that imports `close_stale_issues` now transitively depends on `octocrab`. Every test that wants to exercise the staleness logic needs to spin up a fake HTTP server, because there is no seam to mock.

Worse, when `octocrab` 0.40 renames `params::State` to `params::IssueState` (it has done equivalent renames before), every file that mentions `State::Open` breaks. The compiler will help you find them, but the diff will sprawl across your codebase for a problem that has nothing to do with your domain.

## The adapter, in three pieces

An adapter has three parts: a domain type, a domain error, and a trait that operates on them. The implementation comes fourth, and it is usually the most boring piece.

```rust
// Domain types - what you wish you had received from the outside world.
#[derive(Debug, Clone)]
pub struct Issue {
    pub number: u64,
    pub title: String,
    pub body: String,
    pub state: IssueState,
    pub author: String,
}

#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum IssueState {
    Open,
    Closed,
}

// Domain error - failure modes your callers care about, not transport details.
#[derive(Debug, thiserror::Error)]
pub enum IssueError {
    #[error("issue not found")]
    NotFound,
    #[error("rate limited, retry after {retry_after_secs}s")]
    RateLimited { retry_after_secs: u64 },
    #[error("authentication failed")]
    Unauthorized,
    #[error("network error: {0}")]
    Network(String),
    #[error("upstream error: {0}")]
    Upstream(String),
}

// The trait - the contract you wish you had.
#[async_trait::async_trait]
pub trait IssueTracker: Send + Sync {
    async fn get(&self, repo: &str, number: u64) -> Result<Issue, IssueError>;
    async fn list_open(&self, repo: &str) -> Result<Vec<Issue>, IssueError>;
    async fn close(&self, repo: &str, number: u64) -> Result<(), IssueError>;
}
```

Three things to notice. First, the domain `Issue` is not a subset of the GitHub issue. It is the shape your application wants. If you do not care about labels, milestones, or assignees, they are not on the struct. If a different tracker (Jira, Linear, Forgejo) gives you something that looks roughly like an issue, you can implement the same trait for it.

Second, `IssueError` does not mention HTTP, JSON, or any specific crate. `RateLimited` is a domain concept (you may want to retry); `Network` is a domain concept (the user is offline); `Unauthorized` is a domain concept (the token is wrong). Whether the underlying crate threw a `reqwest::Error::Decode` or an `octocrab::Error::GitHub` is not relevant to anyone above the adapter.

Third, the trait uses `&self`. It is `Send + Sync`. That makes it cheap to share across tasks via `Arc<dyn IssueTracker>`. It also makes object safety trivial - no generic methods, no `Self` in return position, no associated types. You can stuff it in a `Box` or behind a router without any incantations.

## Building a real adapter

Here is the implementation against the GitHub REST API using `reqwest` directly. I am skipping `octocrab` for this post because writing the HTTP calls by hand makes the translation step explicit. In production you would usually wrap a higher level crate.

```rust
use reqwest::{Client, StatusCode};
use serde::Deserialize;

pub struct GitHubAdapter {
    http: Client,
    token: String,
    base_url: String,
}

impl GitHubAdapter {
    pub fn new(token: impl Into<String>) -> Self {
        Self {
            http: Client::builder()
                .user_agent("my-app/1.0")
                .build()
                .expect("client init"),
            token: token.into(),
            base_url: "https://api.github.com".into(),
        }
    }
}

// Wire types - never escape this module.
#[derive(Deserialize)]
struct GhIssue {
    number: u64,
    title: String,
    #[serde(default)]
    body: Option<String>,
    state: String,
    user: GhUser,
}

#[derive(Deserialize)]
struct GhUser {
    login: String,
}

impl From<GhIssue> for Issue {
    fn from(g: GhIssue) -> Self {
        Issue {
            number: g.number,
            title: g.title,
            body: g.body.unwrap_or_default(),
            state: if g.state == "open" { IssueState::Open } else { IssueState::Closed },
            author: g.user.login,
        }
    }
}
```

The `GhIssue` and `GhUser` types are private to the adapter module. They map field-for-field to GitHub's JSON. Nothing in the rest of your application ever sees them. The `From` impl is the translation layer: this is where you decide that GitHub's open-text `state` becomes a clean enum, that a missing body becomes an empty string, that the nested `user.login` flattens to a top-level `author`.

The trait impl is mechanical:

```rust
#[async_trait::async_trait]
impl IssueTracker for GitHubAdapter {
    async fn get(&self, repo: &str, number: u64) -> Result<Issue, IssueError> {
        let url = format!("{}/repos/{}/issues/{}", self.base_url, repo, number);
        let resp = self.http
            .get(&url)
            .bearer_auth(&self.token)
            .send()
            .await
            .map_err(map_transport)?;

        match resp.status() {
            StatusCode::OK => {
                let gh: GhIssue = resp.json().await.map_err(map_transport)?;
                Ok(gh.into())
            }
            StatusCode::NOT_FOUND => Err(IssueError::NotFound),
            StatusCode::UNAUTHORIZED | StatusCode::FORBIDDEN => Err(IssueError::Unauthorized),
            s if s == StatusCode::TOO_MANY_REQUESTS || (s == StatusCode::FORBIDDEN && has_rate_limit(&resp)) => {
                let retry_after = resp.headers()
                    .get("retry-after")
                    .and_then(|v| v.to_str().ok())
                    .and_then(|v| v.parse().ok())
                    .unwrap_or(60);
                Err(IssueError::RateLimited { retry_after_secs: retry_after })
            }
            other => Err(IssueError::Upstream(format!("status {}", other))),
        }
    }

    async fn list_open(&self, repo: &str) -> Result<Vec<Issue>, IssueError> {
        let url = format!("{}/repos/{}/issues?state=open&per_page=100", self.base_url, repo);
        let resp = self.http
            .get(&url)
            .bearer_auth(&self.token)
            .send()
            .await
            .map_err(map_transport)?
            .error_for_status()
            .map_err(|e| IssueError::Upstream(e.to_string()))?;
        let items: Vec<GhIssue> = resp.json().await.map_err(map_transport)?;
        Ok(items.into_iter().map(Into::into).collect())
    }

    async fn close(&self, repo: &str, number: u64) -> Result<(), IssueError> {
        let url = format!("{}/repos/{}/issues/{}", self.base_url, repo, number);
        let body = serde_json::json!({ "state": "closed" });
        self.http
            .patch(&url)
            .bearer_auth(&self.token)
            .json(&body)
            .send()
            .await
            .map_err(map_transport)?
            .error_for_status()
            .map_err(|e| IssueError::Upstream(e.to_string()))?;
        Ok(())
    }
}

fn map_transport(e: reqwest::Error) -> IssueError {
    if e.is_timeout() || e.is_connect() {
        IssueError::Network(e.to_string())
    } else {
        IssueError::Upstream(e.to_string())
    }
}

fn has_rate_limit(resp: &reqwest::Response) -> bool {
    resp.headers().get("x-ratelimit-remaining").map(|v| v.as_bytes() == b"0").unwrap_or(false)
}
```

The interesting work is in the status code matching and the `map_transport` helper. GitHub uses `403 Forbidden` for both auth failures and rate limits, distinguished by an `x-ratelimit-remaining: 0` header. That is exactly the kind of API quirk that you do not want leaking out of the adapter. Your business logic should not have to know that a 403 sometimes means "wait" and sometimes means "you cannot do this."

This is also where you make policy decisions about what counts as a "transient" failure. `is_timeout()` and `is_connect()` go to `Network` because callers may want to retry. Decode errors go to `Upstream` because retrying will not help.

## Why this makes tests bearable

Now write a service that uses the trait:

```rust
use std::sync::Arc;

pub struct StaleIssueCleaner<T: IssueTracker> {
    tracker: Arc<T>,
    threshold_days: u32,
}

impl<T: IssueTracker> StaleIssueCleaner<T> {
    pub async fn run(&self, repo: &str) -> Result<u32, IssueError> {
        let issues = self.tracker.list_open(repo).await?;
        let mut closed = 0;
        for issue in issues {
            if is_stale(&issue, self.threshold_days) {
                self.tracker.close(repo, issue.number).await?;
                closed += 1;
            }
        }
        Ok(closed)
    }
}

fn is_stale(issue: &Issue, _threshold_days: u32) -> bool {
    issue.title.starts_with("[stale]")
}
```

To test `run`, you need a fake `IssueTracker`. You do not need a mock framework, you do not need an HTTP server, you do not need internet access. You write a struct.

```rust
#[cfg(test)]
mod tests {
    use super::*;
    use std::sync::Mutex;

    #[derive(Default)]
    struct FakeTracker {
        issues: Mutex<Vec<Issue>>,
        closed: Mutex<Vec<u64>>,
    }

    #[async_trait::async_trait]
    impl IssueTracker for FakeTracker {
        async fn get(&self, _repo: &str, number: u64) -> Result<Issue, IssueError> {
            self.issues.lock().unwrap().iter().find(|i| i.number == number).cloned().ok_or(IssueError::NotFound)
        }
        async fn list_open(&self, _repo: &str) -> Result<Vec<Issue>, IssueError> {
            Ok(self.issues.lock().unwrap().clone())
        }
        async fn close(&self, _repo: &str, number: u64) -> Result<(), IssueError> {
            self.closed.lock().unwrap().push(number);
            Ok(())
        }
    }

    #[tokio::test]
    async fn closes_stale_issues() {
        let tracker = Arc::new(FakeTracker::default());
        tracker.issues.lock().unwrap().push(Issue {
            number: 42, title: "[stale] old bug".into(), body: String::new(),
            state: IssueState::Open, author: "alice".into(),
        });
        let cleaner = StaleIssueCleaner { tracker: Arc::clone(&tracker), threshold_days: 30 };
        let n = cleaner.run("foo/bar").await.unwrap();
        assert_eq!(n, 1);
        assert_eq!(tracker.closed.lock().unwrap().as_slice(), &[42]);
    }
}
```

That test runs in microseconds, has zero flakes, and exercises the exact code path your production service does. The same structure works for testing rate limit handling: write a `FlakyTracker` whose `list_open` returns `Err(IssueError::RateLimited { retry_after_secs: 1 })` on the first call and a list on the second. You are testing your retry logic, not GitHub's HTTP semantics.

This is the part that pays back the cost of the adapter. Without it, your only option is `wiremock` or a recorded HTTP cassette. Both work. Both are slower, more brittle, and harder to reason about than a 30-line fake.

## Thin wrapper or rich adapter

There are two flavors of adapter, and the difference matters.

A **thin wrapper** mirrors the upstream API one method at a time, just translating types. If GitHub has 80 endpoints, the adapter has 80 methods. The advantage is that callers can use anything the upstream offers; the disadvantage is that callers also see anything the upstream offers, including its inconsistencies. Thin wrappers are useful when you are building a library that other developers will compose, or when you genuinely use most of the upstream surface.

A **rich adapter** exposes only the operations your domain actually needs, often combining multiple upstream calls into one method. `close_stale_issues(repo)` is a rich adapter method. So is `assign_to_least_loaded_reviewer(pr)`. The advantage is that your business logic gets to read like business logic, not like a sequence of HTTP calls; the disadvantage is that adding a new operation requires adding a new method to the trait.

Most real applications want a rich adapter. You are not building a GitHub client; you are building a thing that happens to talk to GitHub. The trait should reflect what your application does, not what the API offers. If you find yourself adding `get_raw_response_metadata()` to the trait, you are leaking the abstraction.

A useful tell: if you swap out GitHub for Forgejo and the trait barely changes, your adapter is at the right level of abstraction. If swapping requires renaming half the methods, you have built a thin wrapper and called it an adapter.

## Adapter vs facade

The two patterns get confused because they look identical from outside. The difference is intent.

A **facade** simplifies a complicated thing. The thing already does what you want; you just wrap it to hide the complexity. `std::fs::read_to_string` is a facade over a sequence of `open`, `read`, and `close` syscalls. The underlying API and the facade serve the same purpose; the facade is just nicer to use.

An **adapter** translates between two contracts. The thing on the other side of the adapter does what you need, but in the wrong shape, with the wrong vocabulary, with the wrong error model. The adapter exists because the contract you have and the contract you want are not the same.

In practice the line blurs. `GitHubAdapter` is technically both: it simplifies the underlying HTTP machinery (facade) and translates GitHub-isms into your domain (adapter). When you see "adapter" in a Rust codebase, treat it as "facade plus translation." When you see "wrapper," reach for context, because the term is doing too much work.

## When to skip it

Adapters are not free. You write a trait, an impl, fake implementations for tests, and the boilerplate of `From` conversions. If you are building a one-shot script, a CLI that calls one endpoint, or a prototype that will be thrown away in a week, this is overkill. Use the upstream crate directly, accept that the test is going to hit the real network, and move on.

The cost is also higher in Rust than in dynamic languages because the trait must be designed up front. You cannot start with a concrete type and "extract an interface" later without touching every call site. If you are unsure whether a dependency will stick around long enough to be worth wrapping, write the direct version, and refactor when you need a second implementation or a test seam.

The signals that say "wrap this":

- More than one call site uses the dependency.
- You expect to mock it in tests.
- The vendor has a history of breaking changes.
- The error type from the crate is wider than the failure modes you care about.
- You might swap implementations later (testing, a self-hosted variant, a different vendor).

If three of those apply, write the adapter. If none do, do not.

## What you get back

The adapter is a one-time tax. After it is in place, the rest of the codebase works in your domain language. Errors mean what they say. Tests run offline. Upgrading `reqwest` from 0.11 to 0.12 touches one file. Replacing GitHub with Forgejo for a self-hosted customer adds a `ForgejoAdapter` module and changes nothing else. Your `main.rs` picks an `Arc<dyn IssueTracker>` based on config, and the rest of the application never knew there was a choice.

That last property - the rest of the application never knowing - is the goal. Not portability for its own sake, not abstraction for the sake of looking professional, but the simple discipline of keeping the outside world's instabilities outside your codebase. The compiler enforces a lot of things in Rust. It cannot enforce architectural boundaries. The adapter pattern is how you draw them by hand, in a way the compiler can then help you maintain.
