# 07 · API Documentation Best Practices

An API is only as usable as its documentation. This module covers what
makes docs actually work for a developer trying to integrate against
your API for the first time, at 11pm, with no one to ask.

## The documentation stack

- **Reference docs** — generated from the OpenAPI spec (Level 1, module
  7): every endpoint, parameter, and response shape, always accurate
  because it's generated, not hand-written.
- **Guides / tutorials** — hand-written, task-oriented walkthroughs
  ("Authenticate and make your first request", "Set up webhooks").
- **Changelog** — dated, human-readable list of what changed, linked to
  migration guides for breaking changes (module 6).
- **Interactive playground** — lets a developer make a real call from
  the browser (Swagger UI, Redoc, Postman collections).

## Reference docs generated from OpenAPI

```yaml
paths:
  /v1/orders/{id}:
    get:
      summary: Retrieve an order
      parameters:
        - name: id
          in: path
          required: true
          schema: { type: integer }
      responses:
        "200":
          description: The order
          content:
            application/json:
              schema: { $ref: "#/components/schemas/Order" }
              example: { id: 42, total: 42.50, status: "shipped" }
        "404":
          description: Order not found
```

The `example` block matters as much as the schema — most developers
copy the example and modify it rather than constructing a request from
the schema alone.

## A good "Quickstart" guide

The single highest-leverage doc page. It should take a developer from
zero to one successful authenticated call in under five minutes:

```bash
# 1. Get an API key from https://example.com/dashboard/keys
# 2. Make your first call:
curl https://api.example.com/v1/orders \
  -H "Authorization: Bearer YOUR_API_KEY"
```

```json
{ "data": [], "meta": { "total": 0 } }
```

If step 2 doesn't work copy-pasted exactly as shown (real hostname, real
header format, realistic response), the guide has failed its one job.

## Documenting errors, not just happy paths

```json
{
  "error": {
    "code": "insufficient_funds",
    "message": "Account balance is too low to complete this order.",
    "field": null,
    "docs_url": "https://docs.example.com/errors/insufficient_funds"
  }
}
```

An errors reference page listing every `code` your API can return, what
it means, and how to handle it, saves far more support tickets than
polishing the happy-path docs further.

## Changelog entries that actually help

```markdown
## 2026-08-15
### Added
- `GET /v1/orders` now supports `sort=-created_at` (module 2, Level 2).

### Deprecated
- `v1/orders.total` is deprecated in favor of `total_amount`. See the
  [migration guide](/migration/total-field). Sunset: 2026-11-01.
```

Dated, categorized (Added/Changed/Deprecated/Removed), and every
breaking or deprecating entry links to a migration guide — not just "we
changed some stuff."

## Worked example: docs for a new endpoint

A team ships `POST /v1/orders/{id}/refund`. A complete doc addition
includes:

1. OpenAPI spec entry (auto-generates the reference page).
2. A short guide section: "Refunding an order" with a working curl
   example and both success and `409 Conflict` (already refunded)
   responses shown.
3. A changelog entry under "Added."
4. If this replaces an older `POST /v1/refunds` endpoint, a deprecation
   entry for the old one, per module 6.

## Exercise

1. Why should example values in an OpenAPI spec be realistic (`"id":
   42`) rather than placeholders (`"id": "string"`)?
2. What's the difference in purpose between a quickstart guide and full
   reference docs — why do you need both?
3. Design an errors-reference entry for a `rate_limited` error code,
   including what a client should do in response.
4. A breaking change ships without a changelog entry. What's the
   most likely support-team consequence a week later?
