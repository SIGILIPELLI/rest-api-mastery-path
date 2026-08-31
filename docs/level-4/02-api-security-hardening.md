# 02 · API Security Hardening (CORS, Injection, OWASP API Top 10)

Getting auth working (module 1, Level 3) is the start, not the finish.
This module covers the specific ways APIs get exploited in production
and how to close each one.

## CORS: controlling which browsers can call your API

Without CORS headers, a browser blocks JavaScript on `evil.com` from
reading responses from `api.example.com` — but a misconfigured API can
turn that protection off entirely.

**Dangerous** — reflects any origin, effectively disabling the
protection:

```http
Access-Control-Allow-Origin: *
Access-Control-Allow-Credentials: true
```

(Browsers actually reject this exact combination, but teams that
dynamically reflect the request's `Origin` header back recreate the
same hole.)

**Correct** — an explicit allowlist:

```http
Access-Control-Allow-Origin: https://app.example.com
Access-Control-Allow-Credentials: true
Vary: Origin
```

```bash
curl -i -X OPTIONS https://api.example.com/v1/orders \
  -H "Origin: https://evil.com" \
  -H "Access-Control-Request-Method: POST"
```

```http
HTTP/1.1 403 Forbidden
```

The preflight is rejected because `evil.com` isn't in the allowlist —
the browser never even sends the real request.

## Injection: never trust request data

```bash
curl "https://api.example.com/v1/orders?sort=id;DROP+TABLE+orders"
```

If `sort` is interpolated directly into SQL, this is catastrophic.
Always use parameterized queries and, per module 2 (Level 2), validate
`sort` against an allowlist of known-safe column names — never pass
user input into a query string directly, even through an ORM's raw
escape hatches.

```python
# vulnerable
db.execute(f"SELECT * FROM orders ORDER BY {sort_param}")

# safe
ALLOWED_SORTS = {"id", "-id", "created_at", "-created_at"}
if sort_param not in ALLOWED_SORTS:
    raise ValidationError("invalid_sort_field")
db.execute("SELECT * FROM orders ORDER BY " + SAFE_SORT_MAP[sort_param])
```

## OWASP API Security Top 10 (selected)

**Broken Object Level Authorization (BOLA)** — the single most common
real-world API vulnerability: checking that a token is valid, but not
that *this* user owns *this specific* resource.

```bash
curl https://api.example.com/v1/orders/999 \
  -H "Authorization: Bearer $USER_A_TOKEN"
```

If order `999` belongs to a different user and the API returns it
anyway because it only checked "is this token valid," that's BOLA. The
fix: every resource lookup must filter by the authenticated user's
identity too, not just its own ID:

```python
order = db.orders.find_one(id=999, owner_id=current_user.id)
if order is None:
    return 404  # not "belongs to someone else", just not found — don't leak existence
```

**Excessive Data Exposure** — returning a full internal object (with a
password hash, internal notes, cost basis) and relying on the client to
ignore fields it doesn't display. Fix: serialize an explicit response
schema, never `return jsonify(user.__dict__)`.

**Mass Assignment** — accepting a full JSON body and applying it
directly to a database model, letting a client set fields it shouldn't:

```bash
curl -X PATCH https://api.example.com/v1/users/42 \
  -H "Authorization: Bearer $TOKEN" \
  -d '{"name": "Ada", "is_admin": true}'
```

If the handler blindly does `user.update(**request.json)`, a client
just made themselves an admin. Fix: an explicit allowlist of
client-settable fields per endpoint.

**Security misconfiguration** — verbose stack traces in error
responses, debug mode left on in production, default credentials left
active. A `500` response should never include a stack trace to an
external client.

## Rate limiting as a security control, not just fairness

Beyond the throughput fairness covered in Level 2 module 9, aggressive
per-IP and per-account limits on auth endpoints specifically defend
against credential stuffing:

```http
POST /v1/login HTTP/1.1
...
HTTP/1.1 429 Too Many Requests
Retry-After: 60
```

## Worked example: fixing a BOLA vulnerability

A security review finds `GET /v1/invoices/{id}` returns any invoice by
ID to any authenticated user, not just their own.

1. Confirm: `curl .../v1/invoices/17 -H "Authorization: Bearer $OTHER_USER_TOKEN"`
   returns 200 with someone else's invoice.
2. Fix the query to filter by `owner_id = current_user.id`.
3. Return `404`, not `403`, for another user's invoice ID — `403` would
   confirm the ID exists, leaking information.
4. Add a regression test asserting this specifically, and an automated
   BOLA scan (varying IDs across authenticated sessions) to CI.

## Exercise

1. Why is reflecting any `Origin` back with `Access-Control-Allow-
   Credentials: true` dangerous even though it looks similar to a valid
   allowlist entry?
2. Explain why a `404` is the correct response for another user's
   resource, rather than a `403`.
3. Design the fix for a mass-assignment vulnerability on `POST
   /v1/orders` where a client can currently set `status: "shipped"`
   directly on creation.
4. What's the difference between validating that a token is valid and
   validating that a token's owner is authorized for a *specific*
   resource? Give an example of an endpoint vulnerable to only the
   first check.
