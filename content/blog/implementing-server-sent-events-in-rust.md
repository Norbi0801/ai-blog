+++
title = "Implementing Server-Sent Events in Rust"
date = 2025-11-08
description = "How SSE works on the wire, how it compares to WebSockets and long polling, and a complete Axum implementation with backpressure, reconnection, and Last-Event-ID handling."

[taxonomies]
tags = ["rust", "axum", "http", "sse"]
+++

Most "real-time" features on the web do not need real-time. A live dashboard ticking once a second, a notification badge, a deploy log streaming into a browser, a progress bar for a long upload - all of these are one-way: the server has news, the client wants to hear it. You do not need a full-duplex socket for that. You need a long-lived HTTP response that the server keeps writing to.

That is what Server-Sent Events are. SSE is a 25-line spec layered on top of HTTP, with a built-in browser API, automatic reconnection, and event ordering. It is the "good enough" answer for the 80% of streaming cases where WebSockets are overkill. If you are not familiar with what HTTP actually looks like on the wire, I covered that in [Understanding TCP/IP - what happens when you curl a URL](/blog/understanding-tcp-ip-what-happens-when-you-curl-a-url). This post is about the layer above: how the response stays open, what the bytes look like, and how to ship a working implementation in Rust with [Axum](https://docs.rs/axum).

<!-- more -->

## The shape of an SSE response

An SSE endpoint is just an HTTP `GET` that never finishes. The server replies with `Content-Type: text/event-stream`, leaves the connection open, and writes lines whenever it has something to say. There is no framing protocol, no handshake, no upgrade. It is plain HTTP/1.1 chunked transfer encoding (or HTTP/2 stream frames - same thing semantically).

Here is a real exchange. Request:

```
GET /events HTTP/1.1
Host: api.example.com
Accept: text/event-stream
```

Response:

```
HTTP/1.1 200 OK
Content-Type: text/event-stream
Cache-Control: no-cache
Connection: keep-alive
Transfer-Encoding: chunked

data: hello

data: {"temp": 21.4}

event: status
id: 42
data: ready

retry: 5000
data: still here

```

That is the entire wire format. Each "event" is a block of `field: value` lines terminated by a blank line. The four fields the [HTML Living Standard](https://html.spec.whatwg.org/multipage/server-sent-events.html) defines are:

- `data:` - the payload. Multiple `data:` lines in one block get concatenated with `\n`.
- `event:` - a name. The browser fires this as a custom event type instead of the default `message`.
- `id:` - last seen event id. The browser stores this and sends it back as `Last-Event-ID` on reconnect.
- `retry:` - reconnect delay in milliseconds. Overrides the browser's default (3 seconds in most implementations).

Anything starting with `:` is a comment. Servers send `:keepalive\n\n` every 15-30 seconds to stop intermediaries from dropping idle connections.

That is it. No length prefix, no escaping rules beyond "newlines split fields," no binary support. It is a line-oriented text protocol designed to be parseable by hand.

## Why this is enough most of the time

The first instinct when you need server push is to reach for WebSockets. That is usually wrong. Compare what each protocol actually gives you:

| | SSE | WebSockets | Long polling |
| --- | --- | --- | --- |
| Direction | server -> client | bidirectional | client -> server -> client |
| Transport | plain HTTP | upgrade to ws/wss | plain HTTP |
| Reconnect | automatic | you write it | implicit (new request) |
| Message ordering | yes (`id` + `Last-Event-ID`) | yes | manual |
| Binary frames | no (use base64) | yes | no |
| Works through HTTP/1.1 proxy | yes | sometimes | yes |
| Works with HTTP/2 multiplexing | yes | no (HTTP/2 had no upgrade until [RFC 8441](https://datatracker.ietf.org/doc/html/rfc8441)) | yes |
| Browser API | `EventSource` | `WebSocket` | `fetch`/`XHR` |
| Backpressure model | TCP | per-frame | TCP |

SSE wins when:

- The data flows one way (server tells client about state changes).
- You want to send through corporate proxies and CDNs without surprises.
- You want the browser to handle reconnection for you.
- Each message is text or JSON and small.

WebSockets win when you need bidirectional comms in the same connection (chat, multiplayer game state, collaborative editing). Long polling is a fallback for environments where neither is allowed - mostly relevant if you target very old browsers or weird middleboxes.

A useful mental model: WebSockets are TCP-with-frames over HTTP. SSE is a unidirectional log file streamed over HTTP. If you are tempted to send bidirectional SSE by also having the client `POST` to a separate endpoint, you have reinvented WebSockets badly. Use the right tool.

## What "long-lived response" actually means

Here is what makes SSE interesting at the systems level: the server never closes the response body. In Rust that means whatever holds the response writer cannot be dropped, which means whatever future drives that writer cannot complete, which means a tokio task is parked on it for the lifetime of the client.

For each connected client you pay:

- One TCP connection (one socket file descriptor).
- One TLS session if HTTPS (~5-10 KB of state).
- One tokio task (~64 bytes of stack frame plus whatever your handler captures).
- One channel receiver if you fan out events through `tokio::sync::broadcast` or `mpsc`.

A modern Linux box with raised `ulimit -n` and decent network tuning can hold tens of thousands of idle SSE connections per core. The bottleneck is rarely CPU; it is usually file descriptors, kernel socket buffers, and your event fan-out structure. If you are pushing a lot of data, you have to think about backpressure - if one slow client cannot drain its socket fast enough, your sender queue grows. We will get to that.

## The minimal Axum SSE handler

`axum` ships with first-class SSE support via [`axum::response::Sse`](https://docs.rs/axum/0.7/axum/response/sse/index.html). It wraps a [`futures::Stream`](https://docs.rs/futures/latest/futures/stream/trait.Stream.html) of `Event` values and handles all the formatting. Here is a complete server:

```rust
use axum::{
    response::sse::{Event, KeepAlive, Sse},
    routing::get,
    Router,
};
use futures::stream::{self, Stream};
use std::{convert::Infallible, time::Duration};
use tokio_stream::StreamExt as _;

async fn events() -> Sse<impl Stream<Item = Result<Event, Infallible>>> {
    let stream = stream::repeat_with(|| {
        let now = chrono::Utc::now().to_rfc3339();
        Event::default().data(now)
    })
    .map(Ok)
    .throttle(Duration::from_secs(1));

    Sse::new(stream).keep_alive(
        KeepAlive::new()
            .interval(Duration::from_secs(15))
            .text("keep-alive"),
    )
}

#[tokio::main]
async fn main() {
    let app = Router::new().route("/events", get(events));
    let listener = tokio::net::TcpListener::bind("0.0.0.0:3000").await.unwrap();
    axum::serve(listener, app).await.unwrap();
}
```

Cargo.toml:

```toml
[dependencies]
axum = "0.7"
tokio = { version = "1", features = ["full"] }
tokio-stream = { version = "0.1", features = ["time"] }
futures = "0.3"
chrono = "0.4"
```

Hit it with curl:

```
$ curl -N http://localhost:3000/events
data: 2026-05-01T10:00:00.000+00:00

data: 2026-05-01T10:00:01.000+00:00

data: 2026-05-01T10:00:02.000+00:00

```

`-N` disables curl's output buffering so you see lines as they arrive. Without it, curl waits for the response to finish, which it never will.

The `KeepAlive` configuration sends a comment line every 15 seconds. Browsers ignore comments, but they keep TCP intermediaries from declaring the connection dead. Cloudflare, for instance, drops idle HTTP/1.1 connections after 100 seconds; nginx defaults to 60. Without keep-alive comments your stream is fine until traffic gets quiet.

## Fanning out events to many clients

A single-client stream is a toy. The real shape of SSE is "one event source, N subscribers." [`tokio::sync::broadcast`](https://docs.rs/tokio/latest/tokio/sync/broadcast/index.html) is built for exactly this:

```rust
use axum::{
    extract::State,
    response::sse::{Event, KeepAlive, Sse},
    routing::{get, post},
    Json, Router,
};
use futures::stream::Stream;
use serde::{Deserialize, Serialize};
use std::{convert::Infallible, sync::Arc, time::Duration};
use tokio::sync::broadcast;
use tokio_stream::wrappers::BroadcastStream;
use tokio_stream::StreamExt as _;

#[derive(Clone, Serialize, Deserialize)]
struct Notification {
    id: u64,
    title: String,
    body: String,
}

#[derive(Clone)]
struct AppState {
    tx: broadcast::Sender<Notification>,
    counter: Arc<std::sync::atomic::AtomicU64>,
}

async fn subscribe(
    State(state): State<AppState>,
) -> Sse<impl Stream<Item = Result<Event, Infallible>>> {
    let rx = state.tx.subscribe();
    let stream = BroadcastStream::new(rx).filter_map(|res| {
        let n = res.ok()?;
        Some(Ok(Event::default()
            .id(n.id.to_string())
            .event("notification")
            .data(serde_json::to_string(&n).unwrap())))
    });

    Sse::new(stream).keep_alive(
        KeepAlive::new()
            .interval(Duration::from_secs(15))
            .text("ka"),
    )
}

async fn publish(
    State(state): State<AppState>,
    Json(mut n): Json<Notification>,
) -> &'static str {
    n.id = state.counter.fetch_add(1, std::sync::atomic::Ordering::Relaxed);
    let _ = state.tx.send(n);
    "ok"
}

#[tokio::main]
async fn main() {
    let (tx, _) = broadcast::channel::<Notification>(1024);
    let state = AppState {
        tx,
        counter: Arc::new(0.into()),
    };
    let app = Router::new()
        .route("/events", get(subscribe))
        .route("/publish", post(publish))
        .with_state(state);
    let listener = tokio::net::TcpListener::bind("0.0.0.0:3000").await.unwrap();
    axum::serve(listener, app).await.unwrap();
}
```

The `broadcast` channel has a fixed-size ring buffer. When a subscriber falls behind by more than the buffer, the stream yields `Err(BroadcastStreamRecvError::Lagged(n))` and skips ahead. The `filter_map` above swallows lag errors. In production you probably want to log them and possibly send a synthetic event so the client knows something was dropped.

This is the SSE backpressure story in a nutshell: TCP gives each client its own `tokio::io::AsyncWrite`. If a client is slow, its writes block. If the writes block, the per-client stream stops polling the broadcast receiver. If the receiver stops draining, it falls behind. The broadcast buffer rolls over and lag is reported. You lose events to that client, but the whole server does not stall.

If losing events is not acceptable, do not use `broadcast`. Give each subscriber an `mpsc::channel` with a deeper buffer, accept the per-client memory cost, and tear down clients that exceed a threshold.

## Handling reconnection - Last-Event-ID

This is the part most tutorials skip. SSE is reliable only if you implement resume.

When the connection drops, the browser waits `retry` milliseconds, then reconnects. On the new request it sends the most recent `id` it saw as a header:

```
GET /events HTTP/1.1
Last-Event-ID: 42
Accept: text/event-stream
```

Your server is supposed to deliver everything after id 42 and then resume the live stream. If you just hook the new connection to the live broadcast, anything that happened between the disconnect and the reconnect is gone.

The fix is a bounded ring buffer of recent events on the server, keyed by id. On connect, you replay anything newer than `Last-Event-ID`, then attach the live stream:

```rust
use axum::{
    extract::State,
    http::HeaderMap,
    response::sse::{Event, KeepAlive, Sse},
};
use futures::stream::{self, Stream, StreamExt};
use std::{collections::VecDeque, convert::Infallible, sync::Arc, time::Duration};
use tokio::sync::{broadcast, RwLock};

#[derive(Clone)]
struct History {
    inner: Arc<RwLock<VecDeque<Notification>>>,
    cap: usize,
}

impl History {
    fn new(cap: usize) -> Self {
        Self {
            inner: Arc::new(RwLock::new(VecDeque::with_capacity(cap))),
            cap,
        }
    }
    async fn push(&self, n: Notification) {
        let mut g = self.inner.write().await;
        if g.len() == self.cap {
            g.pop_front();
        }
        g.push_back(n);
    }
    async fn since(&self, last_id: u64) -> Vec<Notification> {
        let g = self.inner.read().await;
        g.iter().filter(|n| n.id > last_id).cloned().collect()
    }
}

async fn subscribe(
    State(state): State<AppState>,
    headers: HeaderMap,
) -> Sse<impl Stream<Item = Result<Event, Infallible>>> {
    let last_id: u64 = headers
        .get("last-event-id")
        .and_then(|v| v.to_str().ok())
        .and_then(|s| s.parse().ok())
        .unwrap_or(0);

    let replay = state.history.since(last_id).await;
    let replay_stream = stream::iter(replay.into_iter().map(|n| {
        Ok(Event::default()
            .id(n.id.to_string())
            .event("notification")
            .data(serde_json::to_string(&n).unwrap()))
    }));

    let live_rx = state.tx.subscribe();
    let live_stream = tokio_stream::wrappers::BroadcastStream::new(live_rx)
        .filter_map(|res| async move {
            let n = res.ok()?;
            Some(Ok(Event::default()
                .id(n.id.to_string())
                .event("notification")
                .data(serde_json::to_string(&n).unwrap())))
        });

    Sse::new(replay_stream.chain(live_stream)).keep_alive(
        KeepAlive::new().interval(Duration::from_secs(15)),
    )
}
```

Two things to notice. First, you must subscribe to the live channel **before** you read history, otherwise events that arrive during replay are lost. The order in the snippet is wrong on purpose to make it easy to spot - real code should call `subscribe()` first, then read history. Second, `chain` joins the two streams in order: history first, live after. Once history is exhausted, the live stream takes over.

For long-lived state (server restarts) the history needs to survive. Persist events to a database or append-only log, and on connect, query "give me everything after id X." This is what tools like [Mercure](https://mercure.rocks/) and [NATS](https://nats.io/) JetStream do under the hood.

## The client side - EventSource

The browser side is shorter than any of the server snippets:

```javascript
const es = new EventSource("/events");

es.addEventListener("notification", (e) => {
  const n = JSON.parse(e.data);
  console.log("new notification", n);
});

es.onerror = (e) => {
  console.log("disconnected; browser will retry");
};
```

The `EventSource` constructor is part of the [HTML spec](https://html.spec.whatwg.org/multipage/server-sent-events.html#the-eventsource-interface) and shipped in browsers since 2011. It does the heavy lifting:

- Issues the GET with `Accept: text/event-stream`.
- Parses the line-by-line format.
- Tracks the last `id` and resends it as `Last-Event-ID` on reconnect.
- Uses the most recent `retry:` value as the reconnect delay.
- Fires a `message` event for events without a name, and a custom event for named ones.

There is one gotcha: `EventSource` does not let you set request headers, which means you cannot send `Authorization: Bearer ...`. Workarounds are token in query string (visible in logs), cookie auth (CSRF concerns), or use the [`fetch` event-stream](https://developer.mozilla.org/en-US/docs/Web/API/Streams_API) approach with manual reconnection. The [Microsoft `fetch-event-source` library](https://github.com/Azure/fetch-event-source) is a common drop-in for that case.

For non-browser clients (Rust, Go, mobile), the most common approach is to read the response line by line and parse the four fields by hand. The format is so simple that "by hand" is 30 lines of code. The [`eventsource-stream`](https://crates.io/crates/eventsource-stream) crate does it for you in async Rust.

## What the bytes actually look like with HTTP/2

A subtle benefit of SSE is that it composes with HTTP/2 multiplexing. Over HTTP/1.1, each SSE connection takes a TCP socket, and browsers cap concurrent connections per origin at 6. That means a single tab with seven SSE streams blocks. Over HTTP/2, all seven multiplex onto one TCP socket and the cap is per-stream (~100 by default in nginx, configurable), not per-connection.

The wire format does not change - the response body is still `text/event-stream` with the same lines. The transport just chunks them differently: HTTP/2 `DATA` frames instead of HTTP/1.1 chunked encoding. If you want to see the exact frames, run [`nghttp -v https://...`](https://nghttp2.org/documentation/nghttp.1.html) against your server.

WebSockets do not get this for free. The original WebSocket spec required HTTP/1.1 Upgrade, which HTTP/2 does not support. RFC 8441 (2018) added the `:protocol` extended CONNECT method to make WebSockets work over HTTP/2, but support is still uneven. If your reverse proxy sits between the client and your app and downgrades to HTTP/1.1 internally, you give up multiplexing.

## Failure modes worth knowing

A few things that will bite if you do not plan for them:

- **Buffering proxies.** Some HTTP/1.1 proxies buffer responses until they have a complete chunk or timeout. Cloudflare, for example, used to buffer compressed responses. Set `Cache-Control: no-cache, no-transform` and `X-Accel-Buffering: no` (the nginx-specific header that disables proxy buffering) to be safe.
- **Compression.** `Content-Encoding: gzip` over a stream produces a stream of gzip frames, which is fine in theory but breaks badly when intermediaries try to be clever. Disable compression on `text/event-stream` responses. Axum does not compress by default; if you added a compression layer, exclude this content type.
- **Browser tab limits.** With HTTP/1.1, six SSE streams to the same origin saturate Chrome's connection pool. Open a seventh tab to your dashboard and it hangs. Either move to HTTP/2, or use a single stream and multiplex events with a `type` field in your payload.
- **Mobile background tabs.** iOS Safari kills background tab connections aggressively. Your `Last-Event-ID` resume logic is what makes this invisible to users.
- **Load balancer idle timeouts.** AWS ALB defaults to 60 seconds. If your keep-alive interval is 90 seconds, the ALB cuts the connection before your first ping. Match the interval to less than half the LB timeout.

## When to actually use SSE

If you have one of these problems, SSE is the right tool:

- Live dashboard refreshing every few seconds.
- Notification feed (new comment, friend request, alert).
- Build/deploy/job log tailing.
- Progress bar for a long server-side operation.
- Streaming LLM tokens to a chat UI - this is what OpenAI's API uses for `stream=true`. The client side is `EventSource` with the lines parsed as JSON deltas.

If your problem is bidirectional (chat, collaborative editor, multiplayer state sync), use WebSockets. If you need a binary protocol (live audio, video frames), use WebSockets or WebTransport. If you need browser support older than IE 11, use long polling.

For everything else, SSE is the lower-effort choice with better failure modes. The browser API is one constructor call, the server is a `Stream<Item = Event>`, and the wire format is the kind of thing you can read out of a `tcpdump` and understand without a spec open. Most "real-time" features do not need a socket. They need a long response and a server that knows what to write into it.
