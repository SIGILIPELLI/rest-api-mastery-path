# 08 · Error Handling Conventions

A good error response tells the client (a human debugging, or code deciding
whether to retry) exactly what went wrong and, ideally, how to fix it. This
module covers the conventions that make error responses genuinely useful.

## The three ingredients of a good error response

1. **The right HTTP status code** (covered in module 2) — this is what
   generic tooling and monitoring systems check first.
2. **A consistent, structured error body** — so client code can parse errors
   programmatically, not just display them.
3. **A human-readable message** — for developers debugging, and sometimes
   surfaced directly to end users.

## A well-structured error body

There's no single universal standard, but a solid, widely-used shape looks
like this:

```json
{
  "error": {
    "code": "validation_failed",
    "message": "The request could not be processed due to validation errors.",
    "details": [
      { "field": "email", "issue": "must be a valid email address" },
      { "field": "age", "issue": "must be a positive integer" }
    ]
  }
}
```

```http
HTTP/1.1 422 Unprocessable Entity
Content-Type: application/json
```

Key properties of this shape:

- `error.code` is a **stable, machine-readable string** (`validation_failed`,
  not a number or a sentence) — client code can `switch` on this without
  parsing English text, and it won't change if you tweak the message
  wording later.
- `error.message` is a **human-readable summary** — safe to log or show to a
  developer, generally not shown verbatim to end users (it may leak
  implementation details).
- `error.details` is an **array** allowing multiple simultaneous validation
  failures to be reported in one response, rather than forcing the client
  to fix one field, resubmit, discover the next error, fix, resubmit again.

## RFC 9457 — Problem Details for HTTP APIs

There is an actual IETF standard for error responses worth knowing:
[RFC 9457](https://www.rfc-editor.org/rfc/rfc9457) ("Problem Details for
HTTP APIs," obsoleting the earlier RFC 7807), using
`Content-Type: application/problem+json`:

```json
{
  "type": "https://api.example.com/errors/validation-failed",
  "title": "Your request parameters didn't validate.",
  "status": 422,
  "detail": "The 'email' field must be a valid email address.",
  "instance": "/orders/12345"
}
```

- `type` — a URI identifying the error type (doesn't have to be a real,
  dereferenceable page, but often is, serving as living documentation).
- `title` — a short, human-readable summary that shouldn't change between
  occurrences of the same problem `type`.
- `status` — the HTTP status code, duplicated in the body for convenience.
- `detail` — specifics about this particular occurrence.
- `instance` — a URI identifying this specific occurrence (e.g. which
  resource/request triggered it).

Whether you use this exact standard or a custom shape like the first
example, the important thing is **consistency across your entire API** — a
client should never have to guess the shape of an error body per-endpoint.

## Common mistakes

### Mistake 1: inconsistent shapes across endpoints

```json
// /users endpoint
{ "error": "Not found" }

// /orders endpoint
{ "message": "Resource does not exist", "code": 404 }
```

Pick one shape and apply it everywhere. Clients writing generic error-handling
code (e.g. a single interceptor that shows a toast notification) shouldn't
need per-endpoint special cases.

### Mistake 2: leaking internals

```json
{
  "error": "NullPointerException at com.example.orders.OrderService.java:142"
}
```

Stack traces and internal class/file names belong in server-side logs, not
client-facing responses — they leak implementation details (a security
concern) and aren't actionable for the API consumer anyway. Log the full
detail server-side, keyed to a request ID; return the client a clean message
plus that ID for support correlation.

```json
{
  "error": {
    "code": "internal_error",
    "message": "Something went wrong on our end. Please try again or contact support.",
    "request_id": "req_9f8a7c2e"
  }
}
```

### Mistake 3: always returning `200 OK`

Covered in module 2 — never let the body say "error" while the status code
says "success." Tooling that checks status codes (and there's a lot of it —
caches, monitoring, generic SDK error handling) will be wrong every time.

### Mistake 4: not distinguishing retryable from non-retryable errors

A `429 Too Many Requests` or `503 Service Unavailable` tells a client
"retry later, possibly with backoff." A `400 Bad Request` or `422
Unprocessable Entity` tells a client "don't retry until you fix the
request." Client libraries that auto-retry on any non-2xx will hammer a
server pointlessly on a `400` — make sure your status codes correctly signal
which category an error falls in, and consider a `Retry-After` header on
`429`/`503`:

```http
HTTP/1.1 429 Too Many Requests
Retry-After: 30
Content-Type: application/json

{ "error": { "code": "rate_limited", "message": "Too many requests. Retry after 30 seconds." } }
```

## Validation errors: report everything at once

```bash
curl -i -X POST https://api.example.com/users \
  -H "Content-Type: application/json" \
  -d '{"email": "not-an-email", "age": -5}'
```

```http
HTTP/1.1 422 Unprocessable Entity
Content-Type: application/json

{
  "error": {
    "code": "validation_failed",
    "message": "The request could not be processed due to validation errors.",
    "details": [
      { "field": "email", "issue": "must be a valid email address" },
      { "field": "age", "issue": "must be a positive integer" }
    ]
  }
}
```

Both problems are reported together — the client fixes both fields and
resubmits once, rather than a frustrating fix-one-resubmit-repeat loop.

## Exercise

1. Design a consistent JSON error shape (your own, or RFC 9457) for an API
   you're building, and show it applied to three different scenarios: a
   `404 Not Found`, a `422` validation error with two problems, and a
   `500 Internal Server Error`.
2. A client reports "my API integration randomly fails and retries forever."
   You inspect logs and find the API was returning `400 Bad Request` for a
   permanently malformed request, and the client's SDK retries on any
   non-2xx status. Whose bug is this, and what status code convention would
   have prevented the infinite retry?
3. Rewrite this bad error response to follow the conventions in this module:
   `{"err": "cant create, dup email"}` with `HTTP/1.1 200 OK`.
