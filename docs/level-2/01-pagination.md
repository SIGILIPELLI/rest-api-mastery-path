# 01 · Pagination

Once a collection resource grows past a handful of items, returning "all of
them" in one response stops being viable — slow queries, huge payloads, and
a client that can't reasonably render 50,000 rows anyway. Pagination is how
an API hands back a collection in manageable, navigable chunks.

## Offset/limit pagination

The simplest scheme: the client says how many items it wants and where to
start.

```bash
curl "https://api.example.com/books?limit=20&offset=40"
```

```json
{
  "data": [ { "id": 41, "title": "..." }, "... 19 more ..." ],
  "meta": { "limit": 20, "offset": 40, "total": 187 }
}
```

**Pros**: simple to implement (maps directly to SQL `LIMIT`/`OFFSET`), lets
a client jump to an arbitrary page (`offset=100`).

**Cons**: two real problems at scale:

- **Performance** — `OFFSET 100000` still requires the database to scan and
  discard the first 100,000 rows before returning the next page; this gets
  slower as offset grows.
- **Consistency** — if a row is inserted or deleted between two page
  requests, offsets shift and a client can see the same item twice or skip
  one entirely ("page drift").

## Cursor-based (keyset) pagination

Instead of a numeric offset, the server returns an opaque **cursor**
pointing at the last item the client saw; the next request asks for items
after that cursor.

```bash
curl "https://api.example.com/books?limit=20"
```

```json
{
  "data": [ "... 20 items ..." ],
  "meta": { "next_cursor": "eyJpZCI6NjB9", "has_more": true }
}
```

```bash
curl "https://api.example.com/books?limit=20&cursor=eyJpZCI6NjB9"
```

The cursor is typically a base64-encoded pointer to the sort key of the last
row returned (e.g. `{"id": 60}` in the example above) — the server decodes
it and queries `WHERE id > 60 ORDER BY id LIMIT 20`, which uses an index and
stays fast regardless of how deep the client pages.

**Pros**: consistent under concurrent inserts/deletes (no drift), fast at
any depth since it's an indexed range query, not a scan-and-discard.

**Cons**: no jumping to "page 5" directly — only forward/backward
traversal from a cursor; slightly more complex to implement and to encode
opaque, tamper-resistant cursors.

## Page-number pagination

A friendlier variant of offset/limit, common in UI-facing APIs:

```bash
curl "https://api.example.com/books?page=3&per_page=20"
```

This is offset/limit in disguise (`offset = (page - 1) * per_page`) and
carries the same tradeoffs, just with a more human-readable query shape.

## Choosing a strategy

| | Offset/limit | Page number | Cursor |
|---|---|---|---|
| Jump to arbitrary page | Yes | Yes | No |
| Stable under inserts/deletes | No | No | Yes |
| Fast at large depth | No | No | Yes |
| Implementation complexity | Low | Low | Medium |
| Good for | Small/static datasets, admin UIs | User-facing paged UIs | Infinite scroll, feeds, large/live datasets |

Real-world example: GitHub's REST API uses page-number pagination for most
list endpoints but has moved high-volume, frequently-mutated endpoints to
cursor-based pagination. Twitter/X and Stripe both use cursor pagination
(`since_id`/`starting_after`) for their main feeds and list endpoints,
specifically because their data changes constantly under concurrent
traffic.

## Communicating pagination metadata

Two common patterns for surfacing "is there more, and how do I get it":

**1. Response body metadata** (used above) — a `meta` object with `total`,
`limit`/`offset` or `next_cursor`, `has_more`.

**2. `Link` header** (RFC 8288), used by GitHub's API:

```http
HTTP/1.1 200 OK
Link: <https://api.example.com/books?page=4&per_page=20>; rel="next",
      <https://api.example.com/books?page=1&per_page=20>; rel="first",
      <https://api.example.com/books?page=10&per_page=20>; rel="last"
```

The `Link` header keeps the response body focused purely on data and lets a
generic HTTP client (or curl `--next` style tooling) follow pagination
without parsing a custom body shape.

## Worked example: designing pagination for `/books`

Requirements: a public API, dataset grows continuously (new books added
daily), and a mobile client mostly does infinite-scroll (never "jump to
page 7"). Given those constraints, cursor pagination is the right call —
the access pattern is purely sequential, and the dataset's ongoing growth
means offset drift would be a real, user-visible bug (duplicate or skipped
books while scrolling).

```
GET /books?limit=25
→ { "data": [...], "meta": { "next_cursor": "...", "has_more": true } }

GET /books?limit=25&cursor=<next_cursor>
→ { "data": [...], "meta": { "next_cursor": "...", "has_more": true } }
```

## Exercise

1. An admin dashboard needs a "jump to page 12" control alongside "50 items
   per page." Which pagination strategy fits, and why does cursor
   pagination not support this well?
2. A client pages through `/orders` using `offset`/`limit` while new orders
   are being created concurrently. Walk through a concrete sequence of
   events that causes the client to see the same order twice.
3. Design the cursor payload (as JSON, before base64 encoding) for a
   `/books` endpoint sorted by `created_at DESC, id DESC` (a compound sort
   key, needed because `created_at` alone isn't unique).
4. Why is a `Link` header response (GitHub-style) arguably a "more RESTful"
   way to expose pagination than putting `next_page` inside the JSON body?
