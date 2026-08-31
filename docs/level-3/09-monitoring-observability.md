# 09 · Monitoring & Observability for APIs

You can't fix what you can't see. This module covers the three pillars
of observability — logs, metrics, traces — applied specifically to a
REST API, plus the alerts that let a team find out about a problem
before customers do.

## The four golden signals

For any API endpoint, track:

- **Latency** — how long requests take (and split success vs. error
  latency separately — errors are often suspiciously fast, e.g. an
  early auth rejection).
- **Traffic** — requests per second, per endpoint.
- **Errors** — rate of 4xx and 5xx responses.
- **Saturation** — how full the system is (CPU, memory, connection pool,
  queue depth) — the leading indicator before latency/errors spike.

## Structured logging

```json
{
  "timestamp": "2026-08-31T14:22:01Z",
  "level": "info",
  "request_id": "7f3a-91c2-4b6d",
  "method": "POST",
  "path": "/v1/orders",
  "status": 201,
  "duration_ms": 84,
  "user_id": 42
}
```

Structured (JSON) logs, not free-text strings, so they're queryable:
"show me every 5xx on `/v1/orders` in the last hour" is a query, not a
grep-and-pray exercise. `request_id` is generated at the gateway
(module 3) and threaded through every log line and downstream call for
that request.

## Metrics: the numbers a dashboard shows

```bash
curl https://api.example.com/metrics
```

```
http_requests_total{method="POST",path="/v1/orders",status="201"} 18432
http_request_duration_seconds{path="/v1/orders",quantile="0.99"} 0.412
http_requests_total{method="POST",path="/v1/orders",status="500"} 12
```

Prometheus-style metrics like these feed dashboards and alerting rules.
The key metric to alert on is usually **error rate**, not raw error
count — 12 errors out of 18,432 requests (0.07%) is very different from
12 errors out of 40 requests (30%).

## Distributed tracing

A single client request to `/v1/dashboard` (module 3's aggregation
example) fans out to three internal services. A trace ties all of it
together under one `trace_id`:

```
trace_id: 7f3a91c2
├─ gateway            2ms
├─ users-service      GET /users/42          18ms
├─ orders-service     GET /orders?user=42    45ms
│   └─ postgres query                        38ms  ← the actual bottleneck
└─ notifications-svc  GET /unread?user=42    12ms
```

Without tracing, "the dashboard is slow" is a mystery. With it, the
45ms in `orders-service`, mostly a slow Postgres query, is immediately
visible — versus a network problem, a queue backup, or a slow client.

## Alerting: telling a human before a customer does

```yaml
alert: HighErrorRate
expr: rate(http_requests_total{status=~"5.."}[5m]) / rate(http_requests_total[5m]) > 0.05
for: 5m
annotations:
  summary: "5xx rate above 5% on {{ $labels.path }} for 5 minutes"
```

Good alerts are: actionable (someone can do something about it),
symptom-based ("customers are seeing errors") rather than cause-based
("CPU is at 80%" — that alone might be fine), and rate-limited so a
single incident doesn't page the same person 50 times.

## Worked example: diagnosing a production incident

Dashboard shows `p99 latency` on `POST /v1/orders` jumped from 100ms to
4s starting at 14:20.

1. Check traces from that window — nearly all show the time spent
   inside a single downstream call: `payments-service`.
2. Check `payments-service`'s own dashboard — its error rate is fine,
   but its own p99 latency also jumped at 14:20, and its saturation
   metric (DB connection pool usage) is pegged at 100%.
3. Root cause: a slow query introduced in a `payments-service` deploy at
   14:18 is holding connections longer, exhausting the pool, backing up
   every caller.
4. Fix: roll back the `payments-service` deploy; latency across the
   whole chain recovers within a minute.

The trace made it a two-minute diagnosis instead of an hour of guessing
which of a dozen services was actually at fault.

## Exercise

1. Why alert on error *rate* rather than raw error *count*?
2. A request takes 3 seconds end-to-end but each individual service's
   logs show sub-100ms processing time. What would a distributed trace
   likely reveal that logs alone wouldn't?
3. Explain saturation as a "leading indicator" — why would you want to
   alert on high queue depth before latency actually degrades?
4. Design a structured log line for a failed login attempt, including
   fields useful for both debugging and security auditing.
