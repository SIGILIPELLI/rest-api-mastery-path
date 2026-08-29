# 04 · Request/Response Anatomy

Every HTTP request and response is made of the same four parts: a start
line, headers, a blank line, and (optionally) a body. Knowing this structure
cold makes debugging APIs with `curl -v`, browser devtools, or Wireshark
dramatically easier.

## Anatomy of a request

```
POST /books HTTP/1.1                          <- request line: METHOD PATH VERSION
Host: api.example.com                         <- headers...
Content-Type: application/json
Authorization: Bearer eyJhbGciOiJIUzI1NiJ9...
Content-Length: 58
                                               <- blank line separates headers from body
{"title": "Dune", "author": "Frank Herbert"}  <- body
```

- **Request line**: `POST /books HTTP/1.1` — the method, the path (not the
  full URL — the host is a separate header), and the HTTP version.
- **Headers**: key-value metadata about the request. Order mostly doesn't
  matter; header *names* are case-insensitive (`content-type` ==
  `Content-Type`).
- **Blank line**: mandatory — it's what tells the parser "headers are done,
  body starts now."
- **Body**: the payload, present for `POST`/`PUT`/`PATCH`, generally absent
  for `GET`/`DELETE`/`HEAD`.

## Anatomy of a response

```
HTTP/1.1 201 Created                          <- status line: VERSION STATUS_CODE REASON_PHRASE
Content-Type: application/json
Location: /books/101
Content-Length: 76

{"id": 101, "title": "Dune", "author": "Frank Herbert"}
```

- **Status line**: HTTP version, numeric status code, and a human-readable
  reason phrase (`Created` — informational only, clients should check the
  numeric code, not this text).
- **Headers**, **blank line**, **body** — same structure as a request.

## Headers you'll see constantly

| Header | Direction | Purpose |
|--------|-----------|---------|
| `Host` | Request | Which server/domain this request targets (required in HTTP/1.1) |
| `Content-Type` | Both | MIME type of the body, e.g. `application/json`, `application/x-www-form-urlencoded` |
| `Content-Length` | Both | Size of the body in bytes |
| `Accept` | Request | What response formats the client can handle, e.g. `application/json` |
| `Authorization` | Request | Credentials, e.g. `Bearer <token>` or `Basic <base64>` |
| `Location` | Response | URL of a newly created resource (with `201`) or redirect target (with `3xx`) |
| `Cache-Control` | Both | Caching directives (Level 2) |
| `ETag` | Response | A version fingerprint of the resource (Level 2) |
| `User-Agent` | Request | Identifies the calling client/library |
| `X-Request-Id` | Both | A correlation ID for tracing a request through logs (common, non-standard) |

## Content-Type in detail

`Content-Type` tells the receiver how to parse the body. Get it wrong and
the server may reject or misinterpret perfectly valid data.

```
Content-Type: application/json               <- JSON body
Content-Type: application/x-www-form-urlencoded  <- key=value&key2=value2 (classic HTML forms)
Content-Type: multipart/form-data; boundary=...  <- file uploads mixed with fields
Content-Type: text/plain
```

A request whose body is JSON but is missing `Content-Type: application/json`
is a very common source of confusing `400`/`415` errors — many server
frameworks refuse to parse the body as JSON without that header, even if the
bytes are perfectly valid JSON.

## Inspecting real anatomy with curl -v

```bash
curl -v -X POST https://httpbin.org/post \
  -H "Content-Type: application/json" \
  -d '{"title": "Dune"}'
```

`-v` (verbose) prints the request as it's sent and the response as it's
received, each line prefixed so you can tell them apart:

```
> POST /post HTTP/1.1
> Host: httpbin.org
> User-Agent: curl/8.4.0
> Accept: */*
> Content-Type: application/json
> Content-Length: 17
>
< HTTP/1.1 200 OK
< Date: Sat, 29 Aug 2026 10:00:00 GMT
< Content-Type: application/json
< Content-Length: 431
<
{
  "json": { "title": "Dune" },
  "headers": {
    "Content-Type": "application/json",
    "Host": "httpbin.org"
  },
  ...
}
```

`>` lines are what curl sent; `<` lines are what the server sent back. This
example matches httpbin.org's well-documented echo behavior (it wasn't
executed live here, but httpbin's `/post` endpoint reliably echoes the
request back in this shape).

## The JSON body itself

Most modern REST APIs use JSON almost exclusively. A few conventions worth
internalizing:

```json
{
  "id": 101,
  "title": "Dune",
  "year": 1965,
  "in_stock": true,
  "tags": ["sci-fi", "classic"],
  "publisher": {
    "name": "Chilton Books",
    "country": "US"
  },
  "discontinued_at": null
}
```

- Use `null` explicitly for "no value," rather than omitting the field —
  omission is ambiguous between "no value" and "field doesn't exist in this
  version of the schema."
- Prefer `snake_case` or `camelCase` for field names, but pick **one** and
  apply it consistently across the whole API. (`snake_case` is common in
  Python-backed APIs; `camelCase` is common in JavaScript-backed ones.)
- Nest related data (`publisher`) rather than flattening everything
  (`publisher_name`, `publisher_country`) when it represents a genuinely
  structured sub-object.
- Dates: use ISO 8601 (`"2026-08-29T10:00:00Z"`), not ambiguous formats like
  `"08/29/2026"`.

## Exercise

1. Run `curl -v https://httpbin.org/get` (or reason through it if you don't
   have network access) and label each part of the output: request line,
   request headers, response status line, response headers, response body.
2. Write out, as raw HTTP text (request line + headers + blank line + body),
   a `POST` request to `/comments` with a JSON body containing `post_id` and
   `text` fields, including the two headers required for the server to parse
   the JSON body correctly.
3. Explain, in one or two sentences, why sending a JSON body without a
   `Content-Type: application/json` header is a common source of bugs.
