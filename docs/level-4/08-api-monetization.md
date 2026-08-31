# 08 · API Monetization & Developer Portals

Some APIs are the product itself — customers pay directly for access.
This module covers metering usage, billing for it, and the developer
portal experience that makes a paid API self-service.

## Metering: counting what customers use

```bash
curl https://api.example.com/v1/geocode?address=1600+Amphitheatre+Pkwy \
  -H "Authorization: Bearer sk_live_..."
```

```http
HTTP/1.1 200 OK
X-Usage-Billed: 1
```

Every billable call increments a per-account usage counter, typically
recorded asynchronously (so metering never adds latency to the actual
response) via the same event-driven pattern from module 7:

```json
{ "event": "api_call.billed", "account_id": "acct_42", "endpoint": "/v1/geocode", "units": 1 }
```

## Pricing models

```
Tier        | Included calls/mo | Overage       | Price
------------|--------------------|--------------|----------
Free         | 1,000             | blocked       | $0
Pro          | 100,000           | $0.001/call  | $49/mo
Enterprise   | Custom            | Custom        | Custom (negotiated)
```

```bash
curl https://api.example.com/v1/geocode -H "Authorization: Bearer $FREE_TIER_KEY"
```

```json
{ "error": { "code": "quota_exceeded", "message": "Free tier limit of 1,000 calls/mo reached. Upgrade at https://example.com/pricing" } }
```

`HTTP/1.1 429 Too Many Requests`

Unlike the rate limiting in Level 2 (fairness, resets every window),
quota exhaustion here is a *billing* boundary — it resets on the
billing cycle, not every minute, and the fix for the customer is
upgrading a plan, not waiting 60 seconds.

## Usage dashboards: self-service visibility

```bash
curl https://api.example.com/v1/account/usage \
  -H "Authorization: Bearer $DASHBOARD_TOKEN"
```

```json
{
  "period": "2026-08",
  "calls_used": 84213,
  "calls_included": 100000,
  "projected_overage_cost_usd": 0.00,
  "breakdown": { "/v1/geocode": 71000, "/v1/reverse-geocode": 13213 }
}
```

Giving developers this data directly, in real time, avoids the worst
outcome for a paid API: a customer discovering a huge bill at the end
of the month with no way to have seen it coming.

## The developer portal as the whole self-service loop

A monetized API's portal typically bundles:

- Sign-up → key generation (module 4)
- Interactive docs and quickstart (Level 3, module 7)
- Live usage dashboard (above)
- Billing/invoices and plan upgrade, self-service, no sales call needed
  for the lower tiers
- Support ticket / community forum link

```bash
curl -X POST https://api.example.com/v1/account/plan \
  -H "Authorization: Bearer $DASHBOARD_TOKEN" \
  -d '{"plan": "pro"}'
```

```json
{ "plan": "pro", "effective": "2026-09-01", "next_invoice_usd": 49.00 }
```

## Worked example: designing quota enforcement without hurting reliability

A naive implementation checks quota synchronously against a database on
every single call — adding latency and a new point of failure to every
request, and risking overselling if two concurrent requests both read
"999 of 1000 used" before either writes back 1000.

A better design:

1. Maintain a fast, in-memory (Redis) counter per account, incremented
   atomically (`INCR`) on every call — no read-then-write race.
2. Check the counter against the plan's limit synchronously (this part
   must be fast and correct) — return `429 quota_exceeded` immediately
   if over.
3. Asynchronously reconcile the fast counter against the durable
   billing ledger (via the event stream from module 7) for accurate
   invoicing, decoupled from the hot request path.
4. Reset the counter on the billing cycle boundary, not a rolling
   window.

## Exercise

1. Why is a naive read-then-write quota check subject to a race
   condition under concurrent requests, and how does an atomic
   increment (`INCR`) fix it?
2. Explain the difference between rate limiting (Level 2, module 9) and
   billing quota enforcement — why do they need different reset
   windows and different client-facing remedies?
3. Design the response body and status code for a customer who's 80%
   through their monthly quota — should this ever be surfaced
   proactively, and how?
4. A customer disputes an invoice, claiming the usage count is wrong.
   What data would you need to have been logging all along to resolve
   this dispute with confidence?
