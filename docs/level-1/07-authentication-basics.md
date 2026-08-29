# 07 · Authentication Basics (API Keys & Bearer Tokens)

Authentication answers "who is calling?" Authorization (a separate concern,
touched on here and expanded in Level 3's OAuth2 module) answers "what are
they allowed to do?" This module covers the two schemes you'll meet in
almost every API before you get to full OAuth2.

## API keys

An API key is a single, long, random string identifying the calling
application (not necessarily an individual user). It's the simplest scheme
and common for server-to-server integrations, public data APIs, and
third-party developer platforms.

### Sending an API key

Three common conventions — check the specific API's docs for which it uses:

```bash
# 1. Custom header (most common, most explicit)
curl https://api.example.com/data \
  -H "X-API-Key: sk_live_4f8a9c2e1b7d3f6a"

# 2. Authorization header with a custom scheme
curl https://api.example.com/data \
  -H "Authorization: ApiKey sk_live_4f8a9c2e1b7d3f6a"

# 3. Query parameter (least secure — avoid if you control the design)
curl "https://api.example.com/data?api_key=sk_live_4f8a9c2e1b7d3f6a"
```

**Avoid query-parameter API keys when you're designing the API**: URLs get
logged by proxies, browser history, and server access logs by default,
which leaks the secret into places it shouldn't be. A header doesn't have
this problem since headers aren't typically logged by default the same way.

### What an API key does and doesn't give you

- It identifies *which application/account* is calling — useful for billing,
  rate limiting per customer, and revocation.
- It usually does **not** identify an individual end-user within that
  account, and it carries no built-in expiration or scoping — a leaked key
  is valid until manually revoked.
- Because of that, API keys are best suited to trusted server-to-server
  calls, not to public-facing apps (e.g. never embed a secret API key in
  client-side JavaScript or a mobile app binary where anyone can extract it).

## Bearer tokens

A Bearer token is a credential sent as `Authorization: Bearer <token>` where
the token itself proves who you are — "bearer" meaning whoever *holds* it
can use it, no additional proof needed (which is exactly why it must be
transmitted only over HTTPS and stored carefully).

```bash
curl https://api.example.com/me \
  -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiI0MiIsIm5hbWUiOiJBZGEgTG92ZWxhY2UiLCJleHAiOjE3MjQ5NjAwMDB9.4x7f2h..."
```

### JWTs — the most common bearer token format

Many APIs issue **JWTs** (JSON Web Tokens) as bearer tokens. A JWT has three
base64url-encoded parts separated by dots: `header.payload.signature`.

```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiI0MiIsIm5hbWUiOiJBZGEgTG92ZWxhY2UiLCJleHAiOjE3MjQ5NjAwMDB9.4x7f2h...
└──────── header ────────┘ └─────────────────── payload ───────────────────┘ └─ signature ─┘
```

Decoding the payload (base64url-decode the middle segment) reveals claims
about the token:

```json
{
  "sub": "42",
  "name": "Ada Lovelace",
  "exp": 1724960000
}
```

- `sub` (subject) — who this token represents (a user ID here).
- `exp` (expiration) — a Unix timestamp after which the token is invalid.
- The **signature** is what makes it trustworthy: it's a cryptographic hash
  of the header + payload, signed with a secret (or private key) only the
  server knows. Anyone can *read* a JWT's payload (it's just base64, not
  encrypted) but only the server can produce a *valid* signature — so a
  client can't forge or tamper with claims without invalidating the
  signature.

!!! warning "JWTs are readable, not secret"
    Never put sensitive data (passwords, secrets, PII you wouldn't want
    exposed) in a JWT payload — anyone holding the token can decode and read
    it, even though they can't forge a new valid one.

### A typical login-then-authenticated-request flow

```bash
# 1. Log in, receive a bearer token
curl -i -X POST https://api.example.com/login \
  -H "Content-Type: application/json" \
  -d '{"email": "ada@example.com", "password": "correct horse battery staple"}'
```

```http
HTTP/1.1 200 OK
Content-Type: application/json

{
  "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiI0MiJ9.4x7f2h...",
  "expires_in": 3600
}
```

```bash
# 2. Use the token on subsequent requests
curl https://api.example.com/orders \
  -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiI0MiJ9.4x7f2h..."
```

```http
HTTP/1.1 401 Unauthorized
Content-Type: application/json

{ "error": "token_expired", "message": "Your session has expired. Please log in again." }
```

That second response shows what happens once `exp` has passed — a correctly
implemented API rejects it with `401`, and the client is expected to log in
again (or use a refresh token, a pattern covered fully with OAuth2 in
Level 3).

## API keys vs. bearer tokens — when to use which

| | API key | Bearer token (e.g. JWT) |
|---|---|---|
| Identifies | An application/account | Typically an individual user session |
| Expiration | Usually none (until revoked) | Usually short-lived (`exp` claim) |
| Typical source | Issued once in a developer dashboard | Issued per login/session |
| Best for | Server-to-server, third-party integrations | User-facing apps, web/mobile sessions |
| Revocation | Manual (regenerate in dashboard) | Often automatic via expiration + refresh flow |

## Basic Auth — worth recognizing, rarely designing new

```bash
curl -u ada:correcthorsebatterystaple https://api.example.com/orders
```

This sends `Authorization: Basic <base64(username:password)>`. It's simple
but sends credentials on every single request and has no built-in
expiration — fine for quick internal tooling or protecting a staging
environment, but you should not design a new production API around it in
2026; prefer bearer tokens or API keys.

## Exercise

1. Given this JWT payload — `{"sub": "17", "role": "admin", "exp":
   1735689600}` — explain what a server must check before trusting a request
   carrying this token, beyond just verifying the signature.
2. Write the `curl` command to call `GET https://api.example.com/admin/users`
   using a bearer token stored in a shell variable `$TOKEN`.
3. You're designing a public API for third-party developers to integrate
   with your platform server-to-server (no end users involved on their
   side). Would you choose API keys or bearer tokens as the primary scheme,
   and why?
4. Explain, in your own words, why sending an API key as a query parameter
   (`?api_key=...`) is riskier than sending it as a header.
