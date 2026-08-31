# 03 · Performance at Scale

An API that's fast at 10 requests/second can fall over at 10,000. This
module covers the techniques that keep latency flat as load grows:
caching layers, connection pooling, async processing, and load testing
to find the breaking point before your customers do.

## Layered caching

```
Client → CDN (edge cache) → API gateway → application cache (Redis) → database
```

```http
GET /v1/products/101 HTTP/1.1
```

```http
HTTP/1.1 200 OK
Cache-Control: public, max-age=300
X-Cache: HIT (edge)
```

A public, cacheable product listing gets served entirely from the CDN
edge for 5 minutes — the origin server never even sees most of that
traffic. Contrast with the private, per-user caching from Level 2's
ETag example, which still hits the origin but skips the database.

## Application-level caching with Redis

```python
def get_product(product_id):
    cached = redis.get(f"product:{product_id}")
    if cached:
        return json.loads(cached)
    product = db.query("SELECT * FROM products WHERE id = %s", product_id)
    redis.setex(f"product:{product_id}", 300, json.dumps(product))
    return product
```

Cache invalidation on write, not just time-based expiry:

```python
def update_product(product_id, data):
    db.execute("UPDATE products SET ... WHERE id = %s", product_id)
    redis.delete(f"product:{product_id}")  # next read repopulates the cache
```

## Connection pooling

Opening a new database connection per request is expensive (TCP
handshake, auth). A pool keeps connections warm and reused:

```python
pool = ConnectionPool(min_size=5, max_size=50)

def handle_request():
    with pool.acquire() as conn:
        return conn.execute(query)
```

Under load, a pool that's too small causes requests to queue waiting
for a connection (visible as a saturation metric, module 9 in Level 3);
too large and the database itself gets overwhelmed by connection
overhead. Sizing it is a measured trade-off, not a guess.

## Async processing: don't make the client wait for slow work

```bash
curl -X POST https://api.example.com/v1/reports \
  -H "Authorization: Bearer $TOKEN" \
  -d '{"type": "annual_sales"}'
```

```json
{ "id": "report_88", "status": "processing" }
```
`HTTP/1.1 202 Accepted`

Generating the report might take two minutes; the client gets an
immediate `202` and a job ID, then polls or receives a webhook (Level 3
module 2) when it's done:

```bash
curl https://api.example.com/v1/reports/report_88 -H "Authorization: Bearer $TOKEN"
```

```json
{ "id": "report_88", "status": "complete", "download_url": "https://..." }
```

## Load testing to find the breaking point

```bash
k6 run --vus 200 --duration 60s loadtest.js
```

```
http_req_duration.....: avg=45ms  p95=210ms  p99=890ms
http_req_failed.......: 0.4%
```

Ramping virtual users up until latency or error rate crosses an
acceptable threshold reveals the actual capacity of a system —
"handles 200 concurrent users at p99 under 1s" is a number you can plan
capacity and alerting thresholds around; "it feels fast" is not.

## N+1 queries: the silent killer

```python
# N+1: one query for orders, then one MORE query per order for its items
orders = db.query("SELECT * FROM orders WHERE user_id = %s", user_id)
for order in orders:
    order.items = db.query("SELECT * FROM items WHERE order_id = %s", order.id)
```

```python
# fixed: one query total, joined or batched
orders = db.query("""
    SELECT orders.*, items.* FROM orders
    JOIN items ON items.order_id = orders.id
    WHERE orders.user_id = %s
""", user_id)
```

At low traffic, N+1 is invisible. At scale, an endpoint that fires 50
extra queries per request multiplies database load 50x under
concurrency — this is one of the most common causes of "it was fine in
staging, it fell over in production."

## Worked example: diagnosing a slowdown under load

A load test shows `p99` latency degrading sharply past 100 concurrent
users, though `p50` stays fine.

1. Check saturation metrics: the DB connection pool is maxed at 50
   during the test — requests above that queue for a connection.
2. A trace on a slow request shows most of its time spent waiting to
   *acquire* a connection, not executing the query itself.
3. Fix: increase pool size *and* add Redis caching for the specific
   read-heavy endpoint causing the pressure, cutting DB query volume
   directly instead of just widening the pipe.
4. Re-run the load test: `p99` now holds flat past 300 concurrent users.

## Exercise

1. Why does a CDN cache reduce origin load more effectively than an
   application-level Redis cache, for public, non-personalized data?
2. Explain why cache invalidation on write is necessary even with a
   short TTL — what's the user-visible symptom if you skip it?
3. Design the async job pattern for an endpoint that resizes an
   uploaded image — what should the immediate response look like, and
   how does the client learn when it's done?
4. A load test shows `p50` latency is fine but `p99` is terrible. What
   does that gap usually indicate, and what would you check first?
