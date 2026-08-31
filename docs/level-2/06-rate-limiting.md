# 06 · Rate Limiting

Rate limiting caps how many requests a client can make in a given time
window, protecting the API from being overwhelmed — whether by a runaway
client bug, an aggressive scraper, or a genuine traffic spike — and keeping
usage fair across all consumers.

## Common algorithms

### Fixed window

Count requests per client in a fixed time bucket (e.g. per minute), reset
the counter at the boundary.

```
Window: 12:00:00–12:00:59 → limit 100
Client makes 100 requests at 12:00:58 → all allowed
Window resets at 12:01:00 → counter back to 0
Client makes 100 more requests at 12:01:01 → all allowed
```

**Problem**: a client can burst 200 requests in the 2 seconds straddling a
window boundary (100 just before 12:01:00, 100 just after), well above the
intended "100 per minute" rate.

### Sliding window

Instead of resetting at fixed boundaries, count requests in the trailing N
seconds from *now*, recomputed continuously. This smooths out the
boundary-burst problem at the cost of slightly more bookkeeping (usually a
sorted set or a weighted combination of the current and previous fixed
windows).

### Token bucket

A bucket holds tokens, refilled at a steady rate up to a cap; each request
consumes one token, and a request is rejected if the bucket is empty. This
naturally allows short bursts (up to the bucket size) while enforcing a
long-run average rate — the algorithm behind most production rate limiters,
including AWS API Gateway's.

```
Bucket capacity: 20 tokens, refill rate: 10 tokens/sec
t=0:  bucket full (20) → client bursts 20 requests instantly → all allowed, bucket now 0
t=1s: bucket refilled to 10 → 10 more requests allowed
```

### Leaky bucket

Requests queue up and are processed (or discarded) at a constant rate —
smooths bursts into a steady outflow rather than allowing them, trading
burst tolerance for a perfectly steady processing rate. Common in network
traffic shaping more than in typical HTTP API gateways.

## Communicating limits to clients

Standard-ish response headers (formalized by the IETF `RateLimit` header
draft, and used in similar form by GitHub, Twitter/X, and Stripe):

```http
HTTP/1.1 200 OK
RateLimit-Limit: 100
RateLimit-Remaining: 42
RateLimit-Reset: 37
```

`RateLimit-Reset` here means "seconds until the window/bucket resets."
GitHub's API uses slightly different names but the same idea:

```http
X-RateLimit-Limit: 5000
X-RateLimit-Remaining: 4987
X-RateLimit-Reset: 1699999999
```

## The 429 response

When a client exceeds its limit, return `429 Too Many Requests` with a
`Retry-After` header telling it exactly when to try again:

```bash
curl -i https://api.example.com/books
```

```http
HTTP/1.1 429 Too Many Requests
Retry-After: 23
Content-Type: application/json

{
  "error": {
    "code": "rate_limited",
    "message": "Rate limit exceeded. Retry after 23 seconds.",
    "retry_after": 23
  }
}
```

`Retry-After` can be seconds (`23`) or an HTTP date; sending it removes any
guesswork for a well-behaved client, which should back off until that time
rather than hammering the endpoint immediately.

## Per-key vs per-IP limiting

Limiting purely by IP breaks down behind shared NATs or corporate proxies
(many legitimate users share one IP) and is trivially evaded with rotating
IPs. Production APIs almost always key limits on the authenticated
principal — API key, user ID, or OAuth client ID — falling back to IP only
for unauthenticated endpoints.

```python
def rate_limit_key(request):
    if request.api_key:
        return f"key:{request.api_key}"
    return f"ip:{request.remote_addr}"
```

## Tiered limits

Different consumers often get different limits based on their plan:

```json
{
  "free_tier":    { "limit": 100,   "window_seconds": 60 },
  "pro_tier":     { "limit": 5000,  "window_seconds": 60 },
  "enterprise":   { "limit": 100000, "window_seconds": 60 }
}
```

Document per-tier limits clearly (Level 4 covers this further under public
developer APIs) — a client silently throttled without knowing its tier's
ceiling is a common source of support tickets.

## Worked example: implementing token bucket with Redis

```python
import time

def allow_request(client_key, capacity=20, refill_rate=10):
    now = time.time()
    bucket = redis.hgetall(client_key) or {"tokens": capacity, "ts": now}
    elapsed = now - float(bucket["ts"])
    tokens = min(capacity, float(bucket["tokens"]) + elapsed * refill_rate)

    if tokens < 1:
        redis.hset(client_key, mapping={"tokens": tokens, "ts": now})
        return False  # reject -> 429

    redis.hset(client_key, mapping={"tokens": tokens - 1, "ts": now})
    return True  # allow
```

This is a simplified sketch (a production version needs this check to be
atomic, typically via a Lua script executed inside Redis, to avoid a
race between concurrent requests reading and writing the same bucket).

## Exercise

1. Explain concretely why fixed-window counting allows a 2x burst at
   window boundaries, using a timeline like the one above.
2. Design the rate-limit response headers and body for a client that has
   exhausted its daily quota (not a per-minute one) — what should
   `Retry-After` be set to?
3. Why is per-IP rate limiting insufficient for an API used heavily from
   behind corporate NAT gateways or mobile carrier NAT? What's the
   preferred alternative?
4. A token bucket has capacity 20 and refill rate 5/sec. A client sends 20
   requests instantly, then 5 more three seconds later. How many of the 25
   total requests are allowed, and why?
