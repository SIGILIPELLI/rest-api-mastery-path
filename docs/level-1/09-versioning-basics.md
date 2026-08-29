# 09 · API Versioning Basics

APIs change over time — fields get added, removed, or renamed; behavior
shifts. Versioning is how you make those changes without breaking every
existing client overnight. This module covers the core strategies; Level 3's
"Backward Compatibility & Deprecation" module goes deeper into managing the
transition itself.

## Why you need it at all

Once an API has real consumers you don't control (third-party developers,
a mobile app that isn't force-updated, another team's service), you cannot
simply change a field's meaning or remove an endpoint — someone's code will
break in production without warning. Versioning gives you a controlled way
to introduce breaking changes while existing consumers keep working against
the old behavior until they're ready to migrate.

## What counts as a breaking change

| Change | Breaking? |
|---|---|
| Adding a new optional field to a response | No |
| Adding a new endpoint | No |
| Adding a new optional query parameter | No |
| Removing a field from a response | **Yes** |
| Renaming a field | **Yes** |
| Changing a field's type (string → number) | **Yes** |
| Changing validation to be *more* permissive | No |
| Changing validation to be *more* strict | **Yes** |
| Changing the meaning of an existing status code | **Yes** |

Rule of thumb: if existing, unmodified client code could start behaving
incorrectly or erroring out because of your change, it's breaking.

## Strategy 1: URL path versioning

```
GET /v1/books/17
GET /v2/books/17
```

**Pros**: extremely visible and explicit — a developer can see the version
just by looking at the URL, and it's trivial to route different versions to
entirely different backend implementations, or even different servers.

**Cons**: technically, the "same resource" now has two different URLs
(`/v1/books/17` and `/v2/books/17`), which is a minor violation of REST's
idea that a URL identifies one resource — though this is treated as an
acceptable, pragmatic tradeoff by the overwhelming majority of real APIs
(Stripe, GitHub, Twitter/X all use some form of this).

```bash
curl https://api.example.com/v1/books/17
curl https://api.example.com/v2/books/17
```

## Strategy 2: header versioning

The URL stays the same; the client requests a version via a custom header.

```bash
curl https://api.example.com/books/17 \
  -H "X-API-Version: 2"
```

**Pros**: keeps the URL as the single, stable identifier for a resource —
more architecturally "pure" REST.

**Cons**: less discoverable (you can't tell the version from the URL alone
when debugging or sharing a link); easy for a client to forget the header
and silently get a default version.

## Strategy 3: Accept header / media type versioning (content negotiation)

```bash
curl https://api.example.com/books/17 \
  -H "Accept: application/vnd.example.v2+json"
```

**Pros**: uses HTTP's built-in content negotiation mechanism as intended —
technically the most "correct" REST approach, since it treats a version as
a different *representation* of the same resource, not a different
resource.

**Cons**: least intuitive for most developers, more friction to test
manually (you can't just paste a URL in a browser), and tooling support is
patchier.

## Strategy 4: query parameter versioning

```bash
curl "https://api.example.com/books/17?version=2"
```

**Pros**: simple, visible in the URL like path versioning.

**Cons**: mixes a structural concern (the API's contract shape) with query
parameters that usually mean "filter/sort/paginate" — most style guides
consider this the weakest of the four options, though it's simple to
implement and does see real-world use.

## Comparing the strategies

| Strategy | Visible in URL? | REST purity | Real-world adoption |
|---|---|---|---|
| URL path (`/v1/...`) | Yes | Lower | Very high (Stripe, Twitter, GitHub REST) |
| Custom header | No | Medium | Moderate |
| `Accept` media type | No | Highest | Low-moderate (GitHub uses a variant of this too) |
| Query parameter | Yes | Lowest | Low |

**For this course, and for most new APIs, URL path versioning
(`/v1/...`) is the pragmatic default** — it's the easiest to understand,
debug, document, and route, even though it's not the most "theoretically
pure" REST approach.

## Only version the major, breaking changes

```
/v1/books    (original)
/v2/books    (after a breaking change — e.g. `author` changed from a string
              to a nested object)
```

Non-breaking additions (new optional fields, new endpoints) should **not**
require a version bump — that would force every client to migrate constantly
for changes that don't actually affect them. Save version bumps for genuine
breaking changes.

## Worked example: a breaking change and its version bump

Version 1 returns `author` as a plain string:

```json
GET /v1/books/17
{ "id": 17, "title": "Dune", "author": "Frank Herbert" }
```

The team decides authors need more structure (an ID, for linking to an
author resource). This is breaking — any `v1` client parsing `author` as a
string would break if this changed in place. So it ships as `v2`:

```json
GET /v2/books/17
{ "id": 17, "title": "Dune", "author": { "id": 3, "name": "Frank Herbert" } }
```

`v1` keeps working unchanged (the server maintains both response shapes,
usually by having `v2` be the "real" internal model and `v1` be a thin
compatibility-transforming layer on top of it) until it's formally
deprecated and eventually retired — the process covered in Level 3.

## Exercise

1. You need to rename a field from `full_name` to `name` in your `/users`
   endpoint. Is this breaking? What are your options, and which versioning
   strategy would you use to ship it?
2. Compare `/v2/orders` vs. sending `Accept:
   application/vnd.myapi.v2+json` — write one sentence on when you'd prefer
   each, from a "how easy is this to debug with a browser or curl" angle.
3. You're adding a new, optional `discount_code` field that a client can
   include when creating an order — existing clients that don't send it are
   unaffected. Does this need a version bump? Justify your answer using the
   "breaking change" table above.
