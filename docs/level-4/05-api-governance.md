# 05 · API Governance & Design Reviews

Once an organization has dozens of teams each building APIs, without
coordination they diverge: different pagination styles, different error
formats, inconsistent auth. Governance is the process that keeps many
independently-built APIs feeling like one coherent platform.

## A design-first review process

Before a team writes implementation code, they submit an OpenAPI spec
for review:

```yaml
paths:
  /v1/refunds:
    post:
      summary: Issue a refund
      requestBody:
        content:
          application/json:
            schema: { $ref: "#/components/schemas/RefundRequest" }
```

A review checklist a governance team applies to every new spec:

- Does pagination match the org standard (cursor-based, `limit`/
  `cursor` params, per Level 2 module 1) — not a one-off `page`/`per_page`?
- Does the error shape match the org standard envelope (Level 1 error
  handling module) — not a custom format for this one team?
- Are resource names and casing consistent with existing APIs
  (`snake_case` fields, plural resource paths)?
- Is versioning strategy consistent (Level 3, module 6)?
- Are auth scopes named consistently with other services'?

Catching inconsistency at the spec-review stage, before code exists, is
far cheaper than a "fix it later" migration once clients depend on the
inconsistent shape.

## A shared style guide as the actual standard

```markdown
## Org API Style Guide (excerpt)
- Timestamps: ISO 8601 UTC, field name suffix `_at` (`created_at`).
- Money: integer cents, field name suffix `_cents` (`total_cents: 4250`)
  — never a float for currency.
- Errors: `{ "error": { "code", "message", "field" } }` — always.
- Pagination: cursor-based, `meta.next_cursor` — never offset-based for
  new APIs.
```

A written, linkable style guide turns subjective review comments
("I don't like this") into objective checks ("this violates section 3
of the style guide") — faster reviews, less friction between teams.

## Linting the spec automatically

```bash
spectral lint openapi.yaml --ruleset org-api-rules.yaml
```

```
5:7   error   org-money-as-integer   "total" must not be type "number" for a currency field
12:3  warning org-error-shape        error response missing required "field" property
```

Automating the style guide as lint rules that run in CI catches
violations before a human reviewer even looks at the spec — reserving
human review for judgment calls (is this resource modeled well?) rather
than mechanical formatting checks.

## Federated governance: guardrails, not a bottleneck

A common failure mode is governance becoming a slow, centralized
gatekeeper every team waits weeks on. The healthier model:

- **Automated checks** (linting, contract tests) run instantly, in CI,
  for every team, no human bottleneck.
- **Human review** is required only for genuinely new patterns (a new
  auth model, a new resource type unlike anything existing) — not for
  every routine endpoint addition.
- **The style guide itself** is versioned and owned collaboratively,
  not dictated unilaterally — teams that hit a real limitation propose
  a change to the guide, not a one-off exception for themselves.

## Worked example: reviewing a new team's first API

A newly-formed team submits a spec for a `notifications` API before
writing any code.

1. Automated lint flags: `sent_time` should be `sent_at` (naming
   convention), and pagination uses `page`/`per_page` instead of the
   cursor standard.
2. Team fixes both mechanically — five-minute change, caught before any
   client integrated.
3. Human reviewer flags a real design question: the spec models
   "notification preferences" as a sub-resource of `users`, but the org
   has a precedent (from the `billing` team) of preferences as their
   own top-level resource with a reference to `user_id` — reviewer asks
   the team to align, for consistency other teams can rely on.
4. Spec approved; implementation begins with the shape already
   validated, no rework needed after the fact.

## Exercise

1. Why is design-first review (spec before code) cheaper to act on than
   reviewing the finished implementation?
2. Give two style-guide rules besides the ones shown that you'd want
   enforced automatically across every API in an organization.
3. Explain the difference between what should be an automated lint rule
   versus what needs human judgment in a spec review.
4. A team argues their new API needs offset-based pagination instead of
   the org's cursor standard, for a genuine technical reason. What's
   the healthier governance response — grant a silent exception, deny
   it outright, or something else?
