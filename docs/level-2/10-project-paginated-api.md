# 10 · Project — Paginated, Cacheable API

Bring together every Level 2 module into one coherent design: extend the
Level 1 Bookshelf API (`/v1/books`, `/v1/shelves`) with pagination,
filtering/sorting, caching, idempotent writes, rate limiting, content
negotiation, bulk import, and an OpenAPI spec describing all of it.

## 1. Pagination (module 1)

```bash
curl "https://api.example.com/v1/books?limit=20&cursor=eyJpZCI6NDB9" \
  -H "Authorization: Bearer $TOKEN"
```

```json
{
  "data": [ "... up to 20 books ..." ],
  "meta": { "limit": 20, "next_cursor": "eyJpZCI6NjB9", "has_more": true, "total": 187 }
}
```

Cursor-based, because the collection grows continuously as users add books
— matches the reasoning from module 1's worked example.

## 2. Filtering & sorting (module 2)

```bash
curl "https://api.example.com/v1/books?author=Frank+Herbert&published_after=1960&sort=-published_year&limit=20" \
  -H "Authorization: Bearer $TOKEN"
```

```json
{
  "data": [
    { "id": 101, "title": "Dune", "author": "Frank Herbert", "published_year": 1965 }
  ],
  "meta": { "limit": 20, "total": 1, "filters_applied": { "author": "Frank Herbert", "published_after": 1960 } }
}
```

`sort` and every filter field are validated against an allowlist server
side; an unrecognized field returns `400` with `invalid_sort_field` or
`invalid_filter_field`.

## 3. HATEOAS-lite links (module 3)

```json
{
  "id": 101,
  "title": "Dune",
  "links": {
    "self": "/v1/books/101",
    "shelves": "/v1/books/101/shelves"
  }
}
```

## 4. Caching (module 7)

```bash
curl -i https://api.example.com/v1/books/101 -H "Authorization: Bearer $TOKEN"
```

```http
HTTP/1.1 200 OK
Cache-Control: private, max-age=60
ETag: "b7e2f9"
```

```bash
curl -i https://api.example.com/v1/books/101 \
  -H "Authorization: Bearer $TOKEN" \
  -H 'If-None-Match: "b7e2f9"'
```

```http
HTTP/1.1 304 Not Modified
ETag: "b7e2f9"
```

Writes use `If-Match` for optimistic concurrency:

```bash
curl -X PUT https://api.example.com/v1/books/101 \
  -H "Authorization: Bearer $TOKEN" \
  -H 'If-Match: "b7e2f9"' \
  -d '{"title": "Dune", "author": "Frank Herbert", "published_year": 1965}'
```

## 5. Idempotent creation (module 5)

```bash
curl -X POST https://api.example.com/v1/books \
  -H "Authorization: Bearer $TOKEN" \
  -H "Idempotency-Key: add-dune-8f14e45f" \
  -d '{"title": "Dune", "author": "Frank Herbert", "isbn": "9780441013593"}'
```

A retried request with the same key returns the original `201` response
without creating a duplicate book.

## 6. Rate limiting (module 6)

```http
HTTP/1.1 200 OK
RateLimit-Limit: 1000
RateLimit-Remaining: 998
RateLimit-Reset: 45
```

```bash
curl -i https://api.example.com/v1/books -H "Authorization: Bearer $TOKEN"
```

```http
HTTP/1.1 429 Too Many Requests
Retry-After: 45

{ "error": { "code": "rate_limited", "message": "Rate limit exceeded. Retry after 45 seconds." } }
```

Keyed per bearer token, not per IP, since every client authenticates.

## 7. Content negotiation (module 8)

```bash
curl https://api.example.com/v1/books -H "Accept: text/csv" -H "Authorization: Bearer $TOKEN"
```

```http
HTTP/1.1 200 OK
Content-Type: text/csv

id,title,author,published_year
101,Dune,Frank Herbert,1965
```

`Accept: application/json` (or no `Accept` header) remains the default.

## 8. Bulk import (module 9)

```bash
curl -X POST https://api.example.com/v1/books/bulk \
  -H "Authorization: Bearer $TOKEN" \
  -d '{"items": [{"title": "Dune"}, {"title": ""}]}'
```

```http
HTTP/1.1 207 Multi-Status
```

```json
{
  "results": [
    { "status": 201, "data": { "id": 102, "title": "Dune" } },
    { "status": 422, "error": { "code": "validation_failed", "message": "title must not be empty" } }
  ]
}
```

## 9. OpenAPI spec (module 4)

```yaml
openapi: 3.0.3
info:
  title: Bookshelf API
  version: 1.1.0
paths:
  /v1/books:
    get:
      parameters:
        - { name: limit, in: query, schema: { type: integer, default: 20 } }
        - { name: cursor, in: query, schema: { type: string } }
        - { name: author, in: query, schema: { type: string } }
        - { name: sort, in: query, schema: { type: string, enum: [published_year, -published_year, title, -title] } }
      responses:
        '200':
          description: A paginated, filtered list of books
          headers:
            RateLimit-Remaining: { schema: { type: integer } }
  /v1/books/bulk:
    post:
      responses:
        '207':
          description: Per-item bulk creation results
security:
  - bearerAuth: []
```

## Deliverable checklist

- [ ] `GET /v1/books` supports `limit`/`cursor` pagination, `author` and
      `published_after` filters, and `sort`, with an allowlist rejecting
      unknown fields as `400`.
- [ ] `GET /v1/books/{id}` returns `ETag` + `Cache-Control`, honors
      `If-None-Match` with `304`, and honors `If-Match` on `PUT` with `412`
      on mismatch.
- [ ] `POST /v1/books` accepts an `Idempotency-Key` and replays the
      original response on a repeated key.
- [ ] Every response includes `RateLimit-*` headers, and exceeding the
      limit returns `429` with `Retry-After`.
- [ ] `GET /v1/books` honors `Accept: text/csv` as an alternative to JSON.
- [ ] `POST /v1/books/bulk` accepts up to 100 items and returns `207` with
      per-item status.
- [ ] An `openapi.yaml` documents all of the above, including the new
      headers and the `207` bulk response.

## Exercise

Extend the design with a `GET /v1/shelves/{id}/books` endpoint that
supports the same pagination and filtering as `/v1/books`, and add
`ETag`-based caching to it. Write its full OpenAPI `paths` entry and one
worked `curl` example showing a `304` revalidation.
