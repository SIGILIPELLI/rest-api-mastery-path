# 07 · Caching Headers (ETag, Cache-Control)

Level 1 introduced `Cache-Control` as one of REST's core constraints
(cacheability). This module covers the actual headers that implement it —
`Cache-Control`, `ETag`, and conditional requests — and how they combine to
avoid re-sending data the client already has.

## Cache-Control directives

```http
Cache-Control: public, max-age=3600
```

| Directive | Meaning |
|---|---|
| `public` | Any cache (browser, CDN, proxy) may store this response |
| `private` | Only the end client may cache it, not a shared proxy/CDN (typical for per-user data) |
| `no-cache` | May be cached, but **must** be revalidated with the server before reuse (misleadingly named — it doesn't mean "don't cache") |
| `no-store` | Must not be cached anywhere, ever — for sensitive data |
| `max-age=N` | Fresh for N seconds; safe to reuse without contacting the server at all during that window |
| `must-revalidate` | Once stale, must revalidate before use, even if the client would otherwise serve a stale copy |

```bash
curl -i https://api.example.com/books/42
```

```http
HTTP/1.1 200 OK
Cache-Control: private, max-age=60
Content-Type: application/json

{"id": 42, "title": "Dune"}
```

A client can reuse this response for 60 seconds without another network
call. After that, it should revalidate.

## ETags and conditional GET

An `ETag` is an opaque fingerprint of a resource's current state (often a
hash of its content). It lets a client ask "has this changed since I last
saw ETag X?" instead of re-downloading the whole thing.

```bash
curl -i https://api.example.com/books/42
```

```http
HTTP/1.1 200 OK
ETag: "a1b2c3d4"
Cache-Control: private, max-age=60
Content-Type: application/json

{"id": 42, "title": "Dune", "price": 15.99}
```

Once the cached copy goes stale, the client revalidates with
`If-None-Match`:

```bash
curl -i https://api.example.com/books/42 -H 'If-None-Match: "a1b2c3d4"'
```

If unchanged:

```http
HTTP/1.1 304 Not Modified
ETag: "a1b2c3d4"
Cache-Control: private, max-age=60
```

`304 Not Modified` has **no body** — the client is told to keep using its
cached copy, saving the full payload transfer. If the resource *has*
changed, the server responds normally with `200` and the new body and
`ETag`.

## Last-Modified as a lighter alternative

```http
HTTP/1.1 200 OK
Last-Modified: Wed, 12 Aug 2026 14:23:00 GMT
```

```bash
curl -i https://api.example.com/books/42 -H 'If-Modified-Since: Wed, 12 Aug 2026 14:23:00 GMT'
```

`Last-Modified`/`If-Modified-Since` is coarser (second-level precision,
relies on accurate clocks/timestamps) than an `ETag`, but cheaper to
compute when a resource already has a reliable `updated_at` column — no
hashing required.

## ETags for optimistic concurrency control

`ETag` doubles as a concurrency mechanism via `If-Match` on writes: "only
apply this update if the resource still matches the version I last read."

```bash
curl -X PUT https://api.example.com/books/42 \
  -H 'If-Match: "a1b2c3d4"' \
  -d '{"title": "Dune", "price": 12.99}'
```

If another client updated the book in the meantime (ETag changed), the
server rejects with `412 Precondition Failed` instead of silently
overwriting the newer change — a lost-update bug in the making otherwise.

```http
HTTP/1.1 412 Precondition Failed
Content-Type: application/json

{
  "error": {
    "code": "precondition_failed",
    "message": "Resource has changed since it was last fetched. Refetch and retry."
  }
}
```

This is the same pattern as optimistic locking with a `version` column in
a database, expressed over HTTP.

## Combining Cache-Control and ETag

```http
HTTP/1.1 200 OK
Cache-Control: private, max-age=60, must-revalidate
ETag: "a1b2c3d4"
```

Read together: "cache me for 60 seconds without asking; after that, don't
use a stale copy — ask again, but if my content is unchanged, just tell me
`304` instead of resending everything." This combination gets both speed
(no network round trip inside the freshness window) and correctness (no
stale-forever reuse) without over-fetching unchanged data.

## Worked example: a cache-aware client

```python
import requests

def get_book(id, cache):
    entry = cache.get(id)
    headers = {}
    if entry and "etag" in entry:
        headers["If-None-Match"] = entry["etag"]

    resp = requests.get(f"https://api.example.com/books/{id}", headers=headers)

    if resp.status_code == 304:
        return entry["body"]          # unchanged, use cached copy

    body = resp.json()
    cache[id] = {"body": body, "etag": resp.headers.get("ETag")}
    return body
```

This client transfers the full book payload only on the first fetch and any
fetch after a real change — every unchanged revalidation costs a small
request/`304` round trip instead of the full body.

## Exercise

1. Explain the practical difference between `no-cache` and `no-store`, and
   give one type of API response where each would be the right choice.
2. Two clients `GET` the same book, both cache its `ETag`. Client A `PUT`s
   an update. What should happen when Client B then tries to `PUT` its own
   (now stale) update using `If-Match` with its old `ETag`?
3. Why does `304 Not Modified` have no response body, and why does that
   matter for a large resource (e.g. a 2MB report)?
4. When would `Last-Modified`/`If-Modified-Since` be preferable to a
   computed `ETag`, and when would it be insufficient?
