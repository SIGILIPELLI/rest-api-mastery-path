# 05 · API Testing Strategy

APIs need a layered testing strategy, same as any other software — but
the layers map onto HTTP boundaries in a specific way: unit tests below
the wire, contract and integration tests at the wire, and end-to-end
tests across the whole system.

## The pyramid, for APIs

```
        /\        end-to-end (few, slow, full stack + real DB)
       /  \
      /----\      integration (moderate, real DB, HTTP layer, mocked externals)
     /      \
    /--------\    contract tests (schema/consumer-driven, fast)
   /          \
  /------------\  unit tests (many, fast, no network/DB)
```

Most bugs should be caught at the bottom layers; the top layer exists to
catch what only shows up when everything runs together.

## Unit tests: pure logic, no HTTP

```python
def test_calculate_discount():
    assert calculate_discount(price=100, coupon="SAVE10") == 90
```

No server running, no database — just the function under test.

## Integration tests: hit the real HTTP layer

```python
def test_create_order_returns_201(client, db):
    response = client.post("/v1/orders", json={"item_id": 5, "qty": 2})
    assert response.status_code == 201
    assert response.json()["status"] == "pending"
    # verify it actually landed in the DB, not just the response shape
    assert db.orders.find_one(id=response.json()["id"]) is not None
```

This runs against a real (test) database and the real routing/
serialization code, catching bugs unit tests can't — wrong status code,
wrong JSON shape, a route that isn't wired up.

## Contract tests: does the API still match its spec?

Given an OpenAPI spec (Level 1, module 7), validate live responses
against it automatically:

```bash
curl -s https://staging.example.com/v1/orders/42 | \
  openapi-spec-validator --schema openapi.yaml --response -
```

Consumer-driven contract testing (e.g. Pact) goes further: the
**consumer** (a frontend or downstream service) records what it expects
from the API —

```json
{
  "request": { "method": "GET", "path": "/v1/orders/42" },
  "response": { "status": 200, "body": { "id": 42, "total": "number" } }
}
```

— and the **provider**'s CI replays that contract against the real API
before every deploy, catching breaking changes before they ship, without
either side needing the other's full codebase running.

## End-to-end tests: the whole system, from outside

```bash
# a real client hitting a real staging deployment
curl -X POST https://staging.example.com/v1/orders \
  -H "Authorization: Bearer $STAGING_TOKEN" \
  -d '{"item_id": 5, "qty": 2}'
```

Slow and few in number by design — they exercise auth, gateway routing,
the real database, and any real downstream services together. Reserve
them for critical user journeys (checkout, signup), not every endpoint.

## Testing failure modes, not just happy paths

A thorough suite for one endpoint covers:

```python
def test_order_requires_auth(client):
    r = client.post("/v1/orders", json={"item_id": 5})
    assert r.status_code == 401

def test_order_rejects_invalid_item(client):
    r = client.post("/v1/orders", json={"item_id": 99999}, headers=auth)
    assert r.status_code == 404

def test_order_rejects_malformed_body(client):
    r = client.post("/v1/orders", data="not json", headers=auth)
    assert r.status_code == 400

def test_order_is_idempotent(client):
    key = {"Idempotency-Key": "abc-123"}
    r1 = client.post("/v1/orders", json={"item_id": 5}, headers={**auth, **key})
    r2 = client.post("/v1/orders", json={"item_id": 5}, headers={**auth, **key})
    assert r1.json()["id"] == r2.json()["id"]
```

## Worked example: catching a breaking change before it ships

A backend engineer renames a response field from `total` to
`total_amount` to be more explicit. Without contract tests, this passes
unit and integration tests (they only check the new field name) but
breaks every existing client silently in production.

With consumer-driven contracts in CI: the provider pipeline replays the
recorded consumer contract (`{"total": "number"}`) against the new code,
the field is missing, the pipeline fails the build — the break never
reaches production. The fix: add `total_amount` alongside `total`,
deprecate `total` per module 6, remove it only after consumers migrate.

## Exercise

1. Where would you catch a bug where the discount calculation is wrong,
   but the HTTP status code and JSON shape are correct — which layer of
   the pyramid, and why not a layer above or below it?
2. Explain the difference between an integration test and a contract
   test for the same `GET /v1/orders/42` endpoint.
3. Why should end-to-end tests be few and reserved for critical
   journeys rather than covering every endpoint?
4. Design three failure-mode tests (beyond auth/validation/idempotency
   shown above) for a `DELETE /v1/orders/{id}` endpoint.
