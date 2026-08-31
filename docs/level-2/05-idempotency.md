# 05 · Idempotency

An operation is **idempotent** if performing it multiple times has the same
effect as performing it once. This matters enormously for APIs because
networks are unreliable: a client might time out waiting for a response,
retry the request, and now the server may have received the same "logical"
request twice. Idempotency is what makes retries safe.

## Which HTTP methods are idempotent by definition

| Method | Idempotent? | Why |
|---|---|---|
| `GET` | Yes | Read-only, no state change |
| `PUT` | Yes | Replaces a resource with a given representation — doing it twice leaves the same end state |
| `DELETE` | Yes | Deleting an already-deleted resource still ends with it gone |
| `HEAD`, `OPTIONS` | Yes | No state change |
| `PATCH` | Not guaranteed | Depends on the patch semantics (see below) |
| `POST` | **No** | Typically creates a new resource — calling it twice creates two |

```bash
# PUT is naturally idempotent: replacing with the same body twice = same result
curl -X PUT https://api.example.com/books/42 -d '{"title": "Dune", "price": 15}'
curl -X PUT https://api.example.com/books/42 -d '{"title": "Dune", "price": 15}'
# Both calls leave book 42 with title "Dune", price 15. No difference.
```

```bash
# POST is not: retried, it creates two orders instead of one
curl -X POST https://api.example.com/orders -d '{"item": "book-42", "qty": 1}'
curl -X POST https://api.example.com/orders -d '{"item": "book-42", "qty": 1}'
# If the first request actually succeeded but the response was lost,
# the client now has two orders instead of the one it intended.
```

`PATCH` depends on what the patch describes: `{"price": 15}` is idempotent
(setting an absolute value), but `{"price_delta": -5}` ("decrease price by
5") is not — applying it twice decreases the price by 10.

## The idempotency-key pattern for POST

Since `POST` is inherently unsafe to retry, APIs that need retry-safe
creation (payments, orders — anything where a duplicate is costly) adopt an
**idempotency key**: the client generates a unique token per logical
operation and sends it with the request. The server remembers keys it has
already processed and returns the original result instead of repeating the
side effect.

```bash
curl -X POST https://api.example.com/payments \
  -H "Idempotency-Key: 8f14e45f-ceea-4b16-8ad3-1b3fda8b6c93" \
  -H "Content-Type: application/json" \
  -d '{"amount": 5000, "currency": "usd", "customer": "cus_123"}'
```

First call: the server processes the payment, stores the result keyed by
`8f14e45f-...`, and returns `201 Created`.

```bash
curl -X POST https://api.example.com/payments \
  -H "Idempotency-Key: 8f14e45f-ceea-4b16-8ad3-1b3fda8b6c93" \
  -H "Content-Type: application/json" \
  -d '{"amount": 5000, "currency": "usd", "customer": "cus_123"}'
```

Second call (a retry after a timeout, same key): the server recognizes the
key, does **not** charge the customer again, and returns the *original*
`201` response verbatim.

This is exactly how Stripe's `Idempotency-Key` header works in production —
it's the canonical real-world example of this pattern.

## Implementing it server-side

```python
def create_payment(request):
    key = request.headers.get("Idempotency-Key")
    if key:
        cached = idempotency_store.get(key)
        if cached:
            return cached.response, cached.status_code   # replay, no new charge

    result = charge_customer(request.body)
    response, status = serialize(result), 201

    if key:
        idempotency_store.set(key, response, status, ttl=24 * 3600)

    return response, status
```

A few important details:

- **Scope the key to the request body too** — if a client reuses a key with
  a *different* body, that's a client bug; return `422 Unprocessable
  Entity` rather than silently replaying the wrong result.
- **Expire keys** after a reasonable window (Stripe uses 24 hours) — storing
  them forever is unnecessary and costly.
- **Store the key atomically with the side effect**, ideally in the same
  database transaction as the payment/order row, so a crash between
  "charge card" and "record key" can't cause a duplicate charge on retry.

```json
// Client reuses key "8f14e45f-..." but changes the amount — server must reject, not replay
{
  "error": {
    "code": "idempotency_key_conflict",
    "message": "Idempotency-Key 8f14e45f-... was already used with a different request body."
  }
}
```

## Idempotency vs safety

Don't confuse the two: a **safe** method (`GET`, `HEAD`) has no side
effects at all. An **idempotent** method may have side effects, but
repeating it doesn't compound them. `PUT` is idempotent but not safe (it
does write data); `GET` is both safe and idempotent.

## Worked example: designing idempotent order creation

`POST /orders` needs retry safety because a flaky mobile network is exactly
the scenario where a client legitimately doesn't know if its request
succeeded.

```bash
curl -X POST https://api.example.com/orders \
  -H "Idempotency-Key: order-checkout-7f3a" \
  -d '{"cart_id": "cart_99", "payment_method": "pm_1"}'
```

Client-side rule: generate the key once when the user taps "Place Order,"
and reuse the *same* key for every retry of that specific checkout attempt
(store it in memory/local state until success or a final failure). A new
tap of "Place Order" — a genuinely new checkout attempt — gets a new key.

## Exercise

1. Classify `DELETE /books/42` called three times in a row. Is it
   idempotent even though the second and third calls return `404` instead
   of `204`? Justify your answer against the definition above.
2. A client retries a `PATCH /accounts/7 {"balance_delta": -100}` request
   after a timeout, and the first request had actually succeeded. Walk
   through the resulting bug, then redesign the endpoint to be safely
   retryable.
3. Why must the idempotency store check the request body, not just the
   key, before replaying a cached response?
4. Design the idempotency-key storage schema (columns/fields) needed to
   safely replay both the response body and the original status code.
