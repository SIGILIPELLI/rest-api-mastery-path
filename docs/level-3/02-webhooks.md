# 02 · Webhooks

Every module so far has been about a client pulling data from a server on
demand. Webhooks invert that: the **server pushes** an event to a URL the
client registered, the moment something happens — no polling required.

## Why webhooks instead of polling

```bash
# Polling: client keeps asking "did anything change?" — wasteful, and laggy
# by up to the poll interval
while true; do
  curl "https://api.example.com/orders?updated_after=$LAST_CHECK"
  sleep 30
done
```

```
# Webhook: server tells the client the instant an order ships
POST https://myapp.com/webhooks/orders  (sent by api.example.com, not by the client)
{"event": "order.shipped", "order_id": 42, "tracking": "1Z999..."}
```

Webhooks give near-real-time delivery with none of the wasted requests
polling generates, at the cost of the receiver needing a publicly
reachable, always-on endpoint.

## Anatomy of a webhook event

```http
POST /webhooks/orders HTTP/1.1
Host: myapp.com
Content-Type: application/json
X-Webhook-Signature: sha256=7d38cdd689735b008b3c702edd92eea23791c5f6
X-Webhook-Id: evt_8f14e45f
X-Webhook-Timestamp: 1699999999

{
  "event": "order.shipped",
  "id": "evt_8f14e45f",
  "created_at": "2026-08-29T10:00:00Z",
  "data": { "order_id": 42, "tracking_number": "1Z999AA10123456784" }
}
```

The receiving endpoint is just a normal REST endpoint from the receiver's
side — it accepts a `POST`, and should respond quickly:

```http
HTTP/1.1 200 OK
```

## Verifying webhook signatures

Because a webhook endpoint is a public URL, anyone who discovers it could
`POST` fake events unless the receiver verifies the sender's identity. The
standard mechanism: the sender computes an HMAC signature over the raw
request body using a shared secret, and the receiver recomputes it to
confirm the payload wasn't forged or tampered with in transit.

```python
import hmac, hashlib

def verify_signature(raw_body: bytes, signature_header: str, secret: str) -> bool:
    expected = "sha256=" + hmac.new(secret.encode(), raw_body, hashlib.sha256).hexdigest()
    return hmac.compare_digest(expected, signature_header)
```

```python
@app.post("/webhooks/orders")
def handle_webhook(request):
    raw_body = request.get_data()
    if not verify_signature(raw_body, request.headers["X-Webhook-Signature"], WEBHOOK_SECRET):
        return "invalid signature", 401
    event = json.loads(raw_body)
    process_event(event)
    return "", 200
```

Using `hmac.compare_digest` (constant-time comparison) instead of `==`
matters — a naive string comparison leaks timing information an attacker
could use to guess the correct signature byte by byte.

## Idempotent processing

Webhook delivery is "at-least-once," not "exactly-once" — network issues or
receiver downtime cause the sender to retry, so the same event ID may
arrive more than once. The receiver must dedupe using the event ID (module
5's idempotency-key pattern, applied on the receiving side this time):

```python
def process_event(event):
    if events_seen.exists(event["id"]):
        return  # already processed, ignore duplicate delivery
    events_seen.add(event["id"], ttl=7 * 24 * 3600)
    apply_side_effects(event)
```

## Responding fast, processing async

A webhook handler should acknowledge receipt immediately and do the actual
work off the request path — the sender usually times out and retries after
just a few seconds, so slow processing inside the handler risks the sender
treating a *successful* delivery as failed and retrying unnecessarily.

```python
@app.post("/webhooks/orders")
def handle_webhook(request):
    event = verify_and_parse(request)
    job_queue.enqueue("process_order_event", event)   # hand off, don't block
    return "", 200   # ack immediately
```

## Retry and backoff (sender side)

A well-behaved webhook sender retries failed deliveries (`5xx` responses,
timeouts, connection errors) with exponential backoff, and gives up after a
bounded number of attempts:

```
Attempt 1: immediately        → 500 → retry
Attempt 2: after 1 minute     → timeout → retry
Attempt 3: after 5 minutes    → 500 → retry
Attempt 4: after 30 minutes   → 200 → done
```

Most providers (Stripe, GitHub) also expose a dashboard of recent
deliveries with response codes and a manual "redeliver" button, since a
receiver's downtime window can otherwise silently drop events past the
retry budget.

## Webhook payload versioning

Just like a REST response shape, webhook event payloads need a
versioning story — a receiver that hard-codes `data.order_id` breaks if
that field is later renamed. A common approach: an `api_version` field on
every event, matching the sender account's configured API version, so
receivers can pin to a stable shape:

```json
{ "event": "order.shipped", "api_version": "2026-06-01", "data": { "order_id": 42 } }
```

## Worked example: designing a webhook system for order events

```bash
# Registering a webhook endpoint
curl -X POST https://api.example.com/v1/webhooks \
  -H "Authorization: Bearer $TOKEN" \
  -d '{"url": "https://myapp.com/webhooks/orders", "events": ["order.shipped", "order.cancelled"]}'
```

```json
{ "id": "wh_1", "url": "https://myapp.com/webhooks/orders", "secret": "whsec_8f14e45f...", "events": ["order.shipped", "order.cancelled"] }
```

The `secret` is shown once at creation — the receiver stores it to verify
future signatures; it's never retrievable again afterward (only
regenerable), the same convention as an API key.

## Exercise

1. Why is HMAC signature verification necessary even though the webhook
   URL is only known to the sender and receiver (i.e., why isn't secrecy
   of the URL itself sufficient)?
2. Design the idempotency handling for a receiver that must apply
   `order.shipped` events to update inventory exactly once, given
   at-least-once delivery.
3. A webhook handler does heavy processing (sending a confirmation email,
   updating three downstream systems) synchronously before returning
   `200`. What goes wrong under the sender's retry policy, and how would
   you fix it?
4. What should a receiver do if it needs to change how it interprets
   `order.shipped` payloads in a breaking way, given the sender supports
   `api_version` pinning?
