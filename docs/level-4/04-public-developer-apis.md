---
description: "Designing Public Developer-Facing APIs — Building an API for your own frontend is one job. Building one that thousands of third-party developers integrate…"
---

# 04 · Designing Public Developer-Facing APIs

Building an API for your own frontend is one job. Building one that
thousands of third-party developers integrate against, without ever
talking to your team, is a different discipline: stability,
discoverability, and self-service are the product.

## API keys and developer accounts

```bash
curl https://api.example.com/v1/orders \
  -H "Authorization: Bearer sk_live_51H8x..."
```

Unlike an internal OAuth2 token tied to a logged-in user session, a
public API typically issues long-lived **API keys** per developer
account, scoped and revocable independently:

```bash
curl -X POST https://api.example.com/v1/api-keys \
  -H "Authorization: Bearer $DASHBOARD_SESSION_TOKEN" \
  -d '{"name": "production-server", "scopes": ["orders:read"]}'
```

```json
{ "id": "key_9", "key": "sk_live_51H8x...", "scopes": ["orders:read"], "created_at": "2026-08-31T00:00:00Z" }
```

The raw key is shown exactly once at creation and never again — only
its prefix and metadata are retrievable afterward, so a leaked key can
be identified and revoked without the API ever storing the plaintext.

## Sandbox vs. live environments

```bash
curl https://api.example.com/v1/orders -H "Authorization: Bearer sk_test_..."
curl https://api.example.com/v1/orders -H "Authorization: Bearer sk_live_..."
```

A `sk_test_` key routes to a fully isolated sandbox — fake payment
processing, no real charges, resettable data — so a developer building
an integration can experiment freely and safely before ever touching
production. Key *prefix* signals environment, catching the common
mistake of a test key accidentally shipped to production (which simply
fails auth against live) instead of silently doing something dangerous.

## SDKs generated from the OpenAPI spec

```python
import example_api
client = example_api.Client(api_key="sk_live_...")
order = client.orders.create(item_id=5, qty=2)
```

Auto-generating official SDKs in major languages from the same OpenAPI
spec that drives the reference docs (Level 3, module 7) keeps every
language binding in sync automatically when the spec changes — instead
of hand-maintained SDKs drifting out of date.

## Predictable, generous rate limits — and telling developers about them

```http
HTTP/1.1 200 OK
X-RateLimit-Limit: 1000
X-RateLimit-Remaining: 998
X-RateLimit-Reset: 1735689600
```

Third-party developers build automated systems against these headers —
unlike an internal team you can just tell "don't hammer that endpoint,"
a public integrator needs the limit itself, machine-readable, to build
correct backoff logic.

## API stability as a product commitment

Public APIs need a much longer deprecation runway than internal ones
(Level 3, module 6) — a third-party developer might not read your
changelog for months. A common commitment:

```markdown
## Stability policy
- v1 will be supported for a minimum of 24 months after v2 ships.
- Breaking changes are only ever shipped in a new major version, never
  as a patch to an existing one.
- New optional fields may be added to v1 responses at any time.
```

## Worked example: a developer's first hour with your API

1. They sign up, land on a dashboard, and generate a sandbox key with
   zero human involvement.
2. The quickstart guide (Level 3, module 7) gets them to one successful
   `curl` call in under 5 minutes.
3. They install the official SDK (`pip install example-api`) and build
   a small integration against sandbox data.
4. They hit a rate limit while testing in a loop, read
   `X-RateLimit-Reset` in the response, and back off correctly without
   filing a support ticket.
5. They flip to a live key, ship to production, confident nothing will
   silently break because of the published stability policy.

Every step above is self-service — no human on your team was involved,
which is the entire point of a public developer API's design.

## How It Actually Works

Running a public developer API means handling identity and quota
enforcement for callers you don't control, at every single request —
mechanically different from an internal API where you trust the caller.

**API key issuance and lookup**: a key like `sk_live_a1b2c3...` is
typically not stored in your database in plaintext — it's hashed (like a
password) on issuance, and every incoming request re-hashes the presented
key and looks up the hash, so a database leak doesn't hand attackers
usable keys directly (mirroring password storage practice, module 7,
Level 1 auth basics extended). The key's prefix (`sk_live_` vs `sk_test_`)
is often left unhashed specifically so it can be pattern-matched and
revoked in bulk, or displayed truncated in a dashboard, without exposing
the secret portion.

**Per-key rate limiting** reuses the token-bucket mechanism (module 6,
Level 2) but keyed by API key rather than IP or user session — this is
why a public API's `429` responses typically include `X-RateLimit-Limit`,
`X-RateLimit-Remaining`, and `X-RateLimit-Reset` headers computed directly
off that same bucket's current token count and refill schedule, giving
well-behaved SDKs enough information to self-throttle before hitting the
wall.

**SDK generation** from your OpenAPI spec (module 4, Level 2) means a
client library in Python, JS, or Go is mechanically produced by walking
the same schema tree a doc generator uses — a breaking schema change
doesn't just break hand-written client code, it breaks the *next SDK
build* for every language you publish, which is why public APIs are held
to stricter backward-compatibility discipline (module 6, Level 3) than
internal ones: SDK consumers can't just "read the updated docs and adapt"
as fluidly as an internal team sitting next to yours.

## Exercise

1. Why show a raw API key only once at creation, rather than letting a
   developer view it again later from the dashboard?
2. Explain why a sandbox environment needs to be genuinely isolated
   (separate data, no real side effects) rather than just a flag on
   production data.
3. Design the `scopes` a public payments API might offer beyond
   `orders:read` — think about what a developer building a read-only
   analytics dashboard should be limited to versus one processing
   refunds.
4. Why does a public API need a longer deprecation runway than an
   internal one used only by your own frontend team?
