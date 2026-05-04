+++
title = "HTTP caching - ETag, Cache-Control, and conditional requests"
date = 2025-10-11
description = "A practical deep dive into HTTP caching: the exact bytes of Cache-Control, how ETags are computed and compared, what a 304 actually looks like, and how to wire it up in Axum."

[taxonomies]
tags = ["http", "caching", "axum", "performance"]
+++

The fastest HTTP request is the one you never send. The second fastest is the one that returns 304 Not Modified with an empty body. Most of what we call "making an API fast" is actually convincing clients, reverse proxies, and CDNs not to do work they already did five minutes ago.

HTTP caching is old - [RFC 2616](https://datatracker.ietf.org/doc/html/rfc2616) from 1999, then reorganized into RFC 7234 and now [RFC 9111](https://datatracker.ietf.org/doc/html/rfc9111) - but it is one of the places where reading "sort of knowing it" vs "actually knowing what the headers do" produces a 10x difference in latency. If you are not familiar with what happens on the wire when curl hits a URL, I covered that in [Understanding TCP/IP - what happens when you curl a URL](/blog/understanding-tcp-ip-what-happens-when-you-curl-a-url). This post zooms in on the HTTP application layer and the four or five headers that decide whether a response is served from memory, from a nearby CDN edge, or re-fetched from origin.

<!-- more -->

## Who the cache actors are

When you send a GET, there are at least four places that could satisfy it without your server doing anything:

1. **The browser's memory / disk cache** - a `Cache-Control: max-age=3600` response is kept in the client.
2. **A forward proxy** - corporate networks, some ISPs.
3. **A CDN edge node** - Cloudflare, Fastly, CloudFront. These are "shared caches" in RFC terminology.
4. **A reverse proxy** - nginx, Varnish, Caddy sitting in front of your app.

RFC 9111 splits caches into **private** (one user) and **shared** (many users). This distinction matters: a response containing `Authorization` headers or a `Set-Cookie` with a session token must not be stored in a shared cache by default, because the next user would see someone else's data.

You control where a response can be stored, how long for, and what the client should do when it expires. All of that lives in one header: `Cache-Control`.

## Cache-Control - what each directive actually does

`Cache-Control` is a comma-separated list of directives. The common ones:

```
Cache-Control: public, max-age=3600, stale-while-revalidate=60
```

- `public` - any cache may store this, including shared ones. Needed if the response has an `Authorization` header but you still want CDNs to cache it.
- `private` - only the end-user cache (browser). CDNs and proxies must not store it. Use this for user-specific responses you still want the browser to remember.
- `no-cache` - you **can** store it, but you **must** revalidate with the origin before reuse. This is not "do not cache" despite the name. It pairs with ETag or Last-Modified.
- `no-store` - do not write to disk or memory at all. This is what "do not cache" actually means. Use for responses containing secrets you do not want sitting in `/var/cache/`.
- `max-age=N` - seconds the response is "fresh". While fresh, it is served from cache with no network call.
- `s-maxage=N` - overrides `max-age` for shared caches only. Common pattern: `max-age=60, s-maxage=3600` means browsers re-check every minute but the CDN holds it for an hour.
- `must-revalidate` - once stale, you cannot serve it without checking origin. By default, some caches will serve stale on network errors; `must-revalidate` forbids that.
- `immutable` - the response will never change as long as the URL is the same. Browser should not revalidate even when the user presses reload. This is what fingerprinted assets like `app.a8b3c9.js` ship with.
- `stale-while-revalidate=N` - serve stale for up to N seconds past expiry while asynchronously refetching in the background.
- `stale-if-error=N` - if origin returns 5xx, keep serving stale for N seconds.

The browser's freshness check is arithmetic:

```
age = now - response_received_time + Age_header
fresh = age < max_age
```

If `fresh`, the browser returns the cached response directly, no packets on the wire. If not fresh, the cache does not immediately discard the response; it revalidates, which is where ETag comes in.

## ETag - the content fingerprint

An `ETag` is an opaque string the server attaches to a response. The spec says nothing about how you compute it - it just has to change when the representation changes.

```
HTTP/1.1 200 OK
Content-Type: application/json
ETag: "a8b3c9d14f22"
Cache-Control: max-age=60
Content-Length: 412

{"id": 42, ...}
```

There are strong and weak ETags. A strong ETag (`"abc"`) means byte-for-byte identical. A weak ETag (`W/"abc"`) means semantically equivalent - same content, maybe different whitespace or header ordering. Use weak ETags when you compress on the fly, because gzip is non-deterministic at different compression levels.

How do you pick the string? Three common strategies:

1. **Hash of the body** - `sha256(body)[:16]`. Simple, always correct, costs CPU per request.
2. **Version + updated_at** - for database rows, something like `"v2-1718236612"`. Cheap, no hashing, but you must bump it on every mutation.
3. **Revision id** - git commit hash for static assets, UUID rotated on every deploy.

The hash approach is worth its cost because it composes. If nothing changed, the hash did not change. If anything changed - a field, an order, whitespace - the hash changed. No manual bookkeeping.

When a client has a cached response with ETag `"a8b3c9"` and wants to revalidate, it sends:

```
GET /api/posts/42 HTTP/1.1
If-None-Match: "a8b3c9d14f22"
```

The server computes the current ETag. If it matches, it returns:

```
HTTP/1.1 304 Not Modified
ETag: "a8b3c9d14f22"
Cache-Control: max-age=60
```

No body. This is the whole point. The response is ~100 bytes of headers vs potentially megabytes of JSON. The client knows its cached copy is still valid and reuses it.

If the ETag does not match, the server returns a normal 200 with the full body and the new ETag.

`If-None-Match` can take multiple ETags separated by commas, which is used for "here are all the ETags I have cached, return 304 if any matches." It can also be `*`, which means "any representation" and is mostly used with `PUT` for optimistic concurrency.

## Last-Modified and If-Modified-Since

Before ETags, HTTP had `Last-Modified`:

```
HTTP/1.1 200 OK
Last-Modified: Tue, 21 Apr 2026 18:42:17 GMT
```

The client revalidates with:

```
GET /api/posts/42 HTTP/1.1
If-Modified-Since: Tue, 21 Apr 2026 18:42:17 GMT
```

If the resource has not been modified since that timestamp, server returns 304. Otherwise, 200 with a new timestamp.

Problems with `Last-Modified`:

- Only one-second resolution. Two updates within the same second look unchanged.
- Timestamps are easy to get wrong - server clock skew, filesystems returning mtime in the wrong timezone, databases storing `updated_at` with implicit timezone conversion.
- Content revert: if you rollback an edit to the previous version, the timestamp is newer but the content is older than a cached copy.

ETags avoid all of this. In practice, use ETag when you can and fall back to `Last-Modified` for content served from a filesystem where computing a hash is expensive.

If both headers are present, RFC 9111 says `If-None-Match` takes precedence.

## The 304 on the wire

For the curious, here is what a revalidation cycle looks like in Wireshark. Client sends:

```
GET /api/posts/42 HTTP/1.1
Host: api.example.com
If-None-Match: "a8b3c9d14f22"
Accept: application/json

```

Server responds:

```
HTTP/1.1 304 Not Modified
Date: Wed, 22 Apr 2026 10:15:43 GMT
ETag: "a8b3c9d14f22"
Cache-Control: max-age=60

```

That is 113 bytes of response vs whatever the JSON payload was. On a mobile connection with 200ms RTT, you still pay the round trip, but you save the transfer time. Which is why `max-age` and `stale-while-revalidate` exist - they avoid the round trip entirely.

## Setting cache headers in Axum

Axum gives you `HeaderMap` and `IntoResponse`, so adding cache headers is straightforward. The cleanest way is a middleware layer that computes ETag automatically, plus per-route `Cache-Control`.

Here is a minimal Axum 0.7 handler with manual headers:

```rust
use axum::{
    http::{header, HeaderMap, StatusCode},
    response::IntoResponse,
    routing::get,
    Router,
};
use serde::Serialize;
use sha2::{Digest, Sha256};

#[derive(Serialize)]
struct Post {
    id: u64,
    title: String,
}

async fn get_post(headers: HeaderMap) -> impl IntoResponse {
    let post = Post {
        id: 42,
        title: "HTTP caching".to_string(),
    };

    let body = serde_json::to_vec(&post).unwrap();
    let etag = format!("\"{:x}\"", {
        let mut h = Sha256::new();
        h.update(&body);
        let digest = h.finalize();
        u64::from_be_bytes(digest[..8].try_into().unwrap())
    });

    if let Some(inm) = headers.get(header::IF_NONE_MATCH) {
        if inm.to_str().map(|s| s == etag).unwrap_or(false) {
            return (
                StatusCode::NOT_MODIFIED,
                [
                    (header::ETAG, etag.as_str()),
                    (header::CACHE_CONTROL, "public, max-age=60"),
                ],
            )
                .into_response();
        }
    }

    (
        StatusCode::OK,
        [
            (header::CONTENT_TYPE, "application/json"),
            (header::ETAG, etag.as_str()),
            (header::CACHE_CONTROL, "public, max-age=60, stale-while-revalidate=30"),
        ],
        body,
    )
        .into_response()
}
```

This works but repeats the ETag logic for every handler. Extract it into a `tower` layer:

```rust
use axum::{
    body::{Body, to_bytes},
    http::{header, Request, Response, StatusCode},
    middleware::Next,
};
use sha2::{Digest, Sha256};

pub async fn etag_middleware(req: Request<Body>, next: Next) -> Response<Body> {
    let if_none_match = req
        .headers()
        .get(header::IF_NONE_MATCH)
        .and_then(|v| v.to_str().ok())
        .map(|s| s.to_string());

    let resp = next.run(req).await;

    // Only cache successful GETs with a body we can buffer.
    if resp.status() != StatusCode::OK {
        return resp;
    }

    let (parts, body) = resp.into_parts();
    let bytes = match to_bytes(body, 1024 * 1024).await {
        Ok(b) => b,
        Err(_) => return Response::from_parts(parts, Body::empty()),
    };

    let etag = {
        let mut h = Sha256::new();
        h.update(&bytes);
        let d = h.finalize();
        format!("\"{}\"", hex::encode(&d[..8]))
    };

    let mut parts = parts;
    parts.headers.insert(header::ETAG, etag.parse().unwrap());

    if if_none_match.as_deref() == Some(&etag) {
        parts.status = StatusCode::NOT_MODIFIED;
        return Response::from_parts(parts, Body::empty());
    }

    Response::from_parts(parts, Body::from(bytes))
}
```

Wire it in:

```rust
let app = Router::new()
    .route("/api/posts/:id", get(get_post))
    .layer(axum::middleware::from_fn(etag_middleware));
```

Two caveats. First, this buffers the response body in memory to hash it, so it is inappropriate for streaming responses. For large payloads, skip the middleware and compute ETags from the underlying data (e.g. `sha256(row.id + row.updated_at)`). Second, you are hashing after serialization, so if your JSON field ordering is not stable, the ETag flips on every request - `serde_json` with BTreeMap-backed structs gives you stability.

For the `Cache-Control` header itself, route-level configuration is simpler than middleware. A typical pattern:

```rust
use axum::http::header;

async fn static_asset() -> impl IntoResponse {
    (
        [(header::CACHE_CONTROL, "public, max-age=31536000, immutable")],
        include_bytes!("../static/app.js").as_slice(),
    )
}

async fn api_response() -> impl IntoResponse {
    (
        [(header::CACHE_CONTROL, "private, max-age=0, must-revalidate")],
        "user-specific data",
    )
}
```

`max-age=31536000` is one year, the maximum practical value. Combined with `immutable`, the browser never revalidates. This only works if the URL is content-addressable - if `app.js` changes, you rename it to `app.a8b3c9.js`.

## CDN caching - the shared layer

Put Cloudflare or Fastly in front of your origin and suddenly your `Cache-Control` header speaks to a different audience. CDNs respect standard headers but also introduce their own:

- `Cache-Control: s-maxage=3600` - CDN caches for an hour, browser reads `max-age`.
- `Surrogate-Control: max-age=3600` - Fastly-specific, overrides `Cache-Control` at the edge only.
- `CDN-Cache-Control: max-age=3600` - [draft RFC](https://datatracker.ietf.org/doc/html/draft-cdn-control-header) that Cloudflare, Akamai, and Fastly all implement. Highest precedence for CDN, invisible to browser.

Most useful combo for an API:

```
Cache-Control: public, max-age=0, s-maxage=300, stale-while-revalidate=60
```

- Browser: always revalidate (max-age=0).
- CDN: cache 5 minutes, serve stale for 60s more while refetching.

The browser does a cheap conditional GET to the CDN edge (~10ms), which serves 304 from cache. The CDN does the real revalidation to origin once every 5 minutes across all users. Your origin sees a tiny fraction of real traffic.

For this to work, your origin must emit ETags and handle `If-None-Match` - CDNs will forward conditional requests upstream if their copy is stale.

One detail: the `Vary` header. If a response depends on `Accept-Encoding`, `Accept-Language`, or `Authorization`, the cache must key on those headers plus the URL. Forget `Vary: Accept-Encoding` and a client requesting gzip can get served the identity version someone else cached.

```
Cache-Control: public, max-age=300
Vary: Accept-Encoding, Accept-Language
```

Every value in `Vary` multiplies the number of cache entries, so keep it minimal.

## Cache invalidation - the hard part

Phil Karlton's quote about two hard problems in computer science was half joking but he was right about this one. The cache is fast because it is stale; the cost is that sometimes "stale" means "wrong."

Strategies, from easiest to hardest:

1. **Content addressing.** Make the URL change when the content changes. `app.a8b3c9.js` → `app.de7f12.js`. The old URL is still valid, nothing needs to be invalidated. This is how every modern frontend build works.

2. **Short TTLs.** Set `max-age=60` and accept that mutations take up to a minute to propagate. Good for feed-style content where "a minute late" is fine.

3. **ETag-based revalidation.** Long `max-age` but clients must revalidate. This trades storage (cache still holds the entry) for correctness - every request is a conditional GET, so updates propagate immediately but you still save bandwidth.

4. **Purge APIs.** Cloudflare has `/zones/:id/purge_cache`, Fastly has `PURGE https://...`. Your app calls this after a write. Works, but introduces a distributed system - what if the purge succeeds at US-East but fails in Tokyo? For seconds to minutes, users in Tokyo see stale data.

5. **Tagged invalidation.** Fastly's Surrogate-Key header lets you tag responses with keys, then purge everything with that tag in one call. "Purge all responses tagged `user:42`" becomes atomic from your app's point of view.

Invalidation goes wrong because caches are distributed and eventual. If correctness matters (billing, access control, inventory), do not rely on cache invalidation - either do not cache at all (`Cache-Control: no-store`) or use short TTLs with revalidation.

## Stale-while-revalidate - the quiet winner

`stale-while-revalidate` ([RFC 5861](https://datatracker.ietf.org/doc/html/rfc5861)) is underused. It is the directive that makes "fresh" a probability rather than a binary.

```
Cache-Control: max-age=60, stale-while-revalidate=300
```

Behavior:

- Seconds 0-60: fresh, served from cache, no network.
- Seconds 60-360: stale but usable. Served from cache immediately, revalidation happens in background.
- Seconds 360+: must revalidate synchronously.

For the user, this means 60+300 = 360 seconds of zero-latency responses. For you, it means a read spike does not amplify to origin - only the first stale request triggers a refetch, everyone else gets the old copy until that refetch completes.

Browsers ([supported since Chrome 75](https://web.dev/articles/stale-while-revalidate)) and CDNs (Cloudflare, Fastly, Vercel) all honor this. The one gotcha: while stale, the user sees data that is potentially up to `max-age + swr` seconds old. For a blog post, fine. For a bank balance, not fine.

## What to actually do

If you take one thing away:

- Static assets: fingerprint the URL, `Cache-Control: public, max-age=31536000, immutable`.
- API responses: `public, max-age=0, s-maxage=60, stale-while-revalidate=30` plus an ETag. Revalidation is cheap, origin hits are rare.
- User-specific: `private, max-age=0, must-revalidate` plus ETag. Browser caches, no CDN, conditional GETs.
- Secrets: `no-store`. Never.

Measure with `curl -v` and watch for `HIT`, `MISS`, `STALE` headers (Cloudflare returns `cf-cache-status`, Fastly uses `x-cache`). If you see `MISS` on repeat requests, your `Vary` header is wrong or your `Cache-Control` is too restrictive.

And if you are wondering whether to bother - a site with 10,000 rps where 90% of requests are cacheable, switching from "always hit origin" to "ETag + max-age=60 + CDN" typically drops origin load by 100x and p50 latency by 5-10x. There is almost no other single change with that kind of leverage.
