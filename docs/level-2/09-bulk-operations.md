# 09 · Bulk Operations & Batch Requests

Sending one HTTP request per item gets expensive fast — a client needing to
create 500 records pays 500 round trips, 500x TLS/HTTP overhead, and 500
chances to partially fail. Bulk endpoints let a client submit many
operations in a single request.

## Bulk create

```bash
curl -X POST https://api.example.com/books/bulk \
  -H "Content-Type: application/json" \
  -d '{
    "items": [
      {"title": "Dune", "genre": "scifi"},
      {"title": "Foundation", "genre": "scifi"},
      {"title": "", "genre": "scifi"}
    ]
  }'
```

Partial success is the norm for bulk operations — one bad item shouldn't
fail the other 499. Return `207 Multi-Status` with a per-item result:

```json
{
  "results": [
    { "status": 201, "data": { "id": 101, "title": "Dune" } },
    { "status": 201, "data": { "id": 102, "title": "Foundation" } },
    { "status": 422, "error": { "code": "validation_error", "message": "title is required" } }
  ]
}
```

```http
HTTP/1.1 207 Multi-Status
```

`207` (from WebDAV, adopted broadly for bulk REST endpoints) signals "this
request as a whole doesn't have one status — check each item." The client
must inspect `results[i].status` per item rather than trusting one
top-level code.

## Bulk update and delete

```bash
curl -X PATCH https://api.example.com/books/bulk \
  -d '{
    "items": [
      {"id": 101, "genre": "sci-fi"},
      {"id": 999, "genre": "sci-fi"}
    ]
  }'
```

```json
{
  "results": [
    { "status": 200, "data": { "id": 101, "genre": "sci-fi" } },
    { "status": 404, "error": { "code": "not_found", "message": "Book 999 not found" } }
  ]
}
```

```bash
curl -X DELETE https://api.example.com/books/bulk \
  -H "Content-Type: application/json" \
  -d '{"ids": [101, 102, 999]}'
```

## Enforcing sane batch limits

Unbounded batch size is a denial-of-service vector (a client submitting
100,000 items in one request can exhaust server memory or lock a table for
an unacceptable duration). Cap it and reject oversized batches loudly:

```json
{
  "error": {
    "code": "batch_too_large",
    "message": "Maximum 100 items per bulk request; received 500."
  }
}
```

```http
HTTP/1.1 400 Bad Request
```

## Bulk operation atomicity: all-or-nothing vs best-effort

Two valid designs, and the API must document which one it implements:

**Best-effort** (shown above): each item succeeds or fails independently;
partial success is normal and expected.

**All-or-nothing** (transactional): either every item succeeds or none do
— typically implemented as one database transaction, rolled back on any
failure.

```bash
curl -X POST https://api.example.com/transfers/bulk \
  -H "Content-Type: application/json" \
  -d '{"atomic": true, "items": [{"from": "acct_1", "to": "acct_2", "amount": 100}, {"from": "acct_3", "to": "acct_4", "amount": -50}]}'
```

```json
{
  "error": {
    "code": "batch_rolled_back",
    "message": "Item 2 failed validation (amount must be positive); no transfers were applied."
  }
}
```

Financial and inventory operations usually need all-or-nothing (you never
want half of a multi-leg transfer to apply); independent resource creation
(bulk-importing books) usually tolerates best-effort, since each item is
unrelated to the others.

## True HTTP batching: multiple different requests in one call

A more general pattern (used by Google APIs and Microsoft Graph) batches
arbitrary, even different-method, requests together:

```bash
curl -X POST https://graph.microsoft.com/v1.0/$batch \
  -H "Content-Type: application/json" \
  -d '{
    "requests": [
      { "id": "1", "method": "GET", "url": "/me" },
      { "id": "2", "method": "GET", "url": "/me/messages?$top=1" }
    ]
  }'
```

```json
{
  "responses": [
    { "id": "1", "status": 200, "body": { "displayName": "Ada Lovelace" } },
    { "id": "2", "status": 200, "body": { "value": [ { "subject": "Welcome" } ] } }
  ]
}
```

This is heavier to implement (the server effectively becomes a mini HTTP
router operating inside a single request body) but is the most flexible
option, useful for clients that need to fetch several unrelated resources
in one round trip (e.g. a mobile app minimizing cellular round trips on
page load).

## Worked example: designing a CSV-style book import

```bash
curl -X POST https://api.example.com/books/bulk \
  -H "Content-Type: application/json" \
  -d '{
    "items": [
      {"title": "Dune", "isbn": "9780441013593"},
      {"title": "Dune", "isbn": "9780441013593"}
    ]
  }'
```

```json
{
  "results": [
    { "status": 201, "data": { "id": 201, "title": "Dune" } },
    { "status": 409, "error": { "code": "conflict", "message": "ISBN 9780441013593 already exists" } }
  ]
}
```

Design decision: best-effort with per-item `409` for duplicates, since one
duplicate ISBN in a 200-row import shouldn't fail the other 199 valid rows
— exactly the situation bulk best-effort semantics exist for.

## Exercise

1. Design the request/response shape for a bulk endpoint that must be
   all-or-nothing (e.g. splitting a restaurant bill across N people, where
   the amounts must sum exactly to the total). What HTTP status fits a
   full rollback?
2. Why is `207 Multi-Status` a better fit than a single `200` or `400` for
   best-effort bulk operations? What would a client have to guess wrong
   about if the server just returned `200` for the whole batch?
3. What server-side risk does an unbounded bulk-delete endpoint pose, and
   what two mitigations would you put in place?
4. Compare a same-resource bulk endpoint (`POST /books/bulk`) against a
   general multi-request batch endpoint (`POST /$batch`) on: implementation
   complexity, and which client use case each is actually good for.
