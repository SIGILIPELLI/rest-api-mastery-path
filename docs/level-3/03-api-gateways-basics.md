# 03 · API Gateways Basics

An **API gateway** sits in front of your services as the single entry
point clients talk to. Instead of every client knowing the address of
every backend service, they all hit the gateway, which routes, secures,
and shapes traffic on their behalf.

## What a gateway actually does

- **Routing** — `/v1/orders/*` → orders-service, `/v1/users/*` →
  users-service, based on path, host, or header.
- **Authentication** — validates the token once, at the edge, so
  individual services don't each re-implement JWT verification.
- **Rate limiting** — enforces per-client quotas centrally (see module 9
  of Level 2).
- **TLS termination** — the gateway holds the certificate; internal
  traffic can run plain HTTP inside a trusted network.
- **Request/response transformation** — rewriting headers, aggregating
  multiple backend calls into one client-facing response.
- **Observability** — one place to log every request, emit metrics, and
  trace latency across the whole surface.

## A minimal gateway config (Kong-style, conceptually)

```yaml
services:
  - name: orders-service
    url: http://orders.internal:8080
    routes:
      - name: orders-route
        paths: ["/v1/orders"]
    plugins:
      - name: rate-limiting
        config: { minute: 100 }
      - name: jwt
```

A request from a client never sees `orders.internal` — it only ever
talks to `https://api.example.com/v1/orders`, and the gateway resolves
where that actually goes.

## Routing in action

```bash
curl https://api.example.com/v1/orders/42 \
  -H "Authorization: Bearer $TOKEN"
```

The gateway:

1. Terminates TLS.
2. Validates the JWT signature and expiry.
3. Checks the rate-limit bucket for this client.
4. Matches `/v1/orders/*` to the `orders-service` route.
5. Proxies the request internally, adding a trace header:

```http
GET /orders/42 HTTP/1.1
Host: orders.internal:8080
X-Request-Id: 7f3a-91c2-...
X-Forwarded-For: 203.0.113.9
```

6. Streams the backend's response back to the client unchanged (or
   transformed, per config).

## Gateway vs. load balancer vs. service mesh

| Layer | Operates on | Typical job |
|---|---|---|
| Load balancer | L4/L7, IPs and ports | Spread traffic across replicas |
| API gateway | L7, HTTP semantics | Auth, routing, rate limiting, per-API concerns, client-facing |
| Service mesh (e.g. Istio, Linkerd) | L7, service-to-service | mTLS, retries, circuit breaking *between* internal services |

They're complementary, not competing: a gateway handles north-south
traffic (client → cluster); a mesh handles east-west traffic (service →
service) once the request is inside.

## Aggregation: one client call, many backend calls

A mobile app's dashboard screen needs data from three services. Instead
of the client making three round-trips, the gateway (or a
backend-for-frontend layer behind it) fans out internally:

```bash
curl https://api.example.com/v1/dashboard \
  -H "Authorization: Bearer $TOKEN"
```

```json
{
  "profile": { "id": 42, "name": "Ada" },
  "orders": { "open_count": 2 },
  "notifications": { "unread": 5 }
}
```

Internally this is three parallel calls to `users-service`,
`orders-service`, and `notifications-service`, merged before responding
— saving mobile clients three round-trips over a slow network.

## Worked example: adding a new service behind the gateway

Your team ships a new `reviews-service`. Rollout without breaking
anything:

1. Deploy `reviews-service` internally, unreachable from outside the
   cluster.
2. Add a gateway route: `/v1/reviews/*` → `reviews-service`, with the
   same JWT and rate-limit plugins every other route uses.
3. Deploy behind a feature flag or to a small percentage of traffic
   first (gateway-level canary routing) if the platform supports it.
4. Only after the gateway route is live does `reviews-service` become
   reachable by clients at all — until then it's fully internal, so
   there's zero external-facing risk during development.

## Exercise

1. A client complains that two different backend services return
   inconsistent error formats for the same kind of failure (missing
   auth). How would centralizing auth at the gateway fix this?
2. Why does TLS termination usually happen at the gateway rather than at
   every individual service?
3. Explain the north-south vs. east-west distinction and give an example
   of a concern that belongs at each layer.
4. Your gateway aggregates three backend calls into one response. What
   should happen if one of the three backend calls fails — return
   partial data, or fail the whole request? Justify your answer for a
   dashboard endpoint specifically.
