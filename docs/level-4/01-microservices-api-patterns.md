# 01 · Microservices API Design Patterns

Once an API is backed by many independently-deployed services rather
than one monolith, new design questions appear: who owns which data,
how do services talk to each other, and how does a client-facing
request survive one of those services being slow or down.

## Database-per-service

Each microservice owns its own database; no service reaches directly
into another's tables.

```
orders-service   → orders_db     (orders, order_items)
users-service    → users_db      (users, addresses)
inventory-service → inventory_db (stock_levels)
```

`orders-service` never runs `SELECT * FROM users`. If it needs a user's
name, it calls `users-service`'s API — exactly like an external client
would. This is what makes services independently deployable: no one can
break another team's queries by changing a table they don't own.

## The API composition pattern

When a client-facing endpoint needs data owned by multiple services,
something has to compose it. Usually an API gateway or backend-for-
frontend layer (module 3, Level 3) does this:

```bash
curl https://api.example.com/v1/orders/42/summary \
  -H "Authorization: Bearer $TOKEN"
```

```json
{ "order": { "id": 42, "total": 89.99 }, "customer": { "name": "Ada" }, "item_names": ["Widget", "Gadget"] }
```

Internally this is three calls (`orders-service`, `users-service`,
`inventory-service`) merged by the composing layer, not by the client.

## Backend-for-Frontend (BFF)

Different clients need different shapes from the same underlying
services. A mobile app wants a lean payload; a web dashboard wants
everything.

```
Mobile BFF  → GET /mobile/v1/orders/42   → { id, total, status }
Web BFF     → GET /web/v1/orders/42      → { id, total, status, items[], customer{}, history[] }
```

Each BFF is a thin, client-specific layer over the same core services —
it avoids either bloating one shared endpoint with every possible field,
or forcing every client to do its own composition.

## Handling failure: a downstream service is down

```json
{
  "order": { "id": 42, "total": 89.99 },
  "customer": null,
  "_warnings": ["customer service unavailable, showing partial data"]
}
```

A **circuit breaker** in the composing layer detects `users-service`
failing repeatedly, stops calling it for a cooldown period (failing
fast instead of piling up slow timeouts), and the composed response
degrades gracefully rather than the whole `/orders/42/summary` call
failing just because one non-critical field is unavailable.

## Saga pattern: distributed transactions without a shared database

Placing an order touches three services: reserve inventory, charge
payment, create the order record. There's no single database
transaction spanning all three. A **saga** runs it as a sequence of
local transactions with compensating actions on failure:

```
1. inventory-service: reserve 2 units          → succeeds
2. payments-service: charge $89.99             → FAILS (card declined)
3. compensate: inventory-service releases the 2 units reserved in step 1
```

```bash
curl -X POST https://internal.example.com/inventory/reserve -d '{"item_id": 5, "qty": 2}'
curl -X POST https://internal.example.com/payments/charge -d '{"amount": 89.99}'
# → 402 Payment Required
curl -X POST https://internal.example.com/inventory/release -d '{"item_id": 5, "qty": 2}'
```

No global lock, no two-phase commit across services — just a defined
compensating action for every step, run in reverse on failure.

## Worked example: designing the order-placement flow

For an e-commerce platform split into `orders`, `inventory`, and
`payments` services:

1. Client calls `POST /v1/orders` on the gateway.
2. `orders-service` orchestrates the saga: reserve inventory → charge
   payment → mark order confirmed.
3. If any step fails, run compensations for every step that already
   succeeded, and return the client a clear error (`402
   payment_failed`, not a generic `500`).
4. On success, `orders-service` publishes an `order.confirmed` event
   (module 7) that `inventory-service` and `notifications-service` react
   to independently, rather than `orders-service` calling them directly
   and coupling to their availability.

## Exercise

1. Why does database-per-service make independent deployability
   possible in a way that a shared database across services doesn't?
2. Design the compensating action for a saga step that sends a
   confirmation email — is a compensation even needed here, and why or
   why not?
3. A circuit breaker trips on `users-service`. What should the
   composing layer return to the client, and what HTTP status code is
   appropriate for a partially-degraded response?
4. Compare API composition (gateway calls multiple services
   synchronously) with an event-driven approach (module 7) for keeping
   an `orders` read-model in sync with `inventory` changes — what's the
   trade-off?
