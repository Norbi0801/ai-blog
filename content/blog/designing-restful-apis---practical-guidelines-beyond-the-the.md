+++
title = "Designing RESTful APIs - practical guidelines beyond the theory"
date = 2025-08-13
description = "Naming, pagination, error formats, versioning, auth, rate limiting - real API design decisions with reasoning, not another 'use nouns not verbs' post."

[taxonomies]
tags = ["api", "rest", "architecture", "web"]
+++

You know the drill. Every REST API guide opens with the same thing: "use nouns, not verbs." Plural resource names. HTTP methods map to CRUD. Great, you've described the first 15 minutes of every developer's REST journey.

The actual hard parts - how do you paginate a dataset that's being written to while someone is reading it? What do your errors look like when three different clients consume your API? How do you version without breaking everyone? - those get a hand-wave at best.

This post is about those decisions. Each section covers a real design choice, with the reasoning behind it and examples from APIs that get millions of requests per day.

<!-- more -->

## Naming conventions that actually matter

The nouns-vs-verbs rule is fine. What nobody tells you is the dozen smaller naming decisions that create consistency or chaos across 50+ endpoints.

**Plurals everywhere, no exceptions.** `/users`, `/orders`, `/invoices`. Not `/user/123` (singular when fetching one). The resource collection is always plural - you're addressing a collection, and `/users/123` means "user 123 from the users collection." This is what GitHub, Stripe, and Twilio all do.

**Nested resources for ownership, not association.** `/users/42/orders` makes sense - orders belong to a user. `/users/42/products` does not - products don't belong to a user, users just interact with them. For associations, use query params: `/products?purchased_by=42`.

**Actions that don't map to CRUD.** Sometimes you need a verb. Sending an email, canceling an order, archiving a project. Two patterns that work:

```
POST /orders/123/cancel
POST /users/42/verify-email
```

Or use a state-change via PATCH:

```json
PATCH /orders/123
{ "status": "canceled" }
```

I prefer the explicit sub-resource approach (`/cancel`) when the action has side effects beyond updating a field - sending a notification, triggering a refund, logging an audit event. The PATCH approach works when you're genuinely just updating state.

**Use kebab-case for multi-word paths.** `/user-profiles`, not `/userProfiles` or `/user_profiles`. URLs are case-insensitive by convention, and kebab-case reads well. Query parameters are a different story - `snake_case` is the dominant convention there: `?sort_by=created_at&per_page=25`.

## Pagination - cursor beats offset, here's why

Offset pagination (`?page=2&per_page=25`) is intuitive. It's also a performance trap and a correctness problem.

**The performance issue.** With `OFFSET 10000, LIMIT 25`, your database reads 10,025 rows and discards 10,000 of them. Benchmarks from real workloads show what happens as you go deeper:

| Page | Offset latency | Cursor latency |
|------|---------------|----------------|
| 1 | ~12ms | ~8ms |
| 50 | ~45ms | ~9ms |
| 1,000 | ~890ms | ~11ms |
| 10,000 | ~8,234ms | ~12ms |

Cursor pagination stays flat because the query becomes `WHERE id > cursor_value ORDER BY id LIMIT 25` - an indexed seek, not a scan-and-discard.

**The correctness issue.** If someone inserts a row while a client is paginating with offsets, every subsequent page shifts. Page 3 might repeat an item from page 2, or skip one entirely. Cursors don't have this problem because they anchor to a specific row, not a position.

**How Stripe does it.** Stripe's pagination is clean and worth copying:

```bash
curl https://api.stripe.com/v1/customers?limit=3
```

```json
{
  "object": "list",
  "url": "/v1/customers",
  "has_more": true,
  "data": [
    {"id": "cus_A1", "name": "Alice"},
    {"id": "cus_A2", "name": "Bob"},
    {"id": "cus_A3", "name": "Carol"}
  ]
}
```

Next page: `?limit=3&starting_after=cus_A3`. Previous page: `?limit=3&ending_before=cus_A1`.

No page numbers, no total counts (which would require a `COUNT(*)` on every request), just a boolean `has_more` and the cursor baked into the last item's ID.

**When offset is acceptable.** Admin dashboards where users need to jump to "page 47" and the dataset is small (under 100K rows). Search results where you want to show "page 3 of 12." In those cases, cap the maximum offset - Elasticsearch defaults to `max_result_window: 10000` for this exact reason.

## Filtering and sorting - steal from the best

Don't invent your own query language. Look at what works in production.

**Flat params for simple filters.** Most common, most readable:

```
GET /products?status=active&category=electronics&in_stock=true
```

**Bracket notation for ranges.** Stripe popularized this and it works well for dates and numeric ranges:

```
GET /charges?created[gte]=1609459200&created[lte]=1640995200&amount[gt]=1000
```

The `[gte]`, `[lte]`, `[gt]`, `[lt]` suffixes map directly to SQL operators. Clients understand them immediately.

**Sort with explicit direction.** Keep it simple:

```
GET /products?sort=price&order=desc
GET /products?sort=created_at&order=asc
```

Or compound sorts with a comma-separated list: `?sort=status,-created_at` where the `-` prefix means descending. GitHub's search API does this well.

**For complex queries, use a separate search endpoint.** Stripe exposes `/v1/charges/search` with its own query syntax:

```
GET /v1/charges/search?query=amount>1000 AND status:'succeeded'
```

This keeps the standard list endpoint simple while giving power users a dedicated tool. Don't bolt a query language onto your collection endpoint.

## Error format - use RFC 9457

If you're returning errors like `{"error": "something went wrong"}`, you're making every client author's life harder. They have to guess the structure, handle inconsistent formats, and parse human-readable strings for machine decisions.

[RFC 9457](https://www.rfc-editor.org/rfc/rfc9457.html) (published July 2023, obsoletes RFC 7807) defines `application/problem+json` - a standardized error format. Use it.

```http
HTTP/1.1 422 Unprocessable Entity
Content-Type: application/problem+json

{
  "type": "https://api.example.com/errors/validation-failed",
  "title": "Validation failed",
  "status": 422,
  "detail": "The 'email' field is not a valid email address.",
  "instance": "/users/signup"
}
```

The five standard fields:

- **`type`** - URI identifying the error type. Make it dereferenceable - point it at your docs page for that error.
- **`title`** - Short, human-readable summary. Same for every occurrence of this error type.
- **`detail`** - Human-readable explanation specific to _this_ occurrence. What exactly went wrong.
- **`status`** - HTTP status code. Yes, it's in the response headers too. Including it in the body means the client doesn't need to thread HTTP metadata through to the error handler.
- **`instance`** - URI identifying this specific occurrence. Useful for support tickets and log correlation.

You can extend it with custom fields. For validation errors, add a `violations` array:

```json
{
  "type": "https://api.example.com/errors/validation-failed",
  "title": "Validation failed",
  "status": 422,
  "detail": "2 fields failed validation.",
  "instance": "/users/signup",
  "violations": [
    {"field": "email", "message": "not a valid email address"},
    {"field": "password", "message": "must be at least 12 characters"}
  ]
}
```

The key benefit: every error from your API has the same shape. Clients write one error handler, not fifteen. A `type` URI that resolves to documentation means your support team can link directly to an explanation.

## Versioning - three approaches, one winner

There are three main approaches. Here's what the biggest APIs actually chose and why.

**URL path versioning:** `/v1/users`, `/v2/users`

Used by Google Cloud (`/compute/v1/`), Twilio (`/2010-04-01/`). Simple to understand, easy to route. The downside: changing versions means changing every URL in every client, and you end up maintaining parallel codebases for `/v1` and `/v2`.

**Header versioning:** `X-GitHub-Api-Version: 2022-11-28`

Used by GitHub. The URL stays the same, the version is in a header. Clients that don't send the header get a default version. Cleaner than URL versioning, but harder to test (you can't just paste a URL into a browser) and easy to forget.

**Date-based pinning:** `Stripe-Version: 2024-09-30.acacia`

Stripe's approach is the most sophisticated. Your account is pinned to the API version active when you first made an API call. Every breaking change is a new dated version. You upgrade by changing one header, and you can test the new version before committing. Breaking changes are documented per-version with a migration guide.

```bash
curl https://api.stripe.com/v1/customers \
  -H "Authorization: Bearer sk_test_..." \
  -H "Stripe-Version: 2025-03-31.basil"
```

**My recommendation:** for most APIs, URL path versioning (`/v1/`) is the pragmatic choice. It's visible, debuggable, and simple. Use it for major breaking changes only. For minor additions (new fields, new optional params), just add them - additive changes don't need a new version. Reserve header-based versioning for Stripe-scale APIs where you need per-customer version pinning and gradual migration.

The worst thing you can do is version too aggressively. If you're on `/v7` after two years, something went wrong in your design process.

## HATEOAS - why you can (mostly) skip it

HATEOAS (Hypermedia as the Engine of Application State) says every response should include links telling the client what it can do next. In theory, this means clients never hardcode URLs - they discover them from the API responses.

In practice, PayPal is the only major API that does this fully:

```json
{
  "id": "5O190127TN364715T",
  "status": "CREATED",
  "links": [
    {"href": "https://api-m.paypal.com/v2/checkout/orders/5O190127TN364715T", "rel": "self", "method": "GET"},
    {"href": "https://api-m.paypal.com/v2/checkout/orders/5O190127TN364715T/capture", "rel": "capture", "method": "POST"}
  ]
}
```

Stripe doesn't do it. GitHub doesn't do it (beyond `Link` headers for pagination). Twilio doesn't do it. The reason: clients hardcode URLs anyway. Nobody writes a generic HATEOAS crawler for their Stripe integration. They read the docs, construct URLs, and move on.

There is one interesting exception. In March 2026, VMware/Tanzu published guidance on using HATEOAS links to guide AI agents through API workflows - the `links` array tells an LLM what actions are available without hardcoding URL patterns. That's a use case that might actually make HATEOAS worth the effort.

For now, skip full HATEOAS. Add a `self` link to each resource (useful for caching and debugging) and use `Link` headers for pagination. That's enough.

## Authentication - Bearer tokens done right

If you covered webhook signature verification in [a previous post](/blog/building-a-webhook-receiver-in-rust-signatures-idempotency-and-async-processing/), authentication is the other side of the coin. Here's what a modern Bearer token setup looks like.

**The header format is simple:**

```http
Authorization: Bearer eyJhbGciOiJSUzI1NiIs...
```

The hard part is everything around it. OAuth 2.1 (consolidating years of security best practices) makes several things mandatory that were previously optional:

| Practice | OAuth 2.0 | OAuth 2.1 |
|----------|-----------|-----------|
| PKCE | Recommended for public clients | Required for all clients |
| Implicit grant | Allowed | Removed |
| Resource Owner Password grant | Allowed | Removed |
| Refresh token rotation | Optional | Required for public clients |

**Token lifetimes.** Keep access tokens short - 15 to 30 minutes. If a token leaks, the blast radius is limited. Use refresh tokens (single-use, rotated on every use) for long-lived sessions.

**Signing.** Use RS256 (asymmetric) over HS256 (symmetric). With RS256, your resource servers verify tokens using the public key but can't forge new ones. A compromised API server can't mint tokens.

**Claims validation.** On every request, validate all of these:
- `iss` (issuer) - is this token from your auth server?
- `aud` (audience) - is this token intended for this service?
- `exp` (expiration) - has it expired?
- `nbf` (not before) - is it valid yet?

Skipping any of these opens you to token confusion attacks, replay attacks, or accepting tokens minted for a completely different service.

**Never put tokens in query strings.** They end up in server logs, browser history, `Referer` headers, and proxy logs. Always use the `Authorization` header.

## Rate limiting - tell clients what's happening

When you rate limit, don't just return `429 Too Many Requests` and leave clients guessing. Tell them the limits and their current usage.

**What everyone uses today (de facto standard):**

```http
HTTP/1.1 200 OK
X-RateLimit-Limit: 5000
X-RateLimit-Remaining: 4987
X-RateLimit-Reset: 1719849600
```

GitHub adds `X-RateLimit-Used` and `X-RateLimit-Resource` for more granularity. These `X-` prefixed headers aren't standardized, but they're so widespread they might as well be.

**The emerging IETF standard** (`draft-ietf-httpapi-ratelimit-headers`, September 2025) defines two new structured headers:

```http
RateLimit-Policy: "burst";q=100;w=60, "daily";q=1000;w=86400
RateLimit: "burst";q=100;r=55;t=38
```

Where `q` = quota limit, `w` = window in seconds, `r` = remaining, `t` = seconds until reset.

This is cleaner - it supports multiple rate limit policies in one header and uses relative seconds instead of Unix timestamps (which avoids clock sync issues between client and server). But no major API uses it yet.

**My recommendation:** use the `X-RateLimit-*` pattern today. It's understood by every HTTP client library and every developer who's ever hit a rate limit. When the IETF draft becomes an RFC, consider migrating. Always include `Retry-After` on `429` responses - it's the one header every client actually checks:

```http
HTTP/1.1 429 Too Many Requests
Retry-After: 30
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 1719849630
Content-Type: application/problem+json

{
  "type": "https://api.example.com/errors/rate-limited",
  "title": "Rate limit exceeded",
  "status": 429,
  "detail": "You've exceeded 100 requests per minute. Try again in 30 seconds."
}
```

Notice that the error body uses RFC 9457 - consistency across all error responses, including rate limits.

## Putting it together

Here's a quick checklist of the decisions covered in this post:

1. **Naming:** plural nouns, kebab-case paths, snake_case query params, explicit sub-resources for actions with side effects.
2. **Pagination:** cursor-based by default. Offset only for small datasets where jump-to-page is needed.
3. **Filtering:** flat params for simple filters, bracket notation for ranges, separate search endpoint for complex queries.
4. **Errors:** RFC 9457 `application/problem+json` for every error. Extend with domain-specific fields.
5. **Versioning:** URL path (`/v1/`) for most APIs. Additive changes don't need a new version.
6. **HATEOAS:** skip it. Add `self` links and pagination `Link` headers.
7. **Auth:** short-lived JWTs with RS256, refresh token rotation, validate all claims.
8. **Rate limiting:** `X-RateLimit-*` headers on every response, `Retry-After` on 429s, RFC 9457 error body.

None of these are novel. They're just the boring, consistent choices that make an API pleasant to integrate with. The best APIs aren't clever - they're predictable. A developer should be able to guess your endpoint structure, error format, and pagination style after reading one page of your docs.
