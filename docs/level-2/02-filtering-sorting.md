# 02 · Filtering & Sorting

Pagination controls *how many* items come back; filtering and sorting
control *which* items and *in what order*. Together they turn a flat
collection endpoint into something a real client can actually use to answer
questions like "show me the 10 cheapest in-stock books published after
2020."

## Filtering with query parameters

The most common convention: one query parameter per filterable field.

```bash
curl "https://api.example.com/books?genre=scifi&in_stock=true"
```

```json
{
  "data": [
    { "id": 12, "title": "Dune", "genre": "scifi", "in_stock": true },
    { "id": 45, "title": "Foundation", "genre": "scifi", "in_stock": true }
  ],
  "meta": { "total": 2 }
}
```

Multiple filters are implicitly AND-ed together. This convention is simple,
self-documenting in a URL, and cache-friendly (the whole query string is
part of the cache key).

### Range and comparison filters

Plain equality isn't enough for numeric or date fields. A common pattern
uses suffixed operators:

```bash
curl "https://api.example.com/books?price_gte=10&price_lte=30&published_after=2020-01-01"
```

Stripe uses this style for its `created[gte]`/`created[lte]` filters:

```bash
curl "https://api.stripe.com/v1/charges?created[gte]=1609459200&created[lte]=1612137600"
```

Both are valid; pick one convention and apply it consistently across every
endpoint. Mixing `price_gte=10` on one resource and `min_price=10` on
another is the kind of inconsistency that makes an API feel unfinished.

### Filtering on multiple values

```bash
curl "https://api.example.com/books?genre=scifi,fantasy"
```

Interpreted as "genre is scifi OR fantasy." Some APIs instead accept the
parameter repeated (`?genre=scifi&genre=fantasy`); both are common — a
comma-separated list is easier to read in logs and simpler to build client
side, while repeated params map more naturally onto some server frameworks'
native query-parsing.

## Sorting

A `sort` (or `order_by`) parameter, with a `-` prefix or a separate `order`
parameter for direction:

```bash
curl "https://api.example.com/books?sort=-published_date,title"
```

This means "sort by `published_date` descending, then by `title` ascending
as a tiebreaker." Compound sort keys matter for stable pagination: sorting
by `published_date` alone, when two books share a date, gives the database
freedom to order them arbitrarily between requests — which breaks
cursor-based pagination built on that sort. Always include a unique
tiebreaker column (commonly `id`) as the final sort key.

```bash
curl "https://api.example.com/books?sort=published_date&order=desc"
```

is the equivalent two-parameter style, more explicit but clunkier once you
need multi-field sorts.

## Combining filtering, sorting, and pagination

```bash
curl "https://api.example.com/books?genre=scifi&price_lte=25&sort=-published_date&limit=10"
```

```json
{
  "data": [ "... up to 10 scifi books, price <= 25, newest first ..." ],
  "meta": { "limit": 10, "total": 34, "next_cursor": "eyJpZCI6NTV9" }
}
```

Note `meta.total` here reflects the *filtered* count (34 matching books),
not the whole table — a frequent source of bugs when filtering and counting
are implemented against different queries.

## Validating filter/sort input

Never pass a client-supplied field name straight into a SQL `ORDER BY`
clause — that's a direct SQL-injection vector if the value isn't
parameterized, and even parameterized queries can't bind column names
(only values). Maintain an allowlist:

```python
ALLOWED_SORT_FIELDS = {"published_date", "title", "price"}

def parse_sort(raw: str):
    field = raw.lstrip("-")
    if field not in ALLOWED_SORT_FIELDS:
        raise BadRequest(f"Cannot sort by '{field}'")
    direction = "DESC" if raw.startswith("-") else "ASC"
    return field, direction
```

Return `400 Bad Request` with a clear error body for an unsupported filter
field or sort field, rather than silently ignoring it — a client silently
getting unfiltered results because it mistyped `pubished_after` is a much
worse failure mode than a loud 400.

```json
{
  "error": {
    "code": "invalid_sort_field",
    "message": "Cannot sort by 'pubished_after'. Allowed: published_date, title, price."
  }
}
```

## Worked example: a search-like filter endpoint

Design `GET /books` to support genre filtering, a price range, free-text
search on title, and sorting — all combinable:

```bash
curl "https://api.example.com/books?q=dune&genre=scifi&price_gte=5&price_lte=20&sort=-rating&limit=20"
```

```json
{
  "data": [
    { "id": 12, "title": "Dune", "genre": "scifi", "price": 14.99, "rating": 4.8 }
  ],
  "meta": { "total": 1, "limit": 20, "filters_applied": { "q": "dune", "genre": "scifi", "price_gte": 5, "price_lte": 20 } }
}
```

Echoing `filters_applied` in the metadata is a small but valuable touch — it
tells the client exactly how the server interpreted its query, which helps
catch silent typos or unsupported combinations during debugging.

## Exercise

1. Design query parameters for `/orders` supporting: filter by `status`
   (one of several values), filter by a date range on `created_at`, and
   sort by `total` or `created_at` in either direction. Write two example
   `curl` calls.
2. Why must a compound sort always end in a unique field when the API also
   supports cursor pagination? Give a concrete scenario where omitting it
   causes a client-visible bug.
3. A client passes `?sort=password_hash` to try to leak data via error
   messages or timing. What should the server do, and why is an allowlist
   safer than a blocklist here?
4. Should `meta.total` reflect the total across the whole table or just the
   rows matching the current filters? Justify your answer from the
   client's point of view.
