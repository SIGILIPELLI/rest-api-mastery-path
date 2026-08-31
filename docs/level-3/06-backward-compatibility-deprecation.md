# 06 · Backward Compatibility & Deprecation

Once an API has real clients, every change is a promise you might break.
This module is about changing an API safely: what counts as a breaking
change, how to version, and how to retire something without breaking
everyone who depends on it.

## Breaking vs. non-breaking changes

**Safe (non-breaking), because well-behaved clients ignore what they
don't recognize:**

- Adding a new optional field to a response.
- Adding a new endpoint.
- Adding a new optional request parameter with a sensible default.
- Adding a new enum value, *if* clients are documented to handle unknown
  values gracefully.

**Breaking, because existing clients will misbehave or crash:**

- Removing or renaming a field or endpoint.
- Changing a field's type (`"id": 42` → `"id": "42"`).
- Making a previously optional request field required.
- Changing the meaning of an existing field or status code.
- Tightening validation that previously-valid requests now fail.

## Versioning strategies

```
URL path:    /v1/orders   /v2/orders
Header:      Accept: application/vnd.example.v2+json
Query param: /orders?version=2
```

URL-path versioning is the most common because it's visible, cacheable
per-version, and trivial to route at the gateway (module 3). Header
versioning keeps URLs stable but is harder to test with a browser and
easy for clients to forget.

```bash
curl https://api.example.com/v2/orders/42 \
  -H "Authorization: Bearer $TOKEN"
```

```json
{ "id": 42, "total_amount": "42.50", "currency": "USD" }
```

```bash
curl https://api.example.com/v1/orders/42 \
  -H "Authorization: Bearer $TOKEN"
```

```json
{ "id": 42, "total": 42.50 }
```

Both versions run simultaneously against the same underlying data —
`v1` and `v2` are just different serializations of the same resource.

## Deprecation, done properly

Never remove a field or version overnight. Announce, signal, and give a
runway:

```http
HTTP/1.1 200 OK
Deprecation: true
Sunset: Sat, 01 Nov 2026 00:00:00 GMT
Link: <https://docs.example.com/migration/v1-to-v2>; rel="deprecation"
```

The `Deprecation` and `Sunset` headers (both real, standardized HTTP
headers) let automated tooling and dashboards detect deprecated usage
without a human reading changelogs.

## Worked example: retiring `v1/orders`

1. **Ship `v2`** alongside `v1`, with the field rename and any other
   accumulated breaking changes bundled into one release, not drip-fed.
2. **Announce** the deprecation of `v1` with a concrete sunset date, in
   the changelog, docs, and via the `Deprecation`/`Sunset` headers on
   every `v1` response.
3. **Instrument** `v1` usage — log which API keys are still calling it —
   so you know who to actually contact before the cutoff, rather than
   guessing.
4. **Reach out directly** to the remaining heavy `v1` callers as the
   sunset date approaches; a header alone is easy to miss.
5. **Sunset**: after the date, `v1` returns `410 Gone` with a body
   pointing at the migration guide, rather than disappearing silently
   or (worse) returning wrong/broken data.

```json
{
  "error": {
    "code": "version_sunset",
    "message": "API v1 was retired on 2026-11-01. Migrate to v2.",
    "docs": "https://docs.example.com/migration/v1-to-v2"
  }
}
```

## Additive-only evolution within a version

Many teams avoid ever bumping the major version by committing to
additive-only changes within `v1` forever: new optional fields, new
endpoints, but never removing or repurposing anything. This works well
for years but eventually accumulates cruft (unused legacy fields kept
around solely for backward compatibility) — the trade-off is fewer
version migrations for clients, at the cost of a messier schema over
time.

## Exercise

1. Classify each as breaking or non-breaking: (a) adding a `currency`
   field to an order response, (b) changing `total` from a number to a
   string, (c) making a previously-optional `email` field required on
   signup, (d) adding a new `DELETE /v1/orders/{id}` endpoint.
2. Why is returning `410 Gone` with a migration link better than simply
   turning off a sunset endpoint with no response at all?
3. Compare URL-path versioning and header versioning: what's one
   advantage each has that the other lacks?
4. A client depends on an undocumented quirk of your API (a bug that
   happens to be convenient for them). Is fixing that bug a breaking
   change? How would you handle it?
