# 08 · Content Negotiation

Content negotiation lets a client and server agree on the *format* of a
response (or request) — JSON vs XML vs CSV, English vs Spanish, gzip vs
plain — using standard HTTP headers instead of separate URLs per format.

## The Accept header

The client states, in order of preference, what representations it can
handle:

```bash
curl https://api.example.com/books/42 -H "Accept: application/json"
```

```http
HTTP/1.1 200 OK
Content-Type: application/json

{"id": 42, "title": "Dune"}
```

```bash
curl https://api.example.com/books/42 -H "Accept: application/xml"
```

```http
HTTP/1.1 200 OK
Content-Type: application/xml

<book><id>42</id><title>Dune</title></book>
```

Quality values (`q`) express weighted preference when multiple types are
acceptable:

```http
Accept: application/json;q=0.9, application/xml;q=0.5, */*;q=0.1
```

Read as: "prefer JSON, XML is an acceptable fallback, and I'll take
anything as a last resort." The server picks the highest-`q` type it can
actually produce.

## 406 Not Acceptable

If the server truly cannot produce any format the client will accept, it
should say so explicitly rather than guessing:

```bash
curl https://api.example.com/books/42 -H "Accept: application/vnd.ms-excel"
```

```http
HTTP/1.1 406 Not Acceptable
Content-Type: application/json

{
  "error": {
    "code": "not_acceptable",
    "message": "Supported formats: application/json, application/xml"
  }
}
```

In practice, most JSON-only APIs skip strict 406 handling and just always
return JSON regardless of `Accept` — a pragmatic simplification that's fine
as long as it's documented, but a spec-following API (especially one also
serving XML/CSV consumers) should honor `Accept` properly.

## Content negotiation on the request side: Content-Type

While `Accept` negotiates the *response* format, `Content-Type` on the
request tells the server how to parse the *request body* the client is
sending:

```bash
curl -X POST https://api.example.com/books \
  -H "Content-Type: application/json" \
  -d '{"title": "Dune"}'
```

```bash
curl -X POST https://api.example.com/books \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d 'title=Dune'
```

A server receiving an unsupported `Content-Type` should respond `415
Unsupported Media Type`:

```http
HTTP/1.1 415 Unsupported Media Type
Content-Type: application/json

{
  "error": { "code": "unsupported_media_type", "message": "Expected application/json" }
}
```

## Language negotiation

The same mechanism extends to localization via `Accept-Language`:

```bash
curl https://api.example.com/books/42 -H "Accept-Language: es-ES, es;q=0.8, en;q=0.5"
```

```http
HTTP/1.1 200 OK
Content-Language: es-ES

{"id": 42, "title": "Dune", "description": "Una novela de ciencia ficción..."}
```

Responses that vary by a negotiated header should include a `Vary` header
so intermediate caches store separate copies per variant instead of
serving a Spanish response to an English-requesting client:

```http
Vary: Accept-Language, Accept
```

## Versioning via Accept (media-type versioning)

An alternative to URL-path versioning (`/v1/books`, covered in Level 1) is
encoding the version inside a custom media type:

```bash
curl https://api.example.com/books/42 -H "Accept: application/vnd.example.v2+json"
```

```http
HTTP/1.1 200 OK
Content-Type: application/vnd.example.v2+json

{"id": 42, "title": "Dune", "authors": [{"id": 3, "name": "Frank Herbert"}]}
```

This is GitHub's approach for some of its API previews
(`application/vnd.github.v3+json`). It keeps the URL itself
version-agnostic, at the cost of being less discoverable than a version
visible directly in the path — most developers find `/v2/books` easier to
notice and reason about than a version buried in a header.

## Worked example: an API supporting JSON and CSV export

```bash
curl "https://api.example.com/reports/sales" -H "Accept: application/json"
```

```json
{ "data": [ { "date": "2026-08-01", "total": 4200.50 } ] }
```

```bash
curl "https://api.example.com/reports/sales" -H "Accept: text/csv"
```

```http
HTTP/1.1 200 OK
Content-Type: text/csv
Content-Disposition: attachment; filename="sales-report.csv"

date,total
2026-08-01,4200.50
```

Server-side dispatch:

```python
def get_report(request):
    data = build_report()
    accept = request.headers.get("Accept", "application/json")

    if "text/csv" in accept:
        return to_csv(data), "text/csv"
    if "application/json" in accept or "*/*" in accept:
        return to_json(data), "application/json"

    raise NotAcceptable(supported=["application/json", "text/csv"])
```

## Exercise

1. A client sends `Accept: application/xml;q=1.0, application/json;q=0.5`
   to a server that only supports JSON. What should the response be:
   `200` with JSON, or `406`? Justify both a strict and a pragmatic answer.
2. Why does a response that varies by `Accept-Language` need a `Vary`
   header for correctness under a shared CDN cache? Describe the bug that
   happens without it.
3. Compare header-based (`Accept: application/vnd.example.v2+json`) vs
   URL-based (`/v2/books`) versioning on: discoverability, cache-key
   simplicity, and ease of routing in a reverse proxy.
4. Design the `Content-Type` handling for an endpoint that must accept both
   JSON bodies and `multipart/form-data` (e.g. for a book cover image
   upload alongside metadata).
