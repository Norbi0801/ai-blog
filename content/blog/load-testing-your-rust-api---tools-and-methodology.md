+++
title = "Load testing your Rust API - tools and methodology"
date = 2026-04-05
description = "How to load test Rust web services with k6, wrk, drill, and hey - from baseline measurements to finding bottlenecks under pressure."

[taxonomies]
tags = ["rust", "performance", "load-testing", "devops"]
+++

Your Rust API compiles, the tests pass, and a quick curl returns the right JSON. Ship it? Not yet. You have no idea how it behaves when 500 users hit it at the same time. Or 5,000. Or when a steady stream of requests runs for 12 hours straight and that connection pool you configured starts leaking.

Load testing is how you find out before your users do. Rust gives you a strong starting position here - no garbage collector pauses, predictable memory usage, real threads via tokio - but "fast language" does not mean "fast system." A badly tuned database pool, a blocking call on the async runtime, or an O(n^2) serialization path will still bring your API to its knees.

This post covers the practical side: which tools to use, what patterns of load to apply, what numbers to watch, and how to figure out whether your bottleneck is CPU, I/O, or something else entirely.

<!-- more -->

## The toolbox

There are dozens of load testing tools. These four cover the spectrum from dead-simple to full scenario scripting.

### hey - the quick sanity check

[hey](https://github.com/rakyll/hey) (19.9k GitHub stars, written in Go) is what you reach for when you want a number in 10 seconds. It replaced ApacheBench for most people.

```bash
# 10,000 requests, 100 concurrent connections
hey -n 10000 -c 100 http://localhost:8080/api/health

# duration-based instead of count-based
hey -z 30s -c 200 http://localhost:8080/api/products

# POST with a body
hey -m POST -H "Content-Type: application/json" \
    -d '{"name":"test","price":999}' \
    -n 5000 -c 50 http://localhost:8080/api/products
```

hey gives you a latency histogram, percentile breakdown (p10 through p99), status code distribution, and throughput. No config files, no scripts - just a one-liner. Use it for quick before/after comparisons when you change something, or to get rough numbers during development.

Limitations: no scripting, no ramp-up, no multi-endpoint scenarios. When you outgrow it, move to k6.

### wrk - maximum raw throughput

[wrk](https://github.com/wg/wrk) (40k+ stars, written in C with LuaJIT) generates more raw traffic per machine than any other tool on this list. On the same hardware, wrk pushes roughly 5x the requests/sec of k6 because it is a tight C event loop, not a scripting runtime.

```bash
# 4 threads, 200 connections, 30 seconds, show latency percentiles
wrk -t4 -c200 -d30s --latency http://localhost:8080/api/products
```

Output looks like:

```
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency     1.23ms    0.45ms   15.67ms   94.12%
    Req/Sec    40.12k     2.34k    48.90k    72.50%
  Latency Distribution
     50%    1.12ms
     75%    1.34ms
     90%    1.67ms
     99%    3.45ms
  4812045 requests in 30.00s, 1.23GB read
Requests/sec: 160401.50
Transfer/sec:     42.01MB
```

You can extend wrk with Lua scripts for custom requests:

```lua
-- post.lua
wrk.method = "POST"
wrk.headers["Content-Type"] = "application/json"
wrk.body = '{"name": "load-test-item", "price": 1500}'
```

```bash
wrk -t4 -c200 -d30s -s post.lua http://localhost:8080/api/products
```

The big caveat: **wrk uses a closed-loop model**. It waits for each response before sending the next request from that connection. If the server stalls for 2 seconds, wrk sends no new requests during that time - and those 2 seconds disappear from the latency statistics. This is called **coordinated omission** (a term coined by Gil Tene), and it means wrk's latency numbers are optimistic under stress.

[wrk2](https://github.com/giltene/wrk2) fixes this with a constant-throughput model via the `-R` (rate) flag. It sends requests on schedule regardless of whether previous ones completed, and measures latency from when the request *should* have been sent. If you care about accurate tail latency, use wrk2.

Other limitations: HTTP/1.1 only (no HTTP/2), single-machine, no built-in scenario support.

### drill - the Rust-native option

[drill](https://github.com/fcsonline/drill) (crates.io v0.9.0) is a load tester written in Rust. Install it with `cargo install drill`. Configuration is YAML, inspired by Ansible:

```yaml
---
concurrency: 50
base: "http://localhost:8080"
iterations: 100
rampup: 5

plan:
  - name: "Health check"
    request:
      url: /api/health

  - name: "List products"
    request:
      url: /api/products

  - name: "Create product"
    request:
      url: /api/products
      method: POST
      headers:
        Content-Type: application/json
      body: '{"name": "bench-item", "price": 2500}'
    assign: "created"

  - name: "Fetch created product"
    request:
      url: "/api/products/{{ created.body.id }}"
```

The `assign` + interpolation syntax lets you chain requests - create something, then use the ID in the next request. drill also supports CSV data files (`with_items_from_csv`), assertions, cookie propagation, and tag-based filtering.

drill uses [HdrHistogram](https://crates.io/crates/hdrhistogram) internally, so its percentile reporting is accurate. It is a good choice when you want something lighter than k6 and want to stay in the Rust ecosystem. The tradeoff: fewer features than k6, smaller community, no cloud/distributed option.

### k6 - the full solution

[Grafana k6](https://github.com/grafana/k6) (29.9k stars, written in Go, AGPL-3.0) is the tool you graduate to for serious load testing. Since v1.0 (May 2025) it supports TypeScript natively - no transpilation step.

k6 has multiple **executor types** that determine how load is generated:

- `constant-vus` - fixed number of virtual users
- `ramping-vus` - VUs increase/decrease over stages (closed-loop)
- `constant-arrival-rate` - fixed request rate regardless of response time (open-loop, avoids coordinated omission)
- `ramping-arrival-rate` - request rate changes over stages (open-loop)

The distinction between closed-loop (`*-vus`) and open-loop (`*-arrival-rate`) matters. I will come back to this.

A minimal k6 script:

```javascript
import http from 'k6/http';
import { check, sleep } from 'k6';

export const options = {
  stages: [
    { duration: '1m', target: 50 },   // ramp to 50 VUs
    { duration: '3m', target: 50 },   // hold
    { duration: '1m', target: 0 },    // ramp down
  ],
  thresholds: {
    http_req_duration: ['p(95)<200', 'p(99)<500'],
    http_req_failed: ['rate<0.01'],
  },
};

export default function () {
  const res = http.get('http://localhost:8080/api/products');
  check(res, {
    'status is 200': (r) => r.status === 200,
    'response has items': (r) => JSON.parse(r.body).length > 0,
  });
  sleep(1); // think time between requests
}
```

The `thresholds` block is where k6 shines for CI integration: if p95 latency exceeds 200ms or error rate exceeds 1%, the test exits with a non-zero code. You can wire this into your pipeline and fail deployments automatically.

k6 also supports gRPC (`k6/net/grpc`), WebSockets, browser automation, and distributed execution via Grafana Cloud.

### Which tool when?

| Scenario | Tool |
|----------|------|
| Quick sanity check during dev | hey |
| Maximum throughput / raw benchmark | wrk or wrk2 |
| Rust ecosystem, YAML config, chained requests | drill |
| Full scenario testing, CI integration, thresholds | k6 |
| Accurate tail latency under stress | k6 (constant-arrival-rate) or wrk2 |

## What to measure

Three numbers matter. Everything else is derived from them.

### Latency percentiles (p50 / p95 / p99)

Averages are useless for latency. If 99 requests take 10ms and 1 request takes 4 seconds, your average is ~50ms. That looks fine. Meanwhile, 1% of your users are staring at a spinner for 4 seconds.

Percentiles tell the real story:

- **p50 (median)**: the "typical" request. 50% are faster, 50% are slower.
- **p95**: the beginning of the slow tail. At 1 million requests/day, 50,000 requests are slower than this number.
- **p99**: the "unlucky" users. At 1 million requests/day, 10,000 requests exceed this.

Under load, these numbers diverge. p50 might stay at 5ms while p99 jumps to 800ms. That divergence is the single most important signal in a load test. A flat percentile distribution under increasing load means your system scales well. A widening gap means contention is building somewhere.

Practical thresholds for a typical JSON API:

- p50 < 50ms, p95 < 200ms, p99 < 500ms - healthy
- p99 > 1s - investigate
- p99 > 2s - something is fundamentally wrong

### Throughput (requests per second)

How many requests your system handles per unit time. Watch for two things:

1. **Saturation point**: throughput climbs linearly with load, then plateaus. The plateau is your system's capacity.
2. **Throughput collapse**: past the saturation point, throughput can actually *decrease* because the system spends more time managing contention than doing useful work. Thread pool exhaustion, lock contention, and connection pool starvation all cause this.

### Error rate

Percentage of responses with non-2xx status codes or connection failures. Under moderate load this should be 0%. If errors appear before throughput saturates, you have a correctness bug, not a performance problem.

Watch for specific error patterns:
- 503 (Service Unavailable) - server is rejecting requests, usually connection or thread pool exhaustion
- 502/504 (Gateway Timeout) - upstream timeout, often database
- Connection refused / reset - OS-level resource exhaustion (file descriptors, TCP backlog)

## Load test patterns

Each pattern answers a different question. Run them in this order.

### 1. Baseline (smoke test)

```javascript
// k6: baseline.js
export const options = {
  vus: 1,
  duration: '1m',
};
```

One virtual user, one minute. This establishes your floor - the best possible latency with zero contention. If p99 is already 500ms with a single user, no amount of horizontal scaling will help. Fix the endpoint first.

### 2. Ramp-up (load test)

```javascript
// k6: load.js
export const options = {
  stages: [
    { duration: '2m', target: 100 },
    { duration: '5m', target: 100 },
    { duration: '2m', target: 0 },
  ],
  thresholds: {
    http_req_duration: ['p(95)<300'],
    http_req_failed: ['rate<0.01'],
  },
};
```

Gradually increase load to your expected peak. Hold it there. The ramp lets you see exactly when latency starts degrading. The hold period confirms the system is stable under that load, not slowly accumulating state.

If your production traffic is 200 req/s, test up to 300-400 req/s (1.5-2x) to have headroom.

### 3. Soak test (endurance)

```javascript
// k6: soak.js
export const options = {
  stages: [
    { duration: '5m', target: 80 },
    { duration: '8h', target: 80 },
    { duration: '5m', target: 0 },
  ],
};
```

Moderate load for hours. This catches:

- **Memory leaks** - Rust makes these rare but not impossible. `Arc` cycles, growing `Vec`s that never shrink, unbounded channels, forgotten `JoinHandle`s holding onto large state.
- **Connection pool exhaustion** - connections checked out but never returned due to error paths that skip cleanup.
- **File descriptor leaks** - open sockets, temp files, log handles.
- **Log growth** - disk fills up, writes start failing.

Monitor memory (RSS) and open file descriptors throughout the soak. In Rust, flat memory over 8 hours is realistic - it is one of the language's genuine advantages. But you have to verify it.

### 4. Spike test

```javascript
// k6: spike.js
export const options = {
  stages: [
    { duration: '30s', target: 10 },
    { duration: '10s', target: 500 },   // sudden spike
    { duration: '3m', target: 500 },    // hold spike
    { duration: '10s', target: 10 },    // drop back
    { duration: '2m', target: 10 },     // recovery period
  ],
};
```

The spike tests two things: can your system handle a sudden surge without crashing, and how fast does it recover after the surge ends. That recovery period at the end is critical - if latency stays elevated after load drops, something (a queue, a pool, a cache rebuild) is not draining properly.

## Bottleneck identification: CPU-bound vs I/O-bound

Your load test shows p99 spiking at 200 concurrent users. Now what? The fix depends entirely on whether you are CPU-bound or I/O-bound.

### How to tell

Run your load test and watch two things simultaneously:

```bash
# Terminal 1: CPU usage per core
htop
# or for scripting:
mpstat -P ALL 1

# Terminal 2: I/O wait and disk
iostat -x 1

# Terminal 3: open connections and socket state
ss -s
```

**CPU-bound signals:**
- One or more cores pinned at 100% (check per-core, not average - tokio by default uses one thread per core)
- I/O wait near 0%
- Low disk and network utilization
- Latency increases proportionally with concurrency

Common CPU-bound causes in Rust APIs:
- JSON serialization of large payloads (serde_json is fast, but not free)
- Regex compilation on every request (compile once, use `lazy_static` or `std::sync::LazyLock`)
- Cryptographic operations (bcrypt/argon2 password hashing, JWT validation)
- Complex business logic, sorting, filtering large in-memory datasets

The fix: `tokio::task::spawn_blocking` for heavy compute, or move it off the hot path entirely.

```rust
// Bad: blocking the async runtime
async fn hash_password(password: String) -> Result<String, Error> {
    // This blocks the tokio worker thread
    Ok(bcrypt::hash(password, 12)?)
}

// Good: offload to blocking thread pool
async fn hash_password(password: String) -> Result<String, Error> {
    tokio::task::spawn_blocking(move || {
        bcrypt::hash(password, 12)
    }).await?
    .map_err(Into::into)
}
```

If you have been through the [debugging post](/blog/debugging-rust-beyond-println), `tracing` with `#[instrument]` spans is excellent for finding which handler stages consume the most wall-clock time.

**I/O-bound signals:**
- CPU usage low (20-40%) across all cores
- High I/O wait percentage
- Many connections in `TIME_WAIT` or `CLOSE_WAIT` state
- Latency spikes correlate with database query times

Common I/O-bound causes:
- Database queries without indexes (full table scans under load)
- N+1 queries (one query per item in a list)
- Connection pool too small (requests queue waiting for a connection)
- External HTTP calls without timeouts (one slow upstream blocks a worker)
- DNS resolution on every request

### Connection pooling - getting it right

This is the single most common performance issue in Rust APIs under load. Whether you use `sqlx`, `diesel`, or `deadpool`, the pool size determines your concurrent database capacity.

```rust
// sqlx pool configuration
let pool = PgPoolOptions::new()
    .max_connections(20)        // start here, tune based on load tests
    .min_connections(5)         // keep warm connections ready
    .acquire_timeout(Duration::from_secs(3))  // fail fast, don't queue forever
    .idle_timeout(Duration::from_secs(600))
    .max_lifetime(Duration::from_secs(1800))
    .connect("postgres://localhost/mydb")
    .await?;
```

Rules of thumb for pool sizing:

- **Too small**: requests queue waiting for connections. You will see p99 spike while p50 stays flat - classic pool contention pattern.
- **Too large**: overwhelms the database with concurrent queries. PostgreSQL performance degrades past ~100 active connections on typical hardware.
- **Starting point**: 2-3x your CPU core count for the database server. So if your Postgres runs on 8 cores, start with 16-24 max connections.
- **Shared pools**: if you have 4 application instances each with `max_connections(20)`, that is 80 connections to the database. Account for all consumers.

The `acquire_timeout` is critical. Without it (or with a very high value), requests pile up in the pool queue during load spikes and latency balloons. A 3-second timeout means requests fail fast with a clear error rather than hanging for 30 seconds and timing out somewhere upstream.

To test pool behavior specifically, write a k6 test targeting your slowest database endpoint and watch connection count:

```sql
-- PostgreSQL: active connections from your app
SELECT count(*) FROM pg_stat_activity
WHERE application_name = 'your_app'
AND state = 'active';
```

If active connections sit at `max_connections` constantly during a load test while requests queue, your pool is the bottleneck. Either optimize the slow queries (so connections are returned faster) or increase the pool size (up to what the database can handle).

## How Rust holds up vs Node.js and Python

Numbers from [Sharkbench](https://sharkbench.dev/web) (August 2025, Ryzen 7 7800X3D, realistic JSON serialization + I/O workload):

| Framework | Req/s | Median Latency | Memory (RSS) |
|-----------|-------|----------------|--------------|
| Rust / Actix-web | 21,965 | 1.4 ms | 16.6 MB |
| Rust / Axum | 21,030 | 1.6 ms | 8.5 MB |
| JS / Fastify (Node.js) | 9,340 | 3.4 ms | 57.0 MB |
| JS / Express (Node.js) | 5,766 | 5.5 ms | 82.5 MB |
| Python / FastAPI | 1,185 | 21.0 ms | 41.2 MB |
| Python / Django | 950 | 8.8 ms | 130.1 MB |

TechEmpower Round 23 (February 2025, the final round before the project was [archived in March 2026](https://github.com/TechEmpower/FrameworkBenchmarks/issues/10932)) showed even larger gaps on heavier workloads like the Fortunes test (database queries + HTML templating):

| Framework | Req/s |
|-----------|-------|
| Actix-web | ~320,000 |
| Express (Node.js) | ~78,000 |
| Django (Python) | ~32,600 |

A few things stand out:

**Throughput**: Rust frameworks handle 2-4x more than Node.js and 18-20x more than Python under realistic conditions. The gap widens under heavier workloads.

**Memory**: Axum at 8.5 MB RSS is remarkable. That is 10x less than Express and 15x less than Django. This matters for soak tests - less memory means less GC pressure (in Node/Python), less swap risk, and more headroom for spikes.

**Latency stability**: This is where Rust really differentiates. Because there is no garbage collector, there are no GC pause spikes. In Node.js, you will see periodic p99 spikes that correspond to V8 GC cycles. In Python, the GIL creates serialization points that show up as latency steps under concurrency. Rust's p99/p50 ratio stays tight under load - often 3-5x, compared to 10-50x for GC-based runtimes under the same conditions.

**But**: if your bottleneck is the database (as it often is), the framework language matters less than you think. A poorly-indexed PostgreSQL query that takes 200ms dominates the response time regardless of whether the framework overhead is 1ms or 20ms.

## Practical k6 setup for a Rust API

Here is a complete, real-world k6 test suite structure for an API with CRUD endpoints:

```javascript
// test/load/config.js
export const BASE_URL = __ENV.BASE_URL || 'http://localhost:8080';

export const THRESHOLDS = {
  http_req_duration: ['p(50)<50', 'p(95)<200', 'p(99)<500'],
  http_req_failed: ['rate<0.01'],
  http_reqs: ['rate>100'],  // minimum throughput
};
```

```javascript
// test/load/scenarios/products.js
import http from 'k6/http';
import { check, sleep } from 'k6';
import { BASE_URL, THRESHOLDS } from '../config.js';

export const options = {
  scenarios: {
    // Simulate browsing (read-heavy)
    readers: {
      executor: 'ramping-vus',
      startVUs: 0,
      stages: [
        { duration: '1m', target: 50 },
        { duration: '3m', target: 50 },
        { duration: '1m', target: 0 },
      ],
      exec: 'browseProducts',
    },
    // Simulate purchases (write traffic)
    writers: {
      executor: 'constant-arrival-rate',
      rate: 10,              // 10 requests per second
      timeUnit: '1s',
      duration: '5m',
      preAllocatedVUs: 20,
      maxVUs: 50,
      exec: 'createOrder',
    },
  },
  thresholds: THRESHOLDS,
};

export function browseProducts() {
  const list = http.get(`${BASE_URL}/api/products`);
  check(list, { 'list 200': (r) => r.status === 200 });

  const products = JSON.parse(list.body);
  if (products.length > 0) {
    const id = products[Math.floor(Math.random() * products.length)].id;
    const detail = http.get(`${BASE_URL}/api/products/${id}`);
    check(detail, { 'detail 200': (r) => r.status === 200 });
  }

  sleep(Math.random() * 2 + 1); // 1-3s think time
}

export function createOrder() {
  const payload = JSON.stringify({
    product_id: 'some-known-id',
    quantity: Math.floor(Math.random() * 5) + 1,
  });

  const res = http.post(`${BASE_URL}/api/orders`, payload, {
    headers: { 'Content-Type': 'application/json' },
  });

  check(res, {
    'created 201': (r) => r.status === 201,
    'has order id': (r) => JSON.parse(r.body).id !== undefined,
  });
}
```

Run it:

```bash
# Install k6
brew install k6  # macOS
# or: snap install k6  # Linux

# Run against local API
k6 run test/load/scenarios/products.js

# Run with environment override
k6 run -e BASE_URL=https://staging.example.com test/load/scenarios/products.js

# Output to JSON for post-processing
k6 run --out json=results.json test/load/scenarios/products.js
```

Notice the two scenarios run simultaneously: `readers` uses `ramping-vus` (closed-loop, simulates users browsing at their own pace) while `writers` uses `constant-arrival-rate` (open-loop, simulates a steady stream of incoming orders regardless of processing speed). This models realistic traffic where reads vastly outnumber writes but writes must maintain consistent throughput.

The `constant-arrival-rate` executor on the write path is important. If order creation slows down, you want to know that requests are piling up - not have the load generator silently slow down to match.

### CI integration

```yaml
# .github/workflows/load-test.yml
name: Load Test
on:
  pull_request:
    branches: [main]

jobs:
  load-test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_DB: testdb
          POSTGRES_PASSWORD: test
        ports:
          - 5432:5432
    steps:
      - uses: actions/checkout@v4
      - uses: grafana/setup-k6-action@v1
      - name: Build and start API
        run: |
          cargo build --release
          DATABASE_URL=postgres://postgres:test@localhost/testdb \
            ./target/release/my-api &
          sleep 3  # wait for startup
      - name: Run load test
        run: k6 run test/load/scenarios/products.js
      # k6 exits non-zero if thresholds fail, which fails the CI step
```

This gives you automatic regression detection. If a PR introduces a change that pushes p95 over 200ms or error rate over 1%, the build fails.

## A checklist before you ship

Run these in order. Each one catches a different class of problem.

1. **Baseline** (1 VU, 1 min) - establish floor latency. If this is bad, fix the code.
2. **Ramp-up** (0 to 2x expected peak, hold 5 min) - find the saturation point and confirm stability.
3. **Spike** (sudden 10x surge, then recover) - verify graceful degradation and recovery.
4. **Soak** (80% of peak for 4-8 hours) - catch leaks and accumulation bugs.

At each stage, record: p50, p95, p99 latency, throughput (req/s), error rate, CPU usage, memory RSS, database connection count, open file descriptors.

The combination of Rust's predictable runtime behavior and a solid load testing methodology gives you something rare: confidence that your production numbers will look like your test numbers. No GC surprises at 3 AM, no memory bloat after a week of uptime. Just the performance characteristics you measured, holding steady.

That said, load testing is not a one-time task. Run it on every significant change. Automate it in CI. Make it boring. The most dangerous performance regressions are the ones that slip in one PR at a time, invisible until the cumulative effect brings down production on a Friday evening.
