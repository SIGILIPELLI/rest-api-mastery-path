# 10 · Capstone Project

Design and describe a production-grade API platform end to end,
bringing together every module from all four levels. This is a design
exercise: work through each decision below the way you would in a real
architecture review, and be able to justify it.

## The brief

You're building **OrderFlow**, a public API letting third-party
e-commerce platforms manage orders, inventory, and payments through
your service, at a scale of tens of thousands of requests per second
across multiple continents, monetized as a paid product.

## 1. Resource design & Level 1 foundations

```bash
curl https://api.orderflow.example.com/v1/orders/42 \
  -H "Authorization: Bearer sk_live_..."
```

```json
{ "id": 42, "status": "confirmed", "total_amount": 89.99, "links": { "self": "/v1/orders/42", "items": "/v1/orders/42/items" } }
```

Resources: `orders`, `items`, `inventory`, `refunds` — nouns, proper
status codes, consistent error envelope from day one.

## 2. Level 2: pagination, caching, idempotency, rate limiting

```bash
curl "https://api.orderflow.example.com/v1/orders?limit=50&cursor=..." \
  -H "Authorization: Bearer $KEY" \
  -H "Idempotency-Key: create-order-8f2e"
```

Cursor pagination for the ever-growing orders collection, `ETag`
caching on read-heavy `GET /v1/inventory/{sku}`, idempotency keys on
every `POST`, and per-account rate limits enforced at the gateway.

## 3. Level 3: OAuth2, webhooks, gateway, docs, testing, monitoring

- OAuth2 client-credentials flow for server-to-server partner
  integrations; Authorization Code + PKCE for any OrderFlow-hosted
  dashboard.
- Webhooks (`order.confirmed`, `refund.issued`) signed and retried with
  backoff, so partners don't have to poll.
- A gateway terminating TLS, validating tokens once, and routing to
  `orders-service`, `inventory-service`, `payments-service`.
- OpenAPI-generated reference docs, a quickstart, and a changelog with
  `Deprecation`/`Sunset` headers ready for the inevitable `v2`.
- A test pyramid: unit tests for pricing logic, integration tests
  against a staging database, contract tests validating every response
  against the OpenAPI spec in CI, and a handful of end-to-end checkout
  journeys.
- Structured logs with `request_id`, RED metrics (Rate/Errors/
  Duration) per endpoint, and distributed tracing across the three
  services.

## 4. Level 4: architecture at scale

**Microservices & sagas** — placing an order runs as a saga across
`inventory-service` (reserve), `payments-service` (charge), and
`orders-service` (confirm), with compensating actions if any step
fails, and an outbox-pattern event (`order.confirmed`) published only
after the local transaction commits.

**Security hardening** — every resource lookup filters by the
authenticated partner's `account_id`, closing BOLA; mass-assignment is
prevented by explicit per-endpoint field allowlists; CORS is locked to
an explicit allowlist for any browser-facing dashboard.

**Performance** — Redis caches hot inventory reads with write-time
invalidation; slow report generation (`GET /v1/reports/monthly`) runs
async, returning `202` with a job ID; load testing establishes a
verified capacity (e.g. p99 < 300ms at 5,000 req/s) before launch.

**Multi-region** — active-active in `us-east-1` and `eu-central-1`,
GeoDNS routing reads to the nearest healthy region, writes routed to a
per-account home region to avoid cross-region write latency, with
documented RPO/RTO for regional failover.

**Governance** — a style guide (cursor pagination, integer-cents money
fields, the standard error envelope) enforced by CI lint on every new
endpoint any team adds, plus design-first spec review for anything
introducing a new pattern.

**Event-driven** — `order.confirmed` and `refund.issued` flow through
Kafka to `analytics-service` and `notifications-service`, each
consuming idempotently and independently, decoupled from
`orders-service`'s own availability.

**Monetization** — a free tier (1,000 calls/mo) and paid tiers with
overage billing, usage metered asynchronously via the same event
stream, a live usage dashboard, and `429 quota_exceeded` responses that
clearly point at the upgrade path.

**Zero-downtime evolution** — any schema change follows expand-migrate-
contract; deploys are rolling with health-checked instances; the
platform has shipped and fully sunset a `v1` field rename without a
single customer-facing incident.

## Deliverable

Produce, for OrderFlow:

1. An OpenAPI spec covering `orders`, `inventory`, `refunds`, and
   `webhooks`.
2. A one-page architecture diagram showing the gateway, the three
   services, the message broker, and the multi-region layout.
3. A written justification (a few paragraphs) for three of your
   biggest design decisions — e.g. why sagas over distributed
   transactions, why active-active over active-passive, why cursor
   over offset pagination — referencing the specific trade-offs covered
   in the relevant module.

## How It Actually Works

A production-grade API pulls every mechanism from this course together
into one request path, and tracing a single `POST /orders` through a
hardened, multi-region, event-driven system makes the layering concrete:

1. **DNS** resolves to the nearest healthy region (module 6).
2. **Gateway** terminates TLS, checks the rate-limit bucket for this API
   key (module 6, Level 2 + module 4, Level 4), verifies the JWT's
   signature and pinned algorithm (module 2), and forwards internally with
   the verified identity attached.
3. **Application server** validates the body against its OpenAPI schema
   (module 4, Level 2), begins a DB transaction on the primary (never a
   replica, module 3) to avoid replication-lag write conflicts, and
   generates an idempotency key check (module 5, Level 2) before
   committing the order row.
4. **Event emission**: on commit, an `order.created` event is published to
   a partitioned topic keyed by `order_id` (module 7), guaranteeing
   ordering for that order's subsequent events without blocking the HTTP
   response on downstream consumers.
5. **Usage metering**: a separate consumer of that same event increments
   the customer's billing usage, deduplicated by request ID (module 8).
6. **Response**: the gateway streams the `201 Created` back through the
   same reused TLS connection, with `Cache-Control: no-store` (orders
   aren't cacheable) and a `Location` header pointing at the new resource.

Every "module" in that list is a real function call or middleware stage
executing in a specific order on this one request — building the capstone
means implementing enough of that real pipeline to see how a change at
one layer (a slow gateway auth check, an unindexed idempotency lookup)
becomes a measurable regression in the whole chain, not an abstract
diagram.

## Exercise

1. Walk through what happens, end to end, when a partner's `POST
   /v1/orders` call succeeds: every module from this capstone that
   touches that single request, in order.
2. Now walk through what happens when the same call's payment step
   fails midway — the saga compensation, the client-facing error, and
   what (if anything) gets published as an event.
3. A partner reports intermittent `404`s on an order they just created
   moments earlier, from a different API call. Which Level 4 concept
   explains this, and what's the fix?
4. Six months post-launch, you need to rename `total_amount` again to
   `amount_total` after a company-wide naming standard changes. Lay out
   the full multi-deploy plan, referencing the specific pattern that
   makes it safe.
