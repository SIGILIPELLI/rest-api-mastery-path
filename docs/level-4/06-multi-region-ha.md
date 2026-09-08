# 06 · Multi-Region & High Availability APIs

A single-region API has a hard ceiling: if that region's data center
has a bad day, every customer worldwide is down, and customers far from
it always pay extra latency. This module covers running an API across
multiple regions for both resilience and speed.

## Why multi-region

- **Latency** — a customer in Singapore calling a US-only API pays
  ~200ms of pure network transit before any processing even starts;
  a Singapore region serves them in single-digit milliseconds.
- **Availability** — an entire region going down (power, network,
  provider outage) shouldn't mean total downtime if traffic can shift
  to another region.
- **Regulatory** — data residency laws sometimes require EU customer
  data to stay in EU infrastructure.

## Active-active vs. active-passive

**Active-passive**: one region serves all live traffic; a second region
stands by, replicated but idle, promoted only on failure. Simpler, but
wastes capacity and the failover itself takes time (DNS propagation,
promotion scripts).

**Active-active**: multiple regions serve live traffic simultaneously,
each routed the nearest customer.

```bash
curl https://api.example.com/v1/orders   # DNS/anycast routes to nearest healthy region
```

```http
HTTP/1.1 200 OK
X-Served-By: us-east-1
```

Harder to build correctly (data written in one region needs to be
consistent with — or explicitly reconciled against — the same data
written in another) but no wasted capacity and a region failure only
removes a fraction of total capacity, not all of it.

## Routing traffic: GeoDNS and health checks

```
customer in Tokyo    → resolves api.example.com → 203.0.113.10 (ap-northeast-1)
customer in Frankfurt → resolves api.example.com → 198.51.100.20 (eu-central-1)
```

```bash
curl https://ap-northeast-1.internal.example.com/healthz
```

```json
{ "status": "healthy", "region": "ap-northeast-1" }
```

A region that starts failing health checks is automatically removed
from DNS/routing so traffic reroutes to the next-nearest healthy
region within seconds — without any human intervening.

## Data consistency across regions

The hard part isn't running the API in multiple places — stateless
application servers replicate trivially. It's the **database**.

```
Strategy                    | Consistency        | Write latency
-----------------------------|--------------------|--------------
Single primary, one region   | Strong             | Fast (local), slow (remote writes)
Multi-primary, all regions    | Eventual, conflicts possible | Fast everywhere
Read replicas per region      | Reads local, writes to primary | Fast reads, single write bottleneck
```

Most real systems pick "read replicas per region, single write
primary" as a pragmatic middle ground: a customer's `GET /v1/orders`
is served from the nearest regional replica (fast reads everywhere),
but `POST /v1/orders` still routes to the primary region (writes are
consistent, if slightly slower for far-away customers).

```bash
curl -X POST https://api.example.com/v1/orders \
  -H "Authorization: Bearer $TOKEN"
```

```http
HTTP/1.1 201 Created
X-Served-By: us-east-1 (primary)
```

```bash
curl https://api.example.com/v1/orders/42 \
  -H "Authorization: Bearer $TOKEN"
```

```http
HTTP/1.1 200 OK
X-Served-By: ap-northeast-1 (replica)
```

## Handling replication lag

A customer creates an order in `us-east-1` (the primary), then
immediately reads it back from `ap-northeast-1` (a replica) before
replication catches up — a classic read-your-own-writes problem.

```json
{ "error": { "code": "not_found" } }
```

...even though the order exists. Common fixes: route a user's reads to
the primary region for a short window right after they write
(session-affinity), or have the client retry with backoff, or return a
`202`-style "still propagating" hint rather than a bare `404`.

## Worked example: designing failover

`eu-central-1` goes down entirely for 20 minutes.

1. Health checks against `eu-central-1` start failing within seconds;
   GeoDNS stops routing new traffic there.
2. European customers' requests reroute to `eu-west-1`, adding ~15ms of
   latency but zero downtime.
3. Any writes that were in-flight to `eu-central-1`'s primary at the
   moment of failure are the actual risk — this is why the write
   primary needs its own documented failover runbook (promote a
   replica, verify no split-brain) separate from the stateless
   application layer's automatic rerouting.
4. Post-incident: measure how much replication lag existed at failure
   time, to bound possible data loss and confirm the recovery point
   objective (RPO) was met.

## How It Actually Works

Running an API across regions means confronting real physics — light
(and therefore network packets) takes a measurable, non-negotiable amount
of time to cross a continent — and real distributed-systems tradeoffs
around consistency.

**DNS-based geo-routing**: a client's DNS resolver queries for
`api.example.com`, and a geo-aware DNS server (like Route 53 latency-based
routing) returns a *different IP* depending on the resolver's
geographic location — this happens entirely at the DNS layer, before any
HTTP request is even formed, meaning a routing failure here (returning a
region's IP that's currently down) isn't something your application code
can see or fix; only DNS health-checks removing that region's records
from rotation address it, and DNS caching (TTLs) means that fix
propagates with a delay, not instantly.

**Multi-region data consistency** is the harder mechanism: if region A
and region B both accept writes to the same logical record
(active-active), you need conflict resolution — typically **last-write-
wins** using a synchronized timestamp (itself a hard problem: clock drift
between regions means "last" isn't unambiguous without something like
Google Spanner's TrueTime or a logical clock/vector clock scheme), or you
route all writes for a given record to one "home" region and treat other
regions as read replicas (active-passive), avoiding the conflict entirely
at the cost of higher write latency for clients far from the home region.

**Health checks driving failover**: a load balancer or DNS service
periodically calls a `/health` endpoint; consecutive failures beyond a
threshold remove that region from active rotation — the threshold and
check interval directly determine your actual failover time (a 5-second
interval with a 3-failure threshold means roughly 15 seconds of failed
requests before traffic reroutes), which is a real, calculable number, not
a marketing claim of "automatic failover."

## Exercise

1. Why does an active-active architecture handle a regional outage
   better than active-passive, in terms of both downtime and wasted
   capacity?
2. Explain read-your-own-writes and design one fix for it in a
   read-replica-per-region architecture.
3. A single global write primary means every write pays cross-region
   latency for customers far from it. What's the trade-off of moving to
   multi-primary writes instead?
4. Design the health-check contract (endpoint, response, failure
   threshold) that GeoDNS routing should use to decide a region is
   unhealthy.
