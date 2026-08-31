# 03 · HATEOAS Basics

HATEOAS — **H**ypermedia **A**s **T**he **E**ngine **O**f **A**pplication
**S**tate — is the most-cited, least-implemented part of Fielding's uniform
interface constraint. The idea: a client should be able to navigate an
entire API by following links returned in responses, the same way a human
navigates the web by clicking links, rather than by hard-coding every URL
it might ever need.

## The problem it solves

Without hypermedia, a client hard-codes knowledge like "to cancel an order,
`POST` to `/orders/{id}/cancel`." If the server later changes that endpoint,
or if cancellation is only allowed for orders in certain states, every
client needs to know that business logic independently.

```json
// Without HATEOAS — client must already know every possible action and URL
{ "id": 42, "status": "pending", "total": 99.99 }
```

```json
// With HATEOAS — the response tells the client what it can do next
{
  "id": 42,
  "status": "pending",
  "total": 99.99,
  "_links": {
    "self":   { "href": "/orders/42" },
    "cancel": { "href": "/orders/42/cancel", "method": "POST" },
    "items":  { "href": "/orders/42/items" }
  }
}
```

If the order were already shipped, the server simply omits the `cancel`
link — the client doesn't need its own copy of "orders can only be
cancelled while pending"; it just checks whether the link is present.

## Link formats

There's no single mandatory format, but two are widely used:

**HAL (Hypertext Application Language)**, `application/hal+json`:

```json
{
  "id": 42,
  "status": "pending",
  "_links": {
    "self": { "href": "/orders/42" },
    "customer": { "href": "/customers/7" }
  },
  "_embedded": {
    "items": [
      { "id": 1, "sku": "BOOK-1", "_links": { "self": { "href": "/items/1" } } }
    ]
  }
}
```

**JSON:API**, `application/vnd.api+json`:

```json
{
  "data": {
    "type": "orders",
    "id": "42",
    "attributes": { "status": "pending" },
    "relationships": {
      "customer": { "links": { "related": "/orders/42/customer" } }
    },
    "links": { "self": "/orders/42" }
  }
}
```

Both formats standardize *where* links live in the payload so generic
client libraries can parse them without custom code per API.

## A minimal ad-hoc convention

Most real-world APIs skip a formal hypermedia spec and use a plain `_links`
or `links` object, HAL-ish but without full HAL compliance:

```bash
curl https://api.example.com/orders/42
```

```json
{
  "id": 42,
  "status": "shipped",
  "links": {
    "self": "/orders/42",
    "tracking": "/orders/42/tracking",
    "invoice": "/orders/42/invoice"
  }
}
```

This gets most of the practical benefit — discoverability, and no cancel
link when cancellation isn't valid — without adopting a full spec.

## The root/entry-point pattern

A fully hypermedia-driven API only publishes one fixed URL: the root. Every
other URL is discovered by following links from there.

```bash
curl https://api.example.com/
```

```json
{
  "links": {
    "orders": "/orders",
    "customers": "/customers",
    "docs": "/docs"
  }
}
```

A client that only ever hits `/` and follows `links.orders` survives the
server renaming `/orders` to `/v2/orders`, as long as the root link's key
(`orders`) stays the same. This is the theoretical payoff of HATEOAS: URL
structure becomes an implementation detail instead of a client contract.

## Why most real APIs don't fully adopt it

- **Client complexity**: following links dynamically is more code than
  hard-coding a URL template, for marginal day-to-day benefit once a URL
  scheme is stable.
- **Tooling**: most client generators (OpenAPI-based SDKs) work off fixed
  paths, not discovered links, so hypermedia and auto-generated clients
  pull in different directions.
- **Versioning already solves most of what HATEOAS solves**: an explicit
  `/v1/`, `/v2/` scheme (Level 1, Module 9) gives clients a stable contract
  without needing runtime link discovery.

In practice, most APIs adopt a *partial* HATEOAS: `self` links and
state-dependent action links (like the `cancel` example above) because they
add real value, without going all the way to a fully link-driven client.

## Worked example: state-dependent links on an order

```bash
curl https://api.example.com/orders/42   # status: pending
```

```json
{ "id": 42, "status": "pending",
  "links": { "self": "/orders/42", "cancel": "/orders/42/cancel", "pay": "/orders/42/pay" } }
```

```bash
curl https://api.example.com/orders/42   # after payment, status: paid
```

```json
{ "id": 42, "status": "paid",
  "links": { "self": "/orders/42", "ship": "/orders/42/ship", "refund": "/orders/42/refund" } }
```

The set of available actions changes as the resource moves through its
state machine, and the client reads that from the response instead of
re-implementing the state machine itself.

## Exercise

1. Model an `Article` resource that can be `draft`, `published`, or
   `archived`. Write the `links` object for each state, including which
   actions (edit, publish, archive, unpublish) are valid from each.
2. What breaks for a client that hard-codes `/orders/{id}/cancel` if the
   server later renames that route to `/orders/{id}/void`? How would a
   `links`-driven client be unaffected?
3. Why does the root/entry-point pattern reduce coupling between client and
   server even when most concrete resource URLs never actually change?
4. Give one concrete reason a mobile app team might reasonably choose
   *not* to build a link-following client, even knowing the theoretical
   benefits.
