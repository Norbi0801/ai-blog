+++
title = "JWT deep dive - how tokens work and common mistakes"
date = 2025-02-23
description = "What actually sits between those dots in a JWT, how verification really works, and the ways people keep getting it wrong."

[taxonomies]
tags = ["security", "auth", "rust", "web"]
+++

JWT is one of those technologies that looks simple on the surface and gets misused because it looks simple. Three base64 segments separated by dots, a signature, done. Yet the [OWASP top 10](https://owasp.org/Top10/) keeps listing auth failures year after year, and JWT is usually somewhere in the story.

This post walks through what a JWT actually is at the byte level, why verification is subtler than it looks, which mistakes are the most common, and what a correct implementation looks like in Rust using the [jsonwebtoken](https://crates.io/crates/jsonwebtoken) crate (10.3.0 at the time of writing).

<!-- more -->

## What is in those three segments

A JWT is a string of the form `header.payload.signature`. Each segment is base64url-encoded. Not base64. Base64url. That difference matters because regular base64 uses `+` and `/`, which are not URL-safe, and `=` padding, which JWTs drop. If you ever try to decode a JWT segment with a standard base64 decoder and it complains about padding or invalid characters, that is why.

Take a real token and split it:

```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiJ1c2VyXzEyMyIsImV4cCI6MTc2MzM2MzIwMH0.K7zT5T_QZl...
```

The first segment decodes to JSON:

```json
{"alg":"HS256","typ":"JWT"}
```

The second segment:

```json
{"sub":"user_123","exp":1763363200}
```

The third segment is the raw bytes of an HMAC or RSA signature, base64url-encoded. It is NOT JSON. It is a binary blob. People sometimes assume every segment is JSON and try to parse it. It is not.

The signature is computed over the ASCII bytes of `header_b64 + "." + payload_b64`. Not over the decoded JSON. Not over the payload alone. Over the concrete string representation that the sender shipped. This matters because JSON canonicalization is not defined by the spec. If you re-serialize the payload and sign that, you will produce a different signature even though the logical content is identical.

## Header, payload, signature in more detail

The header declares two things that matter: `alg` (the signing algorithm) and `typ` (always `JWT` for our purposes). Optional fields include `kid` (a key ID used when you rotate keys) and `jku` or `jwk` for embedding keys. Do not trust `jku` or `jwk` from the token. More on that below.

The payload carries claims. The IETF-reserved ones you should actually use:

- `iss` - issuer. Who minted this token. A URL or identifier.
- `sub` - subject. Who the token is about. Usually a user ID.
- `aud` - audience. Who the token is for. Your API identifier.
- `exp` - expiration, a Unix timestamp in seconds.
- `iat` - issued at, also Unix seconds.
- `nbf` - not before. Token is invalid until this time.
- `jti` - JWT ID. A unique identifier for the token, useful for revocation lists.

Everything else is a custom claim. Common ones are `email`, `roles`, `scope`. Keep in mind that the payload is not encrypted. It is signed. Anyone who has the token can decode the payload and read it. If you put secrets there, they are not secret.

## HMAC vs RSA vs EdDSA

HS256 uses HMAC-SHA256. It is symmetric: the same key signs and verifies. That key must be at least 256 bits of entropy. A human-picked password like `my-super-secret-2026` has maybe 40 bits. Brute-forcing weak HMAC keys on JWTs is a well-known attack and there are [tools for it](https://github.com/ticarpi/jwt_tool). Use `openssl rand -base64 32` to generate them.

RS256 uses RSA-PKCS1-SHA256. It is asymmetric. You sign with a private key and verify with a public key. The public key can be distributed freely, often over a JWKS endpoint like `/.well-known/jwks.json`. This is what lets you have one auth server and many independent resource servers.

ES256 uses ECDSA over the NIST P-256 curve. Smaller signatures than RSA, similar security.

EdDSA (Ed25519) is newer, has been in the [RFC 8037](https://www.rfc-editor.org/rfc/rfc8037) JOSE extension for years, and is generally preferred over ECDSA when the library supports it. Smaller keys, no nonce reuse footguns, constant time by design.

Rule of thumb: if one process both issues and consumes tokens, HS256 is fine. If the issuer and consumer are separate services, use RS256 or EdDSA so the consumer does not need the signing key.

## The verification flow

This is where most bugs live. A correct verification does all of the following:

1. Split the token into three segments. Reject if you do not get exactly three.
2. Base64url-decode the header. Parse it as JSON.
3. Check the `alg` field against a fixed allowlist your server accepts. NOT against whatever the token says it wants.
4. Load the key for this `alg`. If `kid` is present, use it to pick the right key from your known set. Do not fetch keys from URLs embedded in the token itself.
5. Compute the signature over `header_b64 + "." + payload_b64` using the selected algorithm and key.
6. Compare against the decoded signature using a constant-time comparison.
7. Decode the payload as JSON.
8. Check `exp` against the current time, allowing a small clock skew (30 to 60 seconds is typical).
9. Check `nbf` similarly.
10. Check `iss` and `aud` match what you expect.

Step 3 is the one that has caused the most real-world breaches. There is a family of attacks where a server blindly honors the `alg` field. If a library accepts `alg: none` and returns the payload as valid, anyone can forge any claims. If a server expects RS256 but also accepts HS256, an attacker can sign a token using the public RSA key as an HMAC secret, and the server happily verifies it because the public key is usually not secret. This is the [classic alg confusion attack](https://auth0.com/blog/critical-vulnerabilities-in-json-web-token-libraries/) from 2015, still showing up in audits today.

## The mistakes that keep happening

**Not checking exp.** Surprisingly common. Someone writes a helper that decodes the payload and returns the claims, and never validates the signature or expiration. Any code path that calls `decode_without_verification` and then uses the result is broken. Treat such functions as payload inspectors for debugging only.

**Storing tokens in localStorage.** If you put the token in localStorage, any XSS on your site gives an attacker full account takeover. JavaScript can read localStorage directly. HttpOnly cookies cannot be read by JavaScript. Yes, cookies have CSRF concerns, but SameSite=Lax or Strict plus CSRF tokens handle that. The comparison is not "cookies vs localStorage, which is more secure" - it is "you need some defense against XSS, and localStorage has none."

**No revocation.** This is JWT's big trade-off. Since verification is offline, you cannot take a token back once issued. If a user logs out, resets a password, or gets their account banned, their still-valid token keeps working until `exp`. The mitigations: short expirations (5 to 15 minutes), a refresh token flow, and optionally a denylist of `jti` values for emergency revocation. If you find yourself maintaining a big denylist of active tokens, you probably should have used sessions.

**Symmetric key sharing.** If three services need to verify tokens and you use HS256, all three need the signing key. Any one of them leaking it compromises the whole system. This is exactly the situation RS256 solves.

**Long-lived tokens with everything in them.** Teams put the full user profile, roles, permissions, and feature flags in the token because it saves a database lookup. Then the token is 3 KB, hits cookie size limits, and the user's role cannot change until they get a new token. Keep payloads small. Claims that change should not be in the token.

## Refresh tokens

The refresh token pattern compromises between JWT's offline verification and the need for revocation. Access tokens are short-lived JWTs. Refresh tokens are long-lived opaque strings stored in a database. To get a new access token, the client sends the refresh token to the auth server, which checks the database and issues a new JWT.

Rotation is important here. Each use of a refresh token invalidates the old one and returns a new refresh token. If the same refresh token is ever reused (meaning it was stolen and replayed), invalidate the whole family for that user. This is the OAuth 2.0 refresh token rotation pattern described in [RFC 6819](https://www.rfc-editor.org/rfc/rfc6819) and the [Security Best Current Practice draft](https://datatracker.ietf.org/doc/html/draft-ietf-oauth-security-topics).

## Token size

A minimal HS256 JWT with `sub` and `exp` is around 150 bytes. Add a handful of claims and you quickly reach 500 bytes. Put a full user object in there and it becomes 2 KB or more. Cookies are capped at 4 KB per cookie in most browsers. HTTP/2 HPACK helps with header size, but every request still carries the full token. At 2 KB per request and 100 requests per page load, you are moving 200 KB of auth data per page. That adds up.

If you care about this, look at your payload and remove anything that is not strictly needed for authorization. Roles and a user ID are usually enough.

## When JWT, when sessions

Sessions with an opaque session ID and a server-side store:

- Native revocation (delete the session row)
- Small cookie (32 bytes of session ID)
- Mutable state (update roles, reflect in next request)
- Requires a session store on every verification

JWT:

- Offline verification, no database hit
- Horizontally scales trivially
- Works across services without a shared store
- No built-in revocation
- Larger payload on every request

If you have one monolith and one database, sessions are simpler and usually better. If you have a fleet of microservices in different processes, or a mobile app calling an API, JWT (or its OAuth 2.0 cousin, opaque access tokens with introspection) is the fit. The worst case is using JWT for everything because "it scales," in a monolith that does not need it, then ending up implementing denylists and server-side state anyway.

## Implementation in Rust with jsonwebtoken

Add the dependency. The crate now requires picking a crypto backend explicitly:

```toml
[dependencies]
jsonwebtoken = { version = "10.3", features = ["aws_lc_rs"] }
serde = { version = "1", features = ["derive"] }
```

Define your claims:

```rust
use serde::{Deserialize, Serialize};

#[derive(Debug, Serialize, Deserialize)]
struct Claims {
    sub: String,
    iss: String,
    aud: String,
    exp: u64,
    iat: u64,
    #[serde(default)]
    roles: Vec<String>,
}
```

Issue a token:

```rust
use jsonwebtoken::{encode, EncodingKey, Header, Algorithm};
use std::time::{SystemTime, UNIX_EPOCH};

fn issue(user_id: &str, secret: &[u8]) -> Result<String, jsonwebtoken::errors::Error> {
    let now = SystemTime::now().duration_since(UNIX_EPOCH).unwrap().as_secs();
    let claims = Claims {
        sub: user_id.to_string(),
        iss: "https://auth.example.com".to_string(),
        aud: "https://api.example.com".to_string(),
        iat: now,
        exp: now + 15 * 60, // 15 minutes
        roles: vec!["user".to_string()],
    };

    let header = Header::new(Algorithm::HS256);
    encode(&header, &claims, &EncodingKey::from_secret(secret))
}
```

Verify a token. Note the explicit algorithm allowlist and the audience and issuer checks:

```rust
use jsonwebtoken::{decode, DecodingKey, Validation, Algorithm};

fn verify(token: &str, secret: &[u8]) -> Result<Claims, jsonwebtoken::errors::Error> {
    let mut validation = Validation::new(Algorithm::HS256);
    validation.set_audience(&["https://api.example.com"]);
    validation.set_issuer(&["https://auth.example.com"]);
    validation.set_required_spec_claims(&["exp", "iss", "aud", "sub"]);
    validation.leeway = 30; // seconds of clock skew allowed

    let data = decode::<Claims>(token, &DecodingKey::from_secret(secret), &validation)?;
    Ok(data.claims)
}
```

A few things the crate gets right that your hand-rolled version probably would not: it requires you to pass an `Algorithm` to `Validation::new`, so you cannot accidentally accept whatever the token's header claims. It validates `exp` and `nbf` automatically. It will not decode `alg: none` because that algorithm is gated behind the `Algorithm::None` variant you would have to opt into explicitly.

For RS256 with a JWKS endpoint, fetch the keys, pick by `kid`, and build `DecodingKey::from_rsa_components` or `from_rsa_pem`:

```rust
use jsonwebtoken::{decode_header, DecodingKey};

fn verify_rs256(token: &str, jwks: &MyJwks) -> Result<Claims, Box<dyn std::error::Error>> {
    let header = decode_header(token)?;
    let kid = header.kid.ok_or("missing kid")?;
    let jwk = jwks.get(&kid).ok_or("unknown kid")?;

    let key = DecodingKey::from_rsa_components(&jwk.n, &jwk.e)?;
    let mut validation = Validation::new(Algorithm::RS256);
    validation.set_audience(&["https://api.example.com"]);
    validation.set_issuer(&["https://auth.example.com"]);

    let data = decode::<Claims>(token, &key, &validation)?;
    Ok(data.claims)
}
```

Cache the JWKS response, respect cache headers, and refresh on unknown `kid` so key rotation works without restart.

## The short version

Verify the signature every time. Pin the algorithm. Check `exp`, `iss`, `aud`. Put tokens in HttpOnly cookies. Keep access tokens short-lived and pair them with refresh tokens if you need revocation. Use RS256 or EdDSA when multiple services are involved. If you are running a monolith with a database, sessions are probably fine and you do not need any of this.

JWT is not magic and it is not inherently insecure. It is a narrow tool with a few sharp edges, and almost every real-world incident comes from skipping one of the checks above.

Sources:
- [jsonwebtoken crate on crates.io](https://crates.io/crates/jsonwebtoken)
- [jsonwebtoken docs](https://docs.rs/jsonwebtoken/latest/jsonwebtoken/)
- [Critical vulnerabilities in JWT libraries (Auth0)](https://auth0.com/blog/critical-vulnerabilities-in-json-web-token-libraries/)
- [OAuth 2.0 Security Best Current Practice](https://datatracker.ietf.org/doc/html/draft-ietf-oauth-security-topics)
- [RFC 7519 - JSON Web Token](https://www.rfc-editor.org/rfc/rfc7519)
- [RFC 8037 - EdDSA for JOSE](https://www.rfc-editor.org/rfc/rfc8037)
