+++
title = "Writing a simple HTTP server from scratch in Rust"
date = 2025-03-06
description = "Building a working HTTP server with just std::net::TcpListener - request parsing, routing, query params, JSON bodies - then collapsing it all into three lines of Axum."

[taxonomies]
tags = ["rust", "http", "networking", "axum"]
+++

Every Rust web developer has written `Router::new().route("/", get(handler))` and moved on. Axum, actix-web, warp - they all make HTTP feel like function calls. Which is the point. But there's a cost to that abstraction: you stop thinking about what HTTP actually is. It's bytes on a TCP socket. A text protocol. A request is a string that starts with `GET /path HTTP/1.1\r\n`, and a response is a string that starts with `HTTP/1.1 200 OK\r\n`. That's it.

This post builds a working HTTP server using nothing but `std::net::TcpListener`. We'll parse requests by hand, route GET and POST, extract query parameters, deserialize JSON bodies, and return proper responses. Around 150 lines of Rust, no dependencies except `serde` and `serde_json` for the JSON part. Then we'll rewrite the whole thing in Axum to see exactly how much a framework does for you.

None of this is production-ready. That's the point - production-ready is [hyper](https://github.com/hyperium/hyper) and [tokio](https://github.com/tokio-rs/tokio) and months of edge case handling. This is about understanding.

<!-- more -->

## What HTTP looks like on the wire

Before writing any code, you should know what you're parsing. Here's a raw HTTP/1.1 request as defined in [RFC 9112](https://www.rfc-editor.org/rfc/rfc9112.html):

```
GET /users?role=admin HTTP/1.1\r\n
Host: localhost:8080\r\n
Accept: application/json\r\n
\r\n
```

Three parts:

1. **Request line:** method, path (with query string), protocol version. Separated by spaces, terminated by `\r\n`.
2. **Headers:** key-value pairs, each terminated by `\r\n`. The header block ends with an empty line (`\r\n\r\n` - that double CRLF is the boundary between headers and body).
3. **Body:** everything after the double CRLF. Present in POST/PUT/PATCH, absent in GET/DELETE (usually). The `Content-Length` header tells you how many bytes to read.

A POST with a JSON body:

```
POST /users HTTP/1.1\r\n
Host: localhost:8080\r\n
Content-Type: application/json\r\n
Content-Length: 42\r\n
\r\n
{"name":"alice","email":"alice@example.com"}
```

And a response:

```
HTTP/1.1 200 OK\r\n
Content-Type: application/json\r\n
Content-Length: 27\r\n
\r\n
{"id":1,"name":"alice"}
```

Same structure: status line, headers, blank line, body. The status line has the protocol, a numeric status code, and a reason phrase.

That's the entire protocol at the HTTP/1.1 level. No magic. Let's parse it.

## Step 1: listening for connections

The standard library gives us `std::net::TcpListener`. Bind it to an address, call `incoming()`, and you get a stream of TCP connections. Each connection is a `TcpStream` - a bidirectional byte pipe.

```rust
use std::net::TcpListener;

fn main() {
    let listener = TcpListener::bind("127.0.0.1:8080").unwrap();
    println!("Listening on http://127.0.0.1:8080");

    for stream in listener.incoming() {
        let stream = stream.unwrap();
        handle_connection(stream);
    }
}
```

This is single-threaded and blocking. The `listener.incoming()` call wraps `accept()`, which is a syscall that blocks until a client connects. On Linux, this translates to the `accept4(2)` syscall. You can verify with strace:

```bash
strace -e accept4,read,write ./target/debug/http-server
```

You'll see something like:

```
accept4(3, {sa_family=AF_INET, sin_port=htons(54321), sin_addr=inet_addr("127.0.0.1")}, [128 -> 16], SOCK_CLOEXEC) = 4
read(4, "GET / HTTP/1.1\r\nHost: localhost:8080\r\n...", 4096) = 78
write(4, "HTTP/1.1 200 OK\r\nContent-Length: 13\r\n\r\nHello, world!", 52) = 52
```

File descriptor 3 is the listener socket. When a client connects, `accept4` returns fd 4 - the new connection. Then it's just `read` and `write` on that fd. If you've used tokio extensively but never looked at the synchronous version, if you're not familiar with the async runtime layer, I covered the full tokio I/O driver and `epoll` flow in [Understanding Tokio](/blog/understanding-tokio---the-rust-async-runtime-under-the-hood/). The synchronous version is the same thing minus the event loop - one thread, one connection, blocking.

## Step 2: parsing the request

Now we need to turn raw bytes into something usable. Here's a minimal request struct and parser:

```rust
use std::collections::HashMap;
use std::io::Read;
use std::net::TcpStream;

#[derive(Debug)]
struct Request {
    method: String,
    path: String,
    query: HashMap<String, String>,
    headers: HashMap<String, String>,
    body: String,
}

fn parse_request(stream: &mut TcpStream) -> Option<Request> {
    let mut buf = [0u8; 4096];
    let n = stream.read(&mut buf).ok()?;
    if n == 0 {
        return None;
    }
    let raw = String::from_utf8_lossy(&buf[..n]);

    // Split headers from body at the double CRLF
    let (head, body) = raw.split_once("\r\n\r\n").unwrap_or((&raw, ""));

    let mut lines = head.lines();

    // Request line: "GET /path?key=val HTTP/1.1"
    let request_line = lines.next()?;
    let mut parts = request_line.split_whitespace();
    let method = parts.next()?.to_string();
    let full_path = parts.next()?;
    // parts.next() would be "HTTP/1.1" - we ignore it

    // Split path from query string
    let (path, query) = if let Some((p, q)) = full_path.split_once('?') {
        (p.to_string(), parse_query(q))
    } else {
        (full_path.to_string(), HashMap::new())
    };

    // Headers
    let mut headers = HashMap::new();
    for line in lines {
        if let Some((key, value)) = line.split_once(": ") {
            headers.insert(key.to_lowercase(), value.to_string());
        }
    }

    Some(Request { method, path, query, headers, body: body.to_string() })
}
```

A few things to call out:

**The 4096-byte buffer.** We read up to 4KB in one shot. This works for small requests. A real server would check `Content-Length`, loop on `read()` until it has the full body, handle chunked transfer encoding, and deal with requests larger than any fixed buffer. We're skipping all of that.

**`String::from_utf8_lossy`.** HTTP/1.1 headers are ASCII (technically a subset of ISO-8859-1 per the spec, but in practice ASCII). The body could be anything - binary, UTF-8, whatever `Content-Type` says. We're treating everything as a string, which is fine for JSON but would corrupt binary payloads.

**`split_once("\r\n\r\n")`.** This is the HTTP boundary between headers and body. If the double CRLF isn't in our buffer (because the request is larger than 4KB and got split across reads), this breaks. Again - real servers handle this. We don't.

**Lowercase header keys.** HTTP headers are case-insensitive per [RFC 9110 Section 5.1](https://www.rfc-editor.org/rfc/rfc9110.html#section-5.1). `Content-Type`, `content-type`, and `CONTENT-TYPE` are all the same header. Lowercasing on insert lets us look up with `.get("content-type")` consistently.

## Step 3: query parameter parsing

Query strings follow a simple format: `key1=value1&key2=value2`. URL encoding is a thing (`%20` for spaces, `+` for spaces in form data), but we'll skip that for now:

```rust
fn parse_query(query: &str) -> HashMap<String, String> {
    query
        .split('&')
        .filter_map(|pair| {
            let (k, v) = pair.split_once('=')?;
            Some((k.to_string(), v.to_string()))
        })
        .collect()
}
```

This handles `?name=alice&role=admin` and quietly drops malformed pairs (anything without an `=`). A production URL parser would use the [url](https://crates.io/crates/url) crate or [form_urlencoded](https://docs.rs/form_urlencoded/) from the `url` crate, which handles percent-decoding, repeated keys, empty values, and all the edge cases from [the WHATWG URL Standard](https://url.spec.whatwg.org/#urlencoded-parsing).

## Step 4: building responses

We need a way to construct valid HTTP responses. A small helper:

```rust
use std::io::Write;

fn respond(stream: &mut TcpStream, status: u16, content_type: &str, body: &str) {
    let reason = match status {
        200 => "OK",
        201 => "Created",
        400 => "Bad Request",
        404 => "Not Found",
        405 => "Method Not Allowed",
        500 => "Internal Server Error",
        _ => "Unknown",
    };

    let response = format!(
        "HTTP/1.1 {} {}\r\nContent-Type: {}\r\nContent-Length: {}\r\nConnection: close\r\n\r\n{}",
        status, reason, content_type, body.len(), body
    );

    let _ = stream.write_all(response.as_bytes());
}

fn json_response(stream: &mut TcpStream, status: u16, body: &str) {
    respond(stream, status, "application/json", body);
}
```

We're sending `Connection: close` because we don't handle keep-alive. In HTTP/1.1, connections are persistent by default - the client can send multiple requests on the same TCP connection. Our server reads one request and closes. This is inefficient (a new TCP handshake for every request), but it massively simplifies the code. HTTP/1.0 was `Connection: close` by default; keep-alive was opt-in. HTTP/1.1 flipped that, which is why every real server implements connection management.

## Step 5: routing and handlers

Now we need to map requests to handler functions. A real framework uses a trie or regex-based router. We'll use pattern matching:

```rust
use serde::{Deserialize, Serialize};

#[derive(Serialize)]
struct User {
    id: u32,
    name: String,
    email: String,
}

#[derive(Deserialize)]
struct CreateUser {
    name: String,
    email: String,
}

fn handle_connection(mut stream: TcpStream) {
    let req = match parse_request(&mut stream) {
        Some(r) => r,
        None => return,
    };

    match (req.method.as_str(), req.path.as_str()) {
        ("GET", "/") => {
            respond(&mut stream, 200, "text/plain", "Hello, world!");
        }
        ("GET", "/users") => handle_list_users(&mut stream, &req),
        ("POST", "/users") => handle_create_user(&mut stream, &req),
        _ => {
            json_response(&mut stream, 404, r#"{"error":"not found"}"#);
        }
    }
}
```

The `match (method, path)` pattern is surprisingly readable. It breaks down when you need path parameters (`/users/:id`) - you'd need to split the path into segments and match on those. But for a fixed set of routes, this works.

Now the handlers:

```rust
fn handle_list_users(stream: &mut TcpStream, req: &Request) {
    // Pretend we have a database
    let users = vec![
        User { id: 1, name: "Alice".into(), email: "alice@example.com".into() },
        User { id: 2, name: "Bob".into(), email: "bob@example.com".into() },
    ];

    // Filter by query param if present
    let filtered: Vec<&User> = if let Some(name) = req.query.get("name") {
        users.iter().filter(|u| u.name.to_lowercase() == name.to_lowercase()).collect()
    } else {
        users.iter().collect()
    };

    let body = serde_json::to_string(&filtered).unwrap();
    json_response(stream, 200, &body);
}

fn handle_create_user(stream: &mut TcpStream, req: &Request) {
    // Check content-type
    let ct = req.headers.get("content-type").map(|s| s.as_str()).unwrap_or("");
    if !ct.starts_with("application/json") {
        json_response(stream, 400, r#"{"error":"expected application/json"}"#);
        return;
    }

    // Deserialize body
    let user: CreateUser = match serde_json::from_str(&req.body) {
        Ok(u) => u,
        Err(e) => {
            let msg = format!(r#"{{"error":"invalid json: {}"}}"#, e);
            json_response(stream, 400, &msg);
            return;
        }
    };

    // "Create" the user (in reality, you'd insert into a database)
    let created = User { id: 3, name: user.name, email: user.email };
    let body = serde_json::to_string(&created).unwrap();
    json_response(stream, 201, &body);
}
```

The `handle_create_user` function shows the manual work that a framework does automatically: check `Content-Type`, parse the body, handle deserialization errors with meaningful messages. Axum's `Json<T>` extractor does all three in a single function parameter.

## The complete server

Here's everything stitched together. One file, 148 lines, two dependencies (`serde`, `serde_json`):

```rust
use serde::{Deserialize, Serialize};
use std::collections::HashMap;
use std::io::{Read, Write};
use std::net::{TcpListener, TcpStream};

// --- Types ---

#[derive(Debug)]
struct Request {
    method: String,
    path: String,
    query: HashMap<String, String>,
    headers: HashMap<String, String>,
    body: String,
}

#[derive(Serialize)]
struct User {
    id: u32,
    name: String,
    email: String,
}

#[derive(Deserialize)]
struct CreateUser {
    name: String,
    email: String,
}

// --- Parsing ---

fn parse_query(query: &str) -> HashMap<String, String> {
    query
        .split('&')
        .filter_map(|pair| {
            let (k, v) = pair.split_once('=')?;
            Some((k.to_string(), v.to_string()))
        })
        .collect()
}

fn parse_request(stream: &mut TcpStream) -> Option<Request> {
    let mut buf = [0u8; 4096];
    let n = stream.read(&mut buf).ok()?;
    if n == 0 {
        return None;
    }
    let raw = String::from_utf8_lossy(&buf[..n]);
    let (head, body) = raw.split_once("\r\n\r\n").unwrap_or((&raw, ""));
    let mut lines = head.lines();

    let request_line = lines.next()?;
    let mut parts = request_line.split_whitespace();
    let method = parts.next()?.to_string();
    let full_path = parts.next()?;

    let (path, query) = if let Some((p, q)) = full_path.split_once('?') {
        (p.to_string(), parse_query(q))
    } else {
        (full_path.to_string(), HashMap::new())
    };

    let mut headers = HashMap::new();
    for line in lines {
        if let Some((key, value)) = line.split_once(": ") {
            headers.insert(key.to_lowercase(), value.to_string());
        }
    }

    Some(Request { method, path, query, headers, body: body.to_string() })
}

// --- Response ---

fn respond(stream: &mut TcpStream, status: u16, content_type: &str, body: &str) {
    let reason = match status {
        200 => "OK",
        201 => "Created",
        400 => "Bad Request",
        404 => "Not Found",
        405 => "Method Not Allowed",
        _ => "Unknown",
    };
    let response = format!(
        "HTTP/1.1 {status} {reason}\r\n\
         Content-Type: {content_type}\r\n\
         Content-Length: {}\r\n\
         Connection: close\r\n\r\n\
         {body}",
        body.len(),
    );
    let _ = stream.write_all(response.as_bytes());
}

fn json_response(stream: &mut TcpStream, status: u16, body: &str) {
    respond(stream, status, "application/json", body);
}

// --- Handlers ---

fn handle_list_users(stream: &mut TcpStream, req: &Request) {
    let users = vec![
        User { id: 1, name: "Alice".into(), email: "alice@example.com".into() },
        User { id: 2, name: "Bob".into(), email: "bob@example.com".into() },
    ];
    let filtered: Vec<&User> = if let Some(name) = req.query.get("name") {
        users.iter().filter(|u| u.name.to_lowercase() == name.to_lowercase()).collect()
    } else {
        users.iter().collect()
    };
    let body = serde_json::to_string(&filtered).unwrap();
    json_response(stream, 200, &body);
}

fn handle_create_user(stream: &mut TcpStream, req: &Request) {
    let ct = req.headers.get("content-type").map(|s| s.as_str()).unwrap_or("");
    if !ct.starts_with("application/json") {
        json_response(stream, 400, r#"{"error":"expected application/json"}"#);
        return;
    }
    let user: CreateUser = match serde_json::from_str(&req.body) {
        Ok(u) => u,
        Err(e) => {
            let msg = format!(r#"{{"error":"invalid json: {}"}}"#, e);
            json_response(stream, 400, &msg);
            return;
        }
    };
    let created = User { id: 3, name: user.name, email: user.email };
    let body = serde_json::to_string(&created).unwrap();
    json_response(stream, 201, &body);
}

// --- Router ---

fn handle_connection(mut stream: TcpStream) {
    let req = match parse_request(&mut stream) {
        Some(r) => r,
        None => return,
    };
    match (req.method.as_str(), req.path.as_str()) {
        ("GET", "/") => respond(&mut stream, 200, "text/plain", "Hello, world!"),
        ("GET", "/users") => handle_list_users(&mut stream, &req),
        ("POST", "/users") => handle_create_user(&mut stream, &req),
        _ => json_response(&mut stream, 404, r#"{"error":"not found"}"#),
    }
}

// --- Main ---

fn main() {
    let listener = TcpListener::bind("127.0.0.1:8080").unwrap();
    println!("Listening on http://127.0.0.1:8080");
    for stream in listener.incoming() {
        match stream {
            Ok(s) => handle_connection(s),
            Err(e) => eprintln!("Connection failed: {}", e),
        }
    }
}
```

Test it:

```bash
# GET
curl http://localhost:8080/
# Hello, world!

# GET with query params
curl "http://localhost:8080/users?name=alice"
# [{"id":1,"name":"Alice","email":"alice@example.com"}]

# POST with JSON body
curl -X POST http://localhost:8080/users \
  -H "Content-Type: application/json" \
  -d '{"name":"charlie","email":"charlie@example.com"}'
# {"id":3,"name":"charlie","email":"charlie@example.com"}

# 404
curl http://localhost:8080/nope
# {"error":"not found"}
```

It works. It routes, it parses query strings, it deserializes JSON, it returns proper status codes and content types. 148 lines.

## Everything we're not handling

Before you feel proud of those 148 lines, here's what a real HTTP server does that we don't:

**Concurrency.** Our server handles one connection at a time. While it's reading from client A, clients B through Z are queued in the kernel's TCP backlog (default 128 on Linux). A thread-per-connection model (`std::thread::spawn` for each stream) fixes the obvious problem but caps you at a few thousand concurrent connections due to thread stack memory. The async model (tokio + epoll/kqueue) handles tens of thousands on a single thread. That's what Axum and hyper use.

**Keep-alive.** HTTP/1.1 defaults to persistent connections. A client sends multiple requests on one TCP connection. Our server sends `Connection: close` and bails. In a real server, you loop: read request, write response, read next request, write next response, until the client closes or a timeout fires.

**Chunked transfer encoding.** When the server doesn't know the response size upfront (streaming data), it sends chunks prefixed with their hex length. `Transfer-Encoding: chunked` is how large responses and server-sent events work in HTTP/1.1.

**Request body streaming.** We read the entire body into a string. For a 200MB file upload, that means 200MB of memory per request. A real server streams the body - reads chunks and processes them incrementally.

**URL decoding.** `%20` should become a space. `%2F` becomes `/`. `+` becomes space in form data but stays `+` in path segments. We're treating everything as raw bytes.

**Header parsing edge cases.** Headers can span multiple lines (obsolete line folding), header values can contain commas to represent multiple values, some headers (like `Set-Cookie`) can appear multiple times. Our single-`HashMap` approach loses duplicate headers.

**TLS.** No HTTPS. In production, you terminate TLS at a reverse proxy (nginx, Cloudflare) or use `rustls`/`native-tls` directly.

**HTTP/2 and HTTP/3.** Completely different wire protocols. HTTP/2 uses binary framing and multiplexing. HTTP/3 runs over QUIC (UDP). Our text parser only understands HTTP/1.1.

This is not a small list. And it's why [hyper](https://github.com/hyperium/hyper) (the HTTP library under Axum) is over 20,000 lines of code.

## The same thing in Axum

Let's rewrite the exact same functionality - three routes, query params, JSON body - using [Axum 0.8](https://crates.io/crates/axum):

```rust
use axum::{
    extract::Query,
    routing::get,
    Json, Router,
};
use serde::{Deserialize, Serialize};

#[derive(Serialize, Clone)]
struct User {
    id: u32,
    name: String,
    email: String,
}

#[derive(Deserialize)]
struct CreateUser {
    name: String,
    email: String,
}

#[derive(Deserialize)]
struct ListParams {
    name: Option<String>,
}

async fn root() -> &'static str {
    "Hello, world!"
}

async fn list_users(Query(params): Query<ListParams>) -> Json<Vec<User>> {
    let users = vec![
        User { id: 1, name: "Alice".into(), email: "alice@example.com".into() },
        User { id: 2, name: "Bob".into(), email: "bob@example.com".into() },
    ];
    let filtered = if let Some(ref name) = params.name {
        users.into_iter().filter(|u| u.name.to_lowercase() == name.to_lowercase()).collect()
    } else {
        users
    };
    Json(filtered)
}

async fn create_user(Json(input): Json<CreateUser>) -> (axum::http::StatusCode, Json<User>) {
    let user = User { id: 3, name: input.name, email: input.email };
    (axum::http::StatusCode::CREATED, Json(user))
}

#[tokio::main]
async fn main() {
    let app = Router::new()
        .route("/", get(root))
        .route("/users", get(list_users).post(create_user));

    let listener = tokio::net::TcpListener::bind("127.0.0.1:8080").await.unwrap();
    axum::serve(listener, app).await.unwrap();
}
```

Look at what disappeared:

- **Request parsing** - gone. Axum's `FromRequest` trait implementations handle method matching, header parsing, body reading.
- **Query string parsing** - `Query<T>` deserializes it via serde.
- **JSON body parsing** - `Json<T>` reads the body, checks `Content-Type`, deserializes, and returns a 422 with details if it fails.
- **Response formatting** - returning `Json<T>` sets `Content-Type: application/json`, serializes the body, sets `Content-Length`.
- **Status codes** - return a tuple of `(StatusCode, body)`.
- **Connection management** - `axum::serve` handles keep-alive, concurrent connections (via tokio), graceful shutdown.
- **Error responses** - `Json<T>` extractor returns a detailed error if deserialization fails. Our hand-rolled version just says "invalid json."

The routing in Axum compiles down to a trie-based matcher. The `.route("/users", get(list_users).post(create_user))` creates a node at `/users` with separate handlers per method. Under the hood, this builds a [`matchit::Router`](https://github.com/ibraheemdev/matchit) - a radix tree that matches paths in O(path length), not O(number of routes). Our `match` statement is O(number of routes), which is fine for 3 routes but falls apart at 500.

If you want to understand how Axum wires routes to handlers at the type level - how it works with different function signatures - there's an [excellent discussion in the Axum repo](https://github.com/tokio-rs/axum/discussions/1438) that walks through the `Handler` trait and its blanket implementations.

## The layers between you and the kernel

Here's the full call stack when an Axum handler runs, from your code down to the kernel:

```
Your handler function
    |
axum::Router       -- matches path, extracts params
    |
tower::Service     -- middleware chain (CORS, auth, logging)
    |
hyper::server      -- HTTP/1.1 or HTTP/2 protocol handling
    |
hyper::proto       -- actual byte parsing (httparse crate)
    |
tokio::net         -- async TcpStream, wraps mio
    |
mio::Poll          -- epoll_wait / kqueue / IOCP
    |
kernel             -- socket buffers, TCP state machine
```

Our from-scratch server collapses all of that into:

```
handle_connection function
    |
std::net::TcpStream  -- synchronous read/write
    |
kernel               -- socket buffers, TCP state machine
```

We replaced six layers with one. Each of those layers exists because real-world HTTP has real-world problems: slow clients that trickle bytes, connections that hang, malformed requests designed to crash your parser, servers that need to handle 50,000 concurrent connections on 4 cores.

The `httparse` crate (used by hyper for header parsing) is worth looking at specifically. It parses HTTP headers [without allocating](https://github.com/seanmonstar/httparse) - no `String`, no `HashMap`, just references into the original buffer. Our parser allocates a `String` for every header name and value. On a high-traffic server handling 100k requests/second, that difference is millions of allocations per second. `httparse` also validates headers properly - checking for invalid bytes, handling the obsolete line folding, rejecting requests with ambiguous `Content-Length` (which is a real attack vector - see [HTTP request smuggling](https://portswigger.net/web-security/request-smuggling)).

## Measuring the difference

If you pointed the [load testing tools I covered previously](/blog/load-testing-your-rust-api-tools-and-methodology) at both servers, you'd see stark differences under pressure.

The from-scratch server chokes quickly. With `hey -c 100 -z 10s http://localhost:8080/users`, most connections timeout because the server is single-threaded - it processes one request while 99 others wait. You could spawn a thread per connection:

```rust
for stream in listener.incoming() {
    match stream {
        Ok(s) => {
            std::thread::spawn(move || handle_connection(s));
        }
        Err(e) => eprintln!("Connection failed: {}", e),
    }
}
```

That helps, but threads are expensive. Each one allocates a stack (8MB default on Linux). 1000 concurrent connections means 8GB of virtual memory just for stacks. Plus context switching overhead. The async model (tokio) multiplexes thousands of tasks onto a handful of OS threads using cooperative scheduling and epoll for I/O readiness. That's the difference between "handles 1000 connections" and "handles 100,000."

## Why this exercise matters

If you only ever use frameworks, HTTP becomes a black box. You see `StatusCode::NOT_FOUND` but don't think about the `404` bytes going over the wire. You use `Json<T>` but don't think about `Content-Type` negotiation. You trust `axum::serve` but don't know it's doing keep-alive, pipelining, and header validation for you.

When something breaks - a client sends a malformed header, a proxy strips `Content-Length`, a load balancer sends HTTP/2 to a server expecting HTTP/1.1 - you need to think at the protocol level. `tcpdump`, `curl -v`, Wireshark. These tools show you bytes and headers, not Rust types.

The best part about writing this from scratch is the appreciation. Those 148 lines feel capable until you list everything they can't do. Then you look at Axum and think: yeah, that `.route()` line is doing a lot of work. And it should.

Frameworks aren't hiding complexity from you. They're solving it.
