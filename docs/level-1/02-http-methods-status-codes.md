# 02 · HTTP Methods & Status Codes

<span class="level-badge">Level 1 · Entry</span>

REST APIs run on top of HTTP, so understanding HTTP methods (verbs) and
status codes precisely is non-negotiable. Get these right and most of your
API's behavior becomes predictable to anyone who has used another REST API
before.

## The core HTTP methods

| Method | Purpose | Safe? | Idempotent? | Has a body? |
|---|---|---|---|---|
| `GET` | Retrieve a resource | Yes | Yes | No (by convention) |
| `POST` | Create a resource / trigger an action | No | No | Yes |
| `PUT` | Replace a resource entirely | No | Yes | Yes |
| `PATCH` | Partially update a resource | No | No* | Yes |
| `DELETE` | Remove a resource | No | Yes | Usually no |
| `HEAD` | Like `GET` but headers only, no body | Yes | Yes | No |
| `OPTIONS` | Discover allowed methods/CORS preflight | Yes | Yes | No |

\* `PATCH` is not guaranteed idempotent by the HTTP spec, though many APIs
implement it idempotently in practice (e.g. "set field X to value Y" is
idempotent; "increment field X by 1" is not).

**Safe** means the method doesn't change server state — calling it never has
side effects, so it's safe for browsers/crawlers to call freely.

**Idempotent** means calling it once has the same effect as calling it many
times. `DELETE /books/17` called five times still ends with book 17 gone —
same end state as calling it once (the response codes on repeats may differ,
e.g. `404` after the first successful delete, but the *resource state* is
identical).

!!! warning "GET must never change state"
    A `GET /books/17/delete` endpoint is a classic anti-pattern — it works,
    but violates the safety guarantee, meaning link prefetchers, crawlers,
    and browser "back" button caching can accidentally trigger deletes.
    Always use `DELETE` for deletions.

## Worked example: full CRUD lifecycle

Assume a `books` resource on `https://api.example.com`.

**Create** (`POST`):

```bash
curl -i -X POST https://api.example.com/books \
  -H "Content-Type: application/json" \
  -d '{"title": "Dune", "author": "Frank Herbert", "year": 1965}'
```

```http
HTTP/1.1 201 Created
Content-Type: application/json
Location: /books/101

{
  "id": 101,
  "title": "Dune",
  "author": "Frank Herbert",
  "year": 1965
}
```

**Read** (`GET`):

```bash
curl -i https://api.example.com/books/101
```

```http
HTTP/1.1 200 OK
Content-Type: application/json

{
  "id": 101,
  "title": "Dune",
  "author": "Frank Herbert",
  "year": 1965
}
```

**Replace** (`PUT` — sends the *entire* resource):

```bash
curl -i -X PUT https://api.example.com/books/101 \
  -H "Content-Type: application/json" \
  -d '{"title": "Dune", "author": "Frank Herbert", "year": 1965, "pages": 412}'
```

```http
HTTP/1.1 200 OK
Content-Type: application/json

{
  "id": 101,
  "title": "Dune",
  "author": "Frank Herbert",
  "year": 1965,
  "pages": 412
}
```

**Partially update** (`PATCH` — sends only changed fields):

```bash
curl -i -X PATCH https://api.example.com/books/101 \
  -H "Content-Type: application/json" \
  -d '{"pages": 420}'
```

```http
HTTP/1.1 200 OK
Content-Type: application/json

{
  "id": 101,
  "title": "Dune",
  "author": "Frank Herbert",
  "year": 1965,
  "pages": 420
}
```

**Delete** (`DELETE`):

```bash
curl -i -X DELETE https://api.example.com/books/101
```

```http
HTTP/1.1 204 No Content
```

!!! note
    These requests/responses are worked examples reasoned through against a
    typical well-designed REST API — they were not executed against a live
    server, since `api.example.com` isn't real. The shapes, headers, and
    status codes shown are exactly what a correctly implemented API returns.

## Status code families

| Range | Meaning | Common codes |
|---|---|---|
| 1xx | Informational | 100 Continue |
| 2xx | Success | 200, 201, 202, 204 |
| 3xx | Redirection | 301, 302, 304 |
| 4xx | Client error | 400, 401, 403, 404, 405, 409, 422, 429 |
| 5xx | Server error | 500, 502, 503, 504 |

### The codes you'll use constantly

- **`200 OK`** — generic success, used for `GET`, `PUT`, `PATCH`, sometimes
  `POST` when not creating a new resource.
- **`201 Created`** — a resource was created; include a `Location` header
  pointing at it.
- **`204 No Content`** — success, but nothing to return in the body (typical
  for `DELETE`, sometimes `PUT`).
- **`400 Bad Request`** — the request itself was malformed (bad JSON, wrong
  types) — the client's fault, unrelated to auth or existence.
- **`401 Unauthorized`** — no valid credentials were provided at all.
- **`403 Forbidden`** — credentials were valid, but the caller doesn't have
  permission for this action.
- **`404 Not Found`** — the resource doesn't exist (or, in some APIs, exists
  but is deliberately hidden from this caller for security reasons).
- **`405 Method Not Allowed`** — the resource exists but doesn't support the
  method you used (e.g. `DELETE /health-check`).
- **`409 Conflict`** — the request conflicts with the resource's current
  state (e.g. creating a user with an email that's already taken).
- **`422 Unprocessable Entity`** — the request was well-formed JSON, but
  failed semantic/business validation (e.g. `year: "not a number"`).
- **`429 Too Many Requests`** — rate limit exceeded (Level 2 covers this).
- **`500 Internal Server Error`** — something broke on the server; not the
  client's fault.
- **`503 Service Unavailable`** — server temporarily can't handle the
  request (overloaded, maintenance).

!!! tip "401 vs 403, the classic mix-up"
    `401` = "I don't know who you are" (missing/invalid token).
    `403` = "I know who you are, and you're not allowed to do this."
    A common test: if the caller sent no `Authorization` header at all, that
    should almost always be `401`, never `403`.

## Exercise

For each scenario below, write down which HTTP method and status code you'd
use, and why:

1. A client sends a `POST /orders` with a JSON body missing the required
   `customer_id` field.
2. A client requests `GET /orders/9999`, and order 9999 doesn't exist.
3. A client with a valid token tries to `DELETE /orders/42`, but order 42
   belongs to a different customer.
4. A client successfully creates a new `comment` on a blog post.
5. A client calls `PATCH /users/7` twice in a row with `{"status": "banned"}`
   — should the second call behave differently from the first? Why or why
   not?
