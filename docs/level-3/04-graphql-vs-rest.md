# 04 · GraphQL vs REST

REST models an API as a set of resources reached by URL. GraphQL models
it as a single endpoint with a typed schema the client queries — the
client asks for exactly the fields it needs, in one round trip, no more.

## The core difference, side by side

REST — three requests to render a profile page with recent orders:

```bash
curl https://api.example.com/v1/users/42
curl https://api.example.com/v1/users/42/orders?limit=5
curl https://api.example.com/v1/users/42/orders/91/items
```

GraphQL — one request, one response, exactly the shape asked for:

```bash
curl -X POST https://api.example.com/graphql \
  -H "Content-Type: application/json" \
  -d '{
    "query": "{ user(id: 42) { name orders(limit: 5) { id total items { name } } } }"
  }'
```

```json
{
  "data": {
    "user": {
      "name": "Ada",
      "orders": [
        { "id": 91, "total": 42.50, "items": [ { "name": "Widget" } ] }
      ]
    }
  }
}
```

## Why teams reach for GraphQL

- **Over-fetching** — a REST `/users/42` might return 30 fields when the
  client needs 3; GraphQL returns only the requested fields.
- **Under-fetching / N+1 round trips** — REST often needs a chain of
  calls (user → orders → items); GraphQL does it in one.
- **Client-driven shape** — different clients (mobile vs. web) can ask
  for different fields from the same schema without new backend
  endpoints.

## Why REST still wins in most cases

- **HTTP caching** — REST's `GET /v1/users/42` is cacheable by URL with
  standard `Cache-Control`/`ETag` (module 7, Level 2). GraphQL's single
  `POST /graphql` endpoint defeats HTTP caches entirely; caching has to
  be reimplemented at the application layer (e.g. persisted queries +
  a normalized client cache).
- **Simplicity** — REST's status codes and resource model are
  understood by every HTTP tool (curl, browsers, proxies, CDNs) with no
  extra tooling. GraphQL needs a client library to be pleasant to use.
- **File uploads, webhooks, streaming** — all more natural as plain
  HTTP/REST than as GraphQL operations.
- **Rate limiting and cost control** — a REST endpoint has a
  predictable cost. A single GraphQL query can ask for deeply nested
  data that's expensive to compute, so servers need query complexity
  analysis to avoid abuse.

## What a GraphQL schema looks like

```graphql
type User {
  id: ID!
  name: String!
  orders(limit: Int = 20): [Order!]!
}

type Order {
  id: ID!
  total: Float!
  items: [Item!]!
}

type Query {
  user(id: ID!): User
}
```

The schema is the contract — analogous to an OpenAPI spec (module 7,
Level 1), but the client, not the server, decides the response shape at
request time.

## Errors in GraphQL: always HTTP 200

A key gotcha: GraphQL responses are almost always `200 OK`, even on
error — the error lives inside the JSON body:

```json
{
  "data": { "user": null },
  "errors": [
    { "message": "User 42 not found", "path": ["user"], "extensions": { "code": "NOT_FOUND" } }
  ]
}
```

This trips up REST-trained tooling that checks status codes to detect
failure; GraphQL clients must always inspect `errors`.

## Worked example: choosing for a real product

A team is building a public API for a project-management tool with
many different frontends (web, mobile, third-party integrations) that
each need different slices of deeply nested data (projects → tasks →
comments → attachments).

- If most consumers are internal frontends with fast-changing data
  needs → GraphQL reduces backend churn from "add another endpoint for
  this screen."
- If the API is also meant to be a stable, cacheable, publicly
  documented product other companies integrate against long-term →
  REST's URL-addressable resources and HTTP caching win, and many teams
  end up shipping **both**: REST for the stable public surface, GraphQL
  for an internal BFF layer.

## Exercise

1. Explain why a CDN can cache a REST `GET` response but generally
   cannot cache a GraphQL `POST /graphql` response without extra work.
2. A GraphQL query returns `"data": {"user": null}` and an `errors`
   array, with HTTP status `200`. Why can't a naive REST-style client
   that only checks the status code detect this failure?
3. Design a GraphQL query for a blog: fetch a post's title and the
   first 3 comments' author names. Then show what that would look like
   as REST endpoint calls.
4. What is query-complexity analysis, and why does a GraphQL server
   need it in a way a REST API generally doesn't?
