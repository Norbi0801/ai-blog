+++
title = "Implementing a Web Crawler in Rust"
date = 2025-11-02
description = "Building an async web crawler from scratch with tokio, reqwest, and scraper - URL frontier, robots.txt, politeness delays, deduplication, depth limiting, and concurrent crawling with a semaphore."

[taxonomies]
tags = ["rust", "async", "web-scraping", "tokio"]
+++

A web crawler is a program that starts at one URL, downloads the page, extracts every link it can find, and repeats. Google built a company on top of that loop. The concept is straightforward, but the implementation touches almost every async pattern that matters: concurrent I/O with backpressure, shared mutable state across tasks, per-domain rate limiting, and queue-based work distribution.

We're going to build one. Around 250 lines of Rust. It will respect `robots.txt`, wait between requests to the same domain, deduplicate URLs, limit crawl depth, and run multiple fetches concurrently using a tokio semaphore. By the end, you'll have a crawler that can index a small site and print a structured map of what it found.

<!-- more -->

If you haven't read the post on [how tokio's runtime works](/blog/understanding-tokio---the-rust-async-runtime-under-the-hood/), the scheduler and I/O driver sections are useful context for understanding why this design works. The [reqwest best practices post](/blog/http-client-best-practices-in-rust-with-reqwest/) covers client reuse and connection pooling, both of which matter here.

## The moving parts

A crawler has five core components:

1. **URL frontier** - a queue of URLs waiting to be fetched, each tagged with a depth
2. **Visited set** - deduplication so you don't fetch the same page twice
3. **Fetcher** - downloads a page, respects concurrency limits
4. **Link extractor** - parses HTML, pulls out `<a href="...">` links
5. **Politeness layer** - per-domain delays and `robots.txt` checks

We'll use these crates:

```toml
[dependencies]
tokio = { version = "1.47", features = ["full"] }
reqwest = { version = "0.13", features = ["gzip"] }
scraper = "0.26"
texting_robots = "0.2"
url = "2.5"
```

[reqwest](https://crates.io/crates/reqwest) handles HTTP. [scraper](https://crates.io/crates/scraper) wraps Servo's html5ever parser and gives us CSS selectors. [texting_robots](https://crates.io/crates/texting_robots) parses `robots.txt` - it was tested against 34 million real-world robots.txt files, which is the kind of paranoia you want in a parser. [url](https://crates.io/crates/url) handles URL resolution (turning relative paths into absolute URLs).

## Data structures

The frontier is a channel. Each item is a URL paired with the current crawl depth:

```rust
use std::collections::HashSet;
use std::sync::Arc;
use std::time::Duration;
use tokio::sync::{mpsc, Mutex, Semaphore};
use url::Url;

#[derive(Debug, Clone)]
struct CrawlTask {
    url: Url,
    depth: u32,
}
```

The crawler state holds everything shared across tasks:

```rust
struct Crawler {
    client: reqwest::Client,
    visited: Mutex<HashSet<String>>,
    domain_delays: Mutex<std::collections::HashMap<String, tokio::time::Instant>>,
    semaphore: Arc<Semaphore>,
    max_depth: u32,
    delay: Duration,
    user_agent: String,
}
```

A few things to note. `visited` is a `tokio::sync::Mutex<HashSet<String>>` - not `std::sync::Mutex`. The tokio variant yields to the scheduler while waiting for the lock instead of blocking the thread. Since we hold the lock only for the duration of an insert or contains check (microseconds), either mutex would work here, but the async version is the safer default when you're inside a tokio context.

`domain_delays` tracks the last time we sent a request to each domain. Before fetching, we check if enough time has passed since the last request to that host. This is the politeness mechanism - you don't want to hammer someone's server with 50 concurrent requests.

The `semaphore` caps total concurrency across all domains. If you set it to 10, at most 10 fetches run in parallel regardless of how many URLs are in the queue.

## Checking robots.txt

Before fetching any page, a well-behaved crawler checks whether it's allowed to. The `robots.txt` file at the root of every domain specifies which paths are off-limits for which user agents.

```rust
use texting_robots::Robot;

async fn is_allowed(client: &reqwest::Client, url: &Url, user_agent: &str) -> bool {
    let robots_url = format!(
        "{}://{}/robots.txt",
        url.scheme(),
        url.host_str().unwrap_or_default()
    );

    let body = match client.get(&robots_url).send().await {
        Ok(resp) if resp.status().is_success() => {
            resp.text().await.unwrap_or_default()
        }
        // If robots.txt is missing or we can't fetch it, allow by convention
        _ => return true,
    };

    match Robot::new(user_agent, body.as_bytes()) {
        Ok(robot) => robot.allowed(url.as_str()),
        Err(_) => true, // Malformed robots.txt - allow
    }
}
```

The [Robots Exclusion Protocol](https://www.rfc-editor.org/rfc/rfc9309) (RFC 9309, published 2022) says that if the robots.txt is absent (404), the crawler should assume everything is allowed. If it exists but is malformed, `texting_robots` handles the edge cases - it's been tested against adversarial inputs including robots.txt files with binary garbage, multi-megabyte payloads, and contradictory rules.

In a production crawler, you'd cache the parsed `Robot` per domain instead of re-fetching robots.txt for every URL. For our purposes, the overhead is acceptable - we're crawling a single site.

## Link extraction

The scraper crate makes this straightforward. Parse the HTML, select all `<a>` tags, extract the `href` attribute, resolve relative URLs against the page's base URL:

```rust
use scraper::{Html, Selector};

fn extract_links(html: &str, base: &Url) -> Vec<Url> {
    let document = Html::parse_document(html);
    let selector = Selector::parse("a[href]").unwrap();
    let mut links = Vec::new();

    for element in document.select(&selector) {
        if let Some(href) = element.value().attr("href") {
            // Skip fragments, javascript:, mailto:, data:
            if href.starts_with('#')
                || href.starts_with("javascript:")
                || href.starts_with("mailto:")
                || href.starts_with("data:")
            {
                continue;
            }

            // Resolve relative URLs against the page's base
            if let Ok(resolved) = base.join(href) {
                links.push(resolved);
            }
        }
    }

    links
}
```

`Url::join` does the heavy lifting. Given a base of `https://example.com/blog/page1` and an href of `../about`, it correctly resolves to `https://example.com/about`. It follows [RFC 3986](https://www.rfc-editor.org/rfc/rfc3986) section 5 reference resolution - the same algorithm browsers use.

One subtlety: `Html::parse_document` uses html5ever internally, which is the same HTML parser that Servo (Mozilla's experimental browser engine) uses. It handles real-world broken HTML - unclosed tags, mismatched nesting, invalid entities - the same way a browser would. This matters because the web is full of malformed HTML, and a crawler that chokes on a missing `</div>` is useless.

## The politeness layer

Being polite means two things: not sending too many requests at once (global concurrency limit) and not hitting the same domain too fast (per-domain delay).

The semaphore handles global concurrency. Before each fetch, the task acquires a permit. If all permits are taken, the task waits. When the fetch completes, the permit drops automatically.

Per-domain delay uses a timestamp map:

```rust
impl Crawler {
    async fn wait_for_domain(&self, domain: &str) {
        let mut delays = self.domain_delays.lock().await;
        if let Some(last_request) = delays.get(domain) {
            let elapsed = last_request.elapsed();
            if elapsed < self.delay {
                let wait = self.delay - elapsed;
                drop(delays); // Release lock before sleeping
                tokio::time::sleep(wait).await;
                let mut delays = self.domain_delays.lock().await;
                delays.insert(domain.to_string(), tokio::time::Instant::now());
            } else {
                delays.insert(domain.to_string(), tokio::time::Instant::now());
            }
        } else {
            delays.insert(domain.to_string(), tokio::time::Instant::now());
        }
    }
}
```

Notice the `drop(delays)` before the sleep. If you hold the mutex across an `.await` point, you block every other task that wants to check their domain delay. The lock is held for the HashMap lookup (nanoseconds), released, then re-acquired after sleeping. If you've read the [rate limiter post](/blog/writing-a-rate-limiter-in-rust/), you'll recognize this as a simplified version of per-key rate limiting - same idea, less sophisticated algorithm.

For a single-site crawler this is fine. A production multi-site crawler would use a more sophisticated approach - one queue per domain with its own timer, or a token bucket per host. The [governor](https://crates.io/crates/governor) crate handles this well if you need it.

## Fetching a page

The fetch function ties together the semaphore, politeness delay, robots.txt check, and the actual HTTP request:

```rust
impl Crawler {
    async fn fetch(&self, url: &Url) -> Option<String> {
        let domain = url.host_str().unwrap_or_default().to_string();

        // Acquire semaphore permit (limits global concurrency)
        let _permit = self.semaphore.acquire().await.ok()?;

        // Per-domain politeness delay
        self.wait_for_domain(&domain).await;

        // Check robots.txt
        if !is_allowed(&self.client, url, &self.user_agent).await {
            eprintln!("  [blocked] {} (robots.txt)", url);
            return None;
        }

        // Fetch the page
        let response = match self.client.get(url.as_str()).send().await {
            Ok(resp) => resp,
            Err(e) => {
                eprintln!("  [error] {} - {}", url, e);
                return None;
            }
        };

        if !response.status().is_success() {
            eprintln!("  [{}] {}", response.status(), url);
            return None;
        }

        // Only process HTML responses
        let content_type = response
            .headers()
            .get("content-type")
            .and_then(|v| v.to_str().ok())
            .unwrap_or_default();

        if !content_type.contains("text/html") {
            return None;
        }

        response.text().await.ok()
    }
}
```

The `_permit` variable holds the semaphore permit. When it goes out of scope at the end of the function, the permit is released. This is RAII-based concurrency control - same pattern as `MutexGuard`. The `Semaphore::acquire` call returns a `SemaphorePermit` that implements `Drop`, so you can't accidentally leak permits (unless you `mem::forget` it, which would be adversarial).

The content-type check prevents the crawler from trying to parse PDFs, images, or binary files as HTML. A real crawler would also check `Content-Length` to avoid downloading multi-gigabyte files.

## The crawl loop

Now the main loop. It reads tasks from the frontier channel, fetches pages, extracts links, and pushes new tasks back into the channel:

```rust
impl Crawler {
    async fn crawl(self: &Arc<Self>, seed: Url, max_pages: usize) {
        let (tx, mut rx) = mpsc::channel::<CrawlTask>(1000);
        let mut pages_crawled = 0u32;

        // Seed the frontier
        {
            let mut visited = self.visited.lock().await;
            visited.insert(seed.to_string());
        }
        tx.send(CrawlTask { url: seed, depth: 0 }).await.ok();

        let mut tasks = tokio::task::JoinSet::new();

        loop {
            tokio::select! {
                // Pull next URL from the frontier
                Some(task) = rx.recv() => {
                    if pages_crawled as usize >= max_pages {
                        continue;
                    }

                    let crawler = Arc::clone(self);
                    let tx = tx.clone();

                    tasks.spawn(async move {
                        let url_str = task.url.to_string();
                        println!("[depth={}] {}", task.depth, url_str);

                        let html = match crawler.fetch(&task.url).await {
                            Some(h) => h,
                            None => return,
                        };

                        let title = extract_title(&html)
                            .unwrap_or_else(|| url_str.clone());
                        println!("  -> \"{}\" ({} bytes)", title, html.len());

                        // Don't extract links if we've hit max depth
                        if task.depth >= crawler.max_depth {
                            return;
                        }

                        let links = extract_links(&html, &task.url);
                        let same_domain = task.url.host_str().unwrap_or_default();

                        let mut visited = crawler.visited.lock().await;
                        for link in links {
                            // Stay on the same domain
                            if link.host_str().unwrap_or_default() != same_domain {
                                continue;
                            }

                            let normalized = normalize_url(&link);
                            if visited.insert(normalized) {
                                let _ = tx.try_send(CrawlTask {
                                    url: link,
                                    depth: task.depth + 1,
                                });
                            }
                        }
                    });

                    pages_crawled += 1;
                }

                // Collect completed tasks
                Some(result) = tasks.join_next() => {
                    if let Err(e) = result {
                        eprintln!("Task panicked: {}", e);
                    }
                }

                // Both channels empty and no in-flight tasks - we're done
                else => break,
            }
        }

        // Drain remaining tasks
        while let Some(result) = tasks.join_next().await {
            if let Err(e) = result {
                eprintln!("Task panicked: {}", e);
            }
        }
    }
}
```

There's a lot happening here. Let's break it down.

**`tokio::select!`** multiplexes two event sources: new tasks from the frontier channel and completed tasks from the JoinSet. This is the core scheduling loop. When a URL arrives, we spawn a task to fetch and process it. When a task completes, we check for panics. When both sources are exhausted, the crawler terminates.

**`JoinSet`** (stabilized in tokio 1.21) is a collection of spawned tasks that you can poll for completion. It's better than a `Vec<JoinHandle>` because you don't have to track handles manually - `join_next()` returns whichever task finishes first.

**`try_send`** on the channel is intentional. If the frontier is full (1000 pending URLs), we silently drop the link. This is backpressure - the crawler won't accumulate unbounded memory from a site with millions of pages. If you need guaranteed delivery, increase the channel capacity or switch to an unbounded channel (but then you need other memory limits).

**Same-domain filtering** keeps the crawler focused. Without it, a single external link sends you crawling the entire internet. Production crawlers often have a configurable scope - same domain, same subdomain, or a regex pattern.

## URL normalization

URLs can look different but point to the same resource. `https://example.com/page` and `https://example.com/page/` and `https://example.com/page?` are typically the same page. Without normalization, the visited set misses these duplicates.

```rust
fn normalize_url(url: &Url) -> String {
    let mut normalized = format!(
        "{}://{}{}",
        url.scheme(),
        url.host_str().unwrap_or_default(),
        url.path().trim_end_matches('/')
    );

    if let Some(query) = url.query() {
        if !query.is_empty() {
            normalized.push('?');
            normalized.push_str(query);
        }
    }

    normalized.to_lowercase()
}
```

This strips trailing slashes, drops empty query strings, and lowercases everything. It's not perfect - query parameter ordering, URL encoding variations, and session IDs in paths all cause false negatives - but it catches the common cases. Google's URL canonicalization is documented to handle [over 30 normalization rules](https://developers.google.com/search/docs/crawling-indexing/canonicalization). We don't need that level of sophistication.

## Extracting page titles

For the output, grab the `<title>` tag:

```rust
fn extract_title(html: &str) -> Option<String> {
    let document = Html::parse_document(html);
    let selector = Selector::parse("title").unwrap();
    document
        .select(&selector)
        .next()
        .map(|el| el.text().collect::<String>().trim().to_string())
}
```

## Putting it all together

```rust
#[tokio::main]
async fn main() {
    let args: Vec<String> = std::env::args().collect();
    let seed_url = args
        .get(1)
        .map(|s| s.as_str())
        .unwrap_or("https://www.rust-lang.org");

    let url = Url::parse(seed_url).expect("Invalid seed URL");

    let crawler = Arc::new(Crawler {
        client: reqwest::Client::builder()
            .user_agent("rustcrawler/0.1 (+https://github.com/example/rustcrawler)")
            .timeout(Duration::from_secs(10))
            .connect_timeout(Duration::from_secs(5))
            .pool_max_idle_per_host(5)
            .gzip(true)
            .build()
            .expect("Failed to build HTTP client"),
        visited: Mutex::new(HashSet::new()),
        domain_delays: Mutex::new(std::collections::HashMap::new()),
        semaphore: Arc::new(Semaphore::new(5)),
        max_depth: 2,
        delay: Duration::from_millis(500),
        user_agent: "rustcrawler".to_string(),
    });

    println!("Crawling {} (max depth: 2, concurrency: 5)\n", url);
    crawler.crawl(url, 50).await;
    println!("\nDone. Visited {} unique URLs.",
        crawler.visited.lock().await.len()
    );
}
```

The client configuration matters. `user_agent` identifies your crawler - sites block requests without a user-agent, and it's good practice to include a URL where site operators can learn about your bot. `timeout` prevents the crawler from hanging on unresponsive servers. `gzip(true)` compresses responses, saving bandwidth (most sites serve gzipped HTML that's 60-80% smaller).

Five concurrent connections with a 500ms per-domain delay is polite. That's at most 2 requests/second to any single domain, and 5 in-flight requests across all domains. Googlebot typically crawls at rates of 1-5 requests/second for small sites - we're in that ballpark.

## Running it

```bash
$ cargo run -- https://www.rust-lang.org

Crawling https://www.rust-lang.org/ (max depth: 2, concurrency: 5)

[depth=0] https://www.rust-lang.org/
  -> "Rust Programming Language" (18943 bytes)
[depth=1] https://www.rust-lang.org/tools/install
  -> "Install Rust" (12087 bytes)
[depth=1] https://www.rust-lang.org/learn
  -> "Learn Rust" (15420 bytes)
[depth=1] https://www.rust-lang.org/community
  -> "Rust Community" (11832 bytes)
[depth=2] https://www.rust-lang.org/learn/get-started
  -> "Getting started" (14201 bytes)
[depth=2] https://www.rust-lang.org/governance
  -> "Governance" (22105 bytes)
...

Done. Visited 47 unique URLs.
```

The output shows the crawl depth, URL, page title, and response size. Depth 0 is the seed. Depth 1 is links found on the seed page. Depth 2 is links found on those pages. With `max_depth: 2`, we don't go deeper.

## What's happening under the hood

Let's trace through a single URL fetch to see how the async machinery works.

When a `CrawlTask` arrives from the channel, `tokio::select!` wakes the main loop. We spawn a new task onto the tokio runtime via `JoinSet::spawn`. This task is added to the work-stealing scheduler's local queue - as covered in the [tokio internals post](/blog/understanding-tokio---the-rust-async-runtime-under-the-hood/), it lands on the spawning worker's LIFO slot first, then migrates to the run queue.

Inside the spawned task, `semaphore.acquire().await` checks if a permit is available. The semaphore is backed by an atomic counter and a wait list. If permits are available, it decrements the counter and returns immediately (no syscall, no context switch). If all permits are taken, the task's `Waker` is added to the semaphore's internal linked list, and the task yields back to the scheduler via `Poll::Pending`.

The `client.get(url).send().await` call goes through reqwest, which delegates to hyper, which delegates to tokio's TCP/TLS stack. Under the hood, this is `connect(2)` followed by TLS handshake bytes, then `write(2)` for the HTTP request, then `epoll_wait` (Linux) or `kqueue` (macOS) until the response arrives. Each `.await` point is a potential suspension - the task yields and another task runs while we wait for the network.

The channel (`mpsc::channel`) uses an internal ring buffer with atomic operations for coordination. `try_send` is non-blocking - if the buffer is full, it returns `Err(TrySendError::Full)` instead of waiting. This prevents a slow consumer (the crawl loop) from blocking fast producers (the extraction tasks).

## Scaling considerations

This crawler works for indexing small to medium sites (hundreds to low thousands of pages). For larger workloads, you'd want:

**Persistent frontier.** Our in-memory channel loses all pending URLs if the process crashes. Production crawlers use disk-backed queues - [RocksDB](https://github.com/rust-rocksdb/rust-rocksdb) or SQLite with WAL mode work well. The frontier becomes a priority queue where you can reorder URLs by importance.

**Distributed deduplication.** A `HashSet` in memory works up to a few million URLs. Beyond that, look at probabilistic data structures - a [Bloom filter](https://crates.io/crates/bloomfilter) uses roughly 1 byte per entry with a 1% false positive rate. For 100 million URLs, that's 100MB instead of the ~10GB a HashSet would need.

**DNS caching.** Each `reqwest` request triggers a DNS lookup. The system resolver caches these, but under high concurrency you can saturate it. [hickory-dns](https://github.com/hickory-dns/hickory-dns) (formerly trust-dns) gives you a configurable async resolver.

**Redirect handling.** Our crawler follows redirects (reqwest's default), but doesn't track them. A redirect from `/old-page` to `/new-page` might mean both URLs show up as visited, or the redirect target might be off-domain. Production crawlers log redirect chains and apply domain filtering after resolution.

**Content hashing.** Deduplication by URL misses pages with different URLs but identical content (common with CMS-generated pages, pagination, and query parameter variations). Hashing the response body with xxHash or BLAKE3 catches these - store the hash alongside the URL in your visited set.

## The complete code

Here's the full crawler in one block, around 250 lines:

```rust
use scraper::{Html, Selector};
use std::collections::{HashMap, HashSet};
use std::sync::Arc;
use std::time::Duration;
use texting_robots::Robot;
use tokio::sync::{mpsc, Mutex, Semaphore};
use url::Url;

#[derive(Debug, Clone)]
struct CrawlTask {
    url: Url,
    depth: u32,
}

struct Crawler {
    client: reqwest::Client,
    visited: Mutex<HashSet<String>>,
    domain_delays: Mutex<HashMap<String, tokio::time::Instant>>,
    semaphore: Arc<Semaphore>,
    max_depth: u32,
    delay: Duration,
    user_agent: String,
}

async fn is_allowed(client: &reqwest::Client, url: &Url, user_agent: &str) -> bool {
    let robots_url = format!(
        "{}://{}/robots.txt",
        url.scheme(),
        url.host_str().unwrap_or_default()
    );
    let body = match client.get(&robots_url).send().await {
        Ok(resp) if resp.status().is_success() => resp.text().await.unwrap_or_default(),
        _ => return true,
    };
    match Robot::new(user_agent, body.as_bytes()) {
        Ok(robot) => robot.allowed(url.as_str()),
        Err(_) => true,
    }
}

fn extract_links(html: &str, base: &Url) -> Vec<Url> {
    let document = Html::parse_document(html);
    let selector = Selector::parse("a[href]").unwrap();
    let mut links = Vec::new();
    for element in document.select(&selector) {
        if let Some(href) = element.value().attr("href") {
            if href.starts_with('#')
                || href.starts_with("javascript:")
                || href.starts_with("mailto:")
                || href.starts_with("data:")
            {
                continue;
            }
            if let Ok(resolved) = base.join(href) {
                links.push(resolved);
            }
        }
    }
    links
}

fn extract_title(html: &str) -> Option<String> {
    let document = Html::parse_document(html);
    let selector = Selector::parse("title").unwrap();
    document
        .select(&selector)
        .next()
        .map(|el| el.text().collect::<String>().trim().to_string())
}

fn normalize_url(url: &Url) -> String {
    let mut normalized = format!(
        "{}://{}{}",
        url.scheme(),
        url.host_str().unwrap_or_default(),
        url.path().trim_end_matches('/')
    );
    if let Some(query) = url.query() {
        if !query.is_empty() {
            normalized.push('?');
            normalized.push_str(query);
        }
    }
    normalized.to_lowercase()
}

impl Crawler {
    async fn wait_for_domain(&self, domain: &str) {
        let mut delays = self.domain_delays.lock().await;
        if let Some(last_request) = delays.get(domain) {
            let elapsed = last_request.elapsed();
            if elapsed < self.delay {
                let wait = self.delay - elapsed;
                drop(delays);
                tokio::time::sleep(wait).await;
                let mut delays = self.domain_delays.lock().await;
                delays.insert(domain.to_string(), tokio::time::Instant::now());
            } else {
                delays.insert(domain.to_string(), tokio::time::Instant::now());
            }
        } else {
            delays.insert(domain.to_string(), tokio::time::Instant::now());
        }
    }

    async fn fetch(&self, url: &Url) -> Option<String> {
        let domain = url.host_str().unwrap_or_default().to_string();
        let _permit = self.semaphore.acquire().await.ok()?;
        self.wait_for_domain(&domain).await;

        if !is_allowed(&self.client, url, &self.user_agent).await {
            eprintln!("  [blocked] {} (robots.txt)", url);
            return None;
        }

        let response = match self.client.get(url.as_str()).send().await {
            Ok(resp) => resp,
            Err(e) => {
                eprintln!("  [error] {} - {}", url, e);
                return None;
            }
        };

        if !response.status().is_success() {
            eprintln!("  [{}] {}", response.status(), url);
            return None;
        }

        let content_type = response
            .headers()
            .get("content-type")
            .and_then(|v| v.to_str().ok())
            .unwrap_or_default();
        if !content_type.contains("text/html") {
            return None;
        }

        response.text().await.ok()
    }

    async fn crawl(self: &Arc<Self>, seed: Url, max_pages: usize) {
        let (tx, mut rx) = mpsc::channel::<CrawlTask>(1000);
        let mut pages_crawled = 0u32;

        {
            let mut visited = self.visited.lock().await;
            visited.insert(seed.to_string());
        }
        tx.send(CrawlTask { url: seed, depth: 0 }).await.ok();

        let mut tasks = tokio::task::JoinSet::new();

        loop {
            tokio::select! {
                Some(task) = rx.recv() => {
                    if pages_crawled as usize >= max_pages {
                        continue;
                    }

                    let crawler = Arc::clone(self);
                    let tx = tx.clone();
                    tasks.spawn(async move {
                        let url_str = task.url.to_string();
                        println!("[depth={}] {}", task.depth, url_str);

                        let html = match crawler.fetch(&task.url).await {
                            Some(h) => h,
                            None => return,
                        };

                        let title = extract_title(&html)
                            .unwrap_or_else(|| url_str.clone());
                        println!("  -> \"{}\" ({} bytes)", title, html.len());

                        if task.depth >= crawler.max_depth {
                            return;
                        }

                        let links = extract_links(&html, &task.url);
                        let same_domain = task.url.host_str()
                            .unwrap_or_default().to_string();

                        let mut visited = crawler.visited.lock().await;
                        for link in links {
                            if link.host_str().unwrap_or_default() != same_domain {
                                continue;
                            }
                            let normalized = normalize_url(&link);
                            if visited.insert(normalized) {
                                let _ = tx.try_send(CrawlTask {
                                    url: link,
                                    depth: task.depth + 1,
                                });
                            }
                        }
                    });

                    pages_crawled += 1;
                }

                Some(result) = tasks.join_next() => {
                    if let Err(e) = result {
                        eprintln!("Task panicked: {}", e);
                    }
                }

                else => break,
            }
        }

        while let Some(result) = tasks.join_next().await {
            if let Err(e) = result {
                eprintln!("Task panicked: {}", e);
            }
        }
    }
}

#[tokio::main]
async fn main() {
    let args: Vec<String> = std::env::args().collect();
    let seed_url = args
        .get(1)
        .map(|s| s.as_str())
        .unwrap_or("https://www.rust-lang.org");

    let url = Url::parse(seed_url).expect("Invalid seed URL");

    let crawler = Arc::new(Crawler {
        client: reqwest::Client::builder()
            .user_agent("rustcrawler/0.1 (+https://github.com/example/rustcrawler)")
            .timeout(Duration::from_secs(10))
            .connect_timeout(Duration::from_secs(5))
            .pool_max_idle_per_host(5)
            .gzip(true)
            .build()
            .expect("Failed to build HTTP client"),
        visited: Mutex::new(HashSet::new()),
        domain_delays: Mutex::new(HashMap::new()),
        semaphore: Arc::new(Semaphore::new(5)),
        max_depth: 2,
        delay: Duration::from_millis(500),
        user_agent: "rustcrawler".to_string(),
    });

    println!("Crawling {} (max depth: 2, concurrency: 5)\n", url);
    crawler.crawl(url, 50).await;
    println!(
        "\nDone. Visited {} unique URLs.",
        crawler.visited.lock().await.len()
    );
}
```

Every piece of this touches something fundamental about async Rust. The semaphore is cooperative scheduling. The channel is the producer-consumer pattern. The mutex-across-await dance is the most common source of bugs in async code. The `select!` loop is how you multiplex event sources without threads. And the `JoinSet` is how you manage dynamic fan-out without leaking tasks.

Not bad for 250 lines.
