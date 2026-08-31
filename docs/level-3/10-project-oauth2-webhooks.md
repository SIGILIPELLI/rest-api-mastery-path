# 10 · Project — OAuth2-Secured API with Webhooks

Bring together every Level 3 module: secure the Bookshelf API with
OAuth2, notify subscribers of changes via webhooks, put it behind a
gateway, document it, test it, and monitor it in production.

## 1. OAuth2 Authorization Code + PKCE (module 1)

```bash
curl -X POST https://auth.example.com/token \
  -d "grant_type=authorization_code" \
  -d "code=$AUTH_CODE" \
  -d "code_verifier=$VERIFIER" \
  -d "client_id=bookshelf-web" \
  -d "redirect_uri=https://app.example.com/callback"
```

```json
{ "access_token": "eyJhbGciOiJSUzI1NiIs...", "expires_in": 3600, "refresh_token": "8xLOxBtZp8" }
```

Every `/v1/books` and `/v1/shelves` call now requires this bearer token,
scoped to `books:read` or `books:write`.

## 2. Webhooks for shelf events (module 2)

```bash
curl -X POST https://api.example.com/v1/webhooks \
  -H "Authorization: Bearer $TOKEN" \
  -d '{"url": "https://partner.example.com/hooks/bookshelf", "events": ["book.added"]}'
```

```json
{ "id": "wh_1", "secret": "whsec_abc123", "events": ["book.added"], "status": "active" }
```

Delivery, signed with `whsec_abc123` via `X-Signature`, retried with
exponential backoff on failure, exactly as in module 2.

## 3. Gateway in front of everything (module 3)

```yaml
routes:
  - paths: ["/v1/books", "/v1/shelves"]
    plugins: [jwt, rate-limiting]
  - paths: ["/v1/webhooks"]
    plugins: [jwt]
```

The gateway validates the OAuth2 token's signature once, at the edge,
before any request reaches `books-service` or `webhooks-service`.

## 4. Versioning and deprecation readiness (module 6)

```http
HTTP/1.1 200 OK
Deprecation: false
```

`v1` is the current, supported version from day one; the header is
present but `false`, ready to flip when `v2` eventually ships.

## 5. Documentation (module 7)

- OpenAPI spec covering `/v1/books`, `/v1/shelves`, `/v1/webhooks`.
- A "Set up webhooks" guide with a working curl example and the
  signature-verification code snippet from module 2.
- An errors reference listing `invalid_token`, `insufficient_scope`,
  `webhook_url_unreachable`.

## 6. Testing strategy (module 5)

```python
def test_requires_write_scope(client):
    token = make_token(scopes=["books:read"])
    r = client.post("/v1/books", json={"title": "Dune"}, headers=auth(token))
    assert r.status_code == 403

def test_webhook_fires_on_book_added(client, webhook_catcher):
    client.post("/v1/books", json={"title": "Dune"}, headers=auth(write_token))
    delivered = webhook_catcher.wait_for("book.added")
    assert delivered["data"]["title"] == "Dune"
```

Plus a contract test validating every response against the OpenAPI spec
in CI, so a schema drift fails the build before it ships.

## 7. Monitoring (module 9)

```
http_requests_total{path="/v1/books",status="403"} 41   ← scope rejections, worth watching
webhook_delivery_failures_total{event="book.added"} 3
```

A trace on `POST /v1/books` shows: gateway auth (3ms) → books-service
write (12ms) → webhook dispatch queued (2ms, async, doesn't block the
201 response to the client).

## 8. Full request lifecycle, end to end

```bash
curl -X POST https://api.example.com/v1/books \
  -H "Authorization: Bearer $TOKEN" \
  -d '{"title": "Dune", "author": "Frank Herbert"}'
```

```json
{ "id": 101, "title": "Dune", "author": "Frank Herbert" }
```

Behind that single `201 Created` response: gateway auth, scope check,
DB write, structured log line with a `request_id`, an async webhook
delivery to every subscriber of `book.added`, and a metric increment —
all seven modules working together on one call.

## Exercise

1. Trace exactly what happens, module by module, when a client with a
   `books:read`-only token attempts `POST /v1/books` — where does it
   get rejected, and what status code and error body does it receive?
2. Add rate limiting specifically to the `/v1/webhooks` registration
   endpoint (distinct from the general API rate limit) — why might this
   endpoint deserve a stricter limit than `/v1/books`?
3. A partner's webhook endpoint has been down for six hours. Design the
   monitoring alert that should fire, and what it should tell the
   on-call engineer to do.
4. Sketch the OpenAPI `security` scheme definition for this API's OAuth2
   scopes (`books:read`, `books:write`).
