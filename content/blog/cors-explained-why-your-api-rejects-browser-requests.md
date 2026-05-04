+++
title = "CORS explained - why your API rejects browser requests"
date = 2025-08-02
description = "Same-origin policy, preflight, and the exact headers the browser checks. Includes a working Axum + tower-http CorsLayer setup and a tour of the mistakes that cost people afternoons."
[taxonomies]
tags = ["http", "security", "axum", "browser"]
+++

Every backend developer eventually hits the same wall: `curl` works, Postman works, the frontend works on the colleague's machine, and then the browser console shows a red line that says `blocked by CORS policy`. The request never reaches your handler. The server logs show nothing. The network tab shows an `OPTIONS` request you did not write, with a response your server did not produce the way the browser wanted.

CORS is not an error. It is a browser feature that does exactly what the spec says, and when it blocks you it almost always means your server replied in a way that the [Fetch Standard](https://fetch.spec.whatwg.org/#cors-protocol) says is not allowed. If you are not familiar with what actually goes on the wire when a browser makes a request, I covered that in [Understanding TCP/IP - what happens when you curl a URL](/blog/understanding-tcp-ip-what-happens-when-you-curl-a-url). This post zooms into the browser half: what CORS protects, how preflight works byte by byte, every header involved, and how to configure it correctly in Axum with `tower-http`.

<!-- more -->

## Why CORS exists - the same-origin policy

Browsers enforce the **same-origin policy** (SOP). An origin is the triple `(scheme, host, port)`:

```
https://example.com:443      origin A
https://api.example.com:443  origin B  (different host)
http://example.com:443       origin C  (different scheme)
https://example.com:8443     origin D  (different port)
```

By default, JavaScript running on origin A cannot read responses from origins B, C, or D. This is not about sending the request - the browser will happily send it - it is about letting the page read the result. Without SOP, any site you visit could fetch `https://bank.com/account` using the cookies your browser stores for `bank.com` and read your balance. The rule exists to contain the damage a malicious page can do.

CORS is the opt-in. When `api.example.com` wants to let `example.com` read its responses, it adds headers that say "this other origin is allowed to see me." The browser reads those headers and decides to either hand the response to the JavaScript that asked for it, or throw a `TypeError` into the fetch promise.

The crucial detail: **CORS is enforced by the browser, not the server**. Your server does not reject anything. Your server sends a response, and the browser refuses to pass it to the page. If you want to test whether your server is working at all, `curl` it. `curl` has no same-origin policy.

## Simple vs preflighted requests

The Fetch spec splits cross-origin requests into two classes.

A **simple request** (the spec calls it "CORS-safelisted") is one where:

- The method is `GET`, `HEAD`, or `POST`
- The only headers set by JavaScript are `Accept`, `Accept-Language`, `Content-Language`, `Content-Type`, `Range`, or a handful of other safelisted ones
- If `Content-Type` is set, it is `application/x-www-form-urlencoded`, `multipart/form-data`, or `text/plain`

These three content types are the historical forms a `<form>` element could submit before the Fetch API existed. Since a malicious page could already send them via an invisible form, the spec does not require a preflight - the damage was already possible. The browser just sends the request and checks the response headers afterwards.

Everything else is **preflighted**: `PUT`, `DELETE`, `PATCH`, anything with a custom header like `Authorization: Bearer ...` or `X-Trace-Id`, and crucially anything with `Content-Type: application/json`. Since you cannot send JSON from a form, the browser treats it as a "non-simple" request and asks for permission first.

## What a preflight actually looks like on the wire

Say your React app at `https://app.example.com` does:

```js
await fetch("https://api.example.com/users/42", {
  method: "PATCH",
  headers: {
    "Content-Type": "application/json",
    "Authorization": "Bearer eyJhbGc...",
  },
  body: JSON.stringify({ name: "Ada" }),
});
```

Before the `PATCH`, the browser sends:

```
OPTIONS /users/42 HTTP/1.1
Host: api.example.com
Origin: https://app.example.com
Access-Control-Request-Method: PATCH
Access-Control-Request-Headers: authorization,content-type
```

Three headers matter:

- `Origin` - who is asking
- `Access-Control-Request-Method` - what method the real request will use
- `Access-Control-Request-Headers` - a comma-separated list of non-simple headers, lowercased

The server answers:

```
HTTP/1.1 204 No Content
Access-Control-Allow-Origin: https://app.example.com
Access-Control-Allow-Methods: GET, POST, PATCH, DELETE
Access-Control-Allow-Headers: authorization, content-type
Access-Control-Max-Age: 600
Vary: Origin
```

Five things the browser will now verify:

1. Is the requesting `Origin` in the `Access-Control-Allow-Origin` value (or is it `*`)?
2. Is the requested method listed in `Access-Control-Allow-Methods`?
3. Is every requested header listed (case-insensitively) in `Access-Control-Allow-Headers`?
4. If the request uses credentials, are they allowed (more below)?
5. Does `Access-Control-Max-Age` let us skip preflight for a while?

If all checks pass, the browser caches the permission for `Max-Age` seconds and fires the real `PATCH`. If any check fails, the promise rejects and the real request is never sent.

The response to the actual `PATCH` must also carry `Access-Control-Allow-Origin`. Preflight allows you to send the request, but the browser verifies the response headers a second time. Forgetting this is one of the most common mistakes - preflight succeeds, the actual call returns 200, but the browser still blocks it because the 200 response is missing `Access-Control-Allow-Origin`.

## The credentials rule

By default, `fetch` does not send cookies or HTTP auth to a cross-origin endpoint. You have to opt in:

```js
fetch("https://api.example.com/me", { credentials: "include" });
```

The moment you do, two things change. The browser attaches cookies. The server must now satisfy two extra conditions or the response gets dropped:

1. `Access-Control-Allow-Credentials: true`
2. `Access-Control-Allow-Origin` must be a **specific origin**, never `*`

`Access-Control-Allow-Origin: *` is forbidden with credentials. This is deliberate - if `*` worked with cookies, any malicious site could read authenticated responses from your API. You must echo the exact requesting origin back, which means the server needs to maintain an allowlist and pick the matching entry:

```
if Origin in allowlist:
    respond Access-Control-Allow-Origin: <that origin>
    respond Vary: Origin
```

The `Vary: Origin` header tells caches (your CDN, the browser's disk cache, nginx) that the response differs per origin. Without it, a cache might return the `Access-Control-Allow-Origin: https://app.a.com` response to a request from `https://app.b.com`, and the browser would block that.

## Access-Control-Expose-Headers

By default, JavaScript can only read a short list of **CORS-safelisted response headers**: `Cache-Control`, `Content-Language`, `Content-Length`, `Content-Type`, `Expires`, `Last-Modified`, and `Pragma`. Anything else - `X-Request-Id`, `X-RateLimit-Remaining`, a custom `X-Trace-Id` - is filtered out of the Response object even though it arrived over the wire. You have to expose it:

```
Access-Control-Expose-Headers: x-request-id, x-ratelimit-remaining
```

This is why your rate limit dashboard in the frontend shows `null`. The header is there, the browser just will not hand it to you until you say so on the server.

## Common mistakes and how they fail

A quick tour of the ways CORS eats your afternoon:

**1. Returning `Access-Control-Allow-Origin: *` with credentials.** Browser drops the response silently. Fix: echo the specific origin, keep an allowlist.

**2. Missing `Access-Control-Allow-Origin` on the actual response after a successful preflight.** Preflight is only the permission slip; the real response also needs the header. Fix: put the CORS layer in front of every route, not only on `OPTIONS`.

**3. Wrong case on header names in `Access-Control-Allow-Headers`.** The spec says the comparison is ASCII case-insensitive, but some libraries used to be strict. Today all major browsers lowercase, but your allowlist in code should too.

**4. Custom header `Authorization` not listed.** The browser sees you trying to set `Authorization` on a cross-origin request, adds it to `Access-Control-Request-Headers`, and if your server does not echo it back in `Access-Control-Allow-Headers` preflight fails. `Authorization` is not on the simple-request safelist.

**5. Trailing slashes in the allowlist.** `https://app.example.com/` is not a valid origin. Origin is scheme + host + port, nothing else. Strip trailing slashes when you build the allowlist.

**6. Proxy strips the headers.** Your app returns the right headers; nginx or a load balancer in front of it filters them out. `curl -I` against your public URL before blaming the app.

**7. Redirects through a third origin.** If the browser follows a redirect to another origin, that second origin must also satisfy CORS. Preflight cannot carry across redirects for non-simple methods - you have to avoid the redirect.

**8. `Vary: Origin` missing behind a shared cache.** CDN serves the cached CORS headers for one origin to a different origin. Always add `Vary: Origin` when the allowed origin is dynamic.

## Configuring CORS in Axum with tower-http

`tower-http` ships a `CorsLayer` that handles all of the above. Add it to `Cargo.toml`:

```toml
[dependencies]
axum = "0.7"
tower-http = { version = "0.5", features = ["cors"] }
tokio = { version = "1", features = ["full"] }
```

The minimal version:

```rust
use axum::{routing::get, Router};
use http::{header, Method};
use tower_http::cors::{Any, CorsLayer};

#[tokio::main]
async fn main() {
    let cors = CorsLayer::new()
        .allow_origin(Any)
        .allow_methods([Method::GET, Method::POST])
        .allow_headers([header::CONTENT_TYPE]);

    let app = Router::new()
        .route("/hello", get(|| async { "hi" }))
        .layer(cors);

    let listener = tokio::net::TcpListener::bind("0.0.0.0:3000").await.unwrap();
    axum::serve(listener, app).await.unwrap();
}
```

`Any` becomes `*` on the wire. This is fine for a public, credentialless API.

For a real app with cookies, you need an allowlist plus credentials:

```rust
use axum::http::HeaderValue;
use tower_http::cors::{AllowOrigin, CorsLayer};

let origins = [
    "https://app.example.com".parse::<HeaderValue>().unwrap(),
    "https://admin.example.com".parse::<HeaderValue>().unwrap(),
];

let cors = CorsLayer::new()
    .allow_origin(AllowOrigin::list(origins))
    .allow_methods([Method::GET, Method::POST, Method::PATCH, Method::DELETE])
    .allow_headers([header::AUTHORIZATION, header::CONTENT_TYPE])
    .allow_credentials(true)
    .expose_headers([header::HeaderName::from_static("x-request-id")])
    .max_age(std::time::Duration::from_secs(600));
```

A few things the layer does for you:

- Answers `OPTIONS` requests automatically with the right headers, before they hit your handlers.
- Echoes the requesting origin when you pass a list, and adds `Vary: Origin`.
- Rejects the `allow_origin(Any)` + `allow_credentials(true)` combination at construction time - `tower-http` will panic, which is the correct behavior because that combination is invalid per spec.
- Filters `Access-Control-Allow-Headers` case-insensitively.

For dynamic origins - say you want to allow any subdomain of `example.com` - use a predicate:

```rust
use tower_http::cors::AllowOrigin;

let cors = CorsLayer::new()
    .allow_origin(AllowOrigin::predicate(|origin, _request_parts| {
        origin.as_bytes().ends_with(b".example.com")
            || origin == "https://example.com"
    }))
    .allow_credentials(true);
```

The predicate runs per request. Keep it cheap - for a long allowlist, use a `HashSet` and check in O(1).

## When CORS does not apply

CORS is a **browser** mechanism. These cases have no CORS check:

- **Server-to-server calls.** Your backend calling another backend with `reqwest` or `curl` is not a browser, has no Origin header (well, it can, but nobody checks), and the other server will not enforce SOP on it. This is why your frontend sometimes proxies calls through your own backend: you sidestep CORS entirely at the cost of a hop.
- **`<img>`, `<script>`, `<link>` tags.** The classic tags can load cross-origin resources with no CORS check, because they predate CORS. This is also why they are the source of CSRF issues. A `<script src="https://evil.com/x.js">` executes in your origin. CORS for `<script>` exists via the `crossorigin` attribute for subresource integrity, but by default it does not apply.
- **Top-level navigations.** Clicking a link or submitting a `<form>` to a different origin is not a CORS situation - the browser navigates the top-level document, which resets the origin.
- **Service workers, WebSockets, `EventSource`.** These have their own cross-origin rules that are related but distinct. WebSockets check `Origin` header on the handshake but there is no preflight.
- **Native mobile apps.** iOS and Android HTTP clients do not enforce CORS. Do not rely on CORS to secure your API - it is a browser defense against malicious pages, not an authorization mechanism.

That last point is worth repeating. CORS does not authenticate anyone. It protects users from their own browsers being turned against them. Your API still needs real authorization - tokens, signatures, session cookies with `SameSite`, the works. I covered JWT specifically in [JWT deep dive - how tokens work and common mistakes](/blog/jwt-deep-dive-how-tokens-work-and-common-mistakes).

## Debugging checklist

When the browser blocks a request, in order:

1. Open the network tab. Is there an `OPTIONS` request? What headers does the response carry?
2. Is `Access-Control-Allow-Origin` present on the OPTIONS response, and does it match the `Origin` exactly?
3. Is `Access-Control-Allow-Methods` listing the method of the real request?
4. Is every header in `Access-Control-Request-Headers` present in `Access-Control-Allow-Headers`?
5. If credentials are involved, is `Access-Control-Allow-Credentials: true` set, and is `Access-Control-Allow-Origin` a specific origin (not `*`)?
6. Does the real response (not just the OPTIONS) also carry the Allow headers?
7. Is there a proxy in the middle that strips headers? `curl -I -H "Origin: https://app.example.com" https://api.example.com/...` against the public URL.

Nine times out of ten you are missing a header or sending `*` where you need a specific origin. Once you internalize the preflight handshake as "browser asks for permission, server grants or denies with specific headers," CORS stops being a mystery and becomes a protocol you can reason about like any other.
