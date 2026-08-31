# Level 2 · Intermediate <span class="level-badge">Building Real APIs</span>

Goal: handle the concerns every production API needs — pagination, filtering,
documentation, idempotency, rate limiting, and caching.

## Modules

1. [Pagination](01-pagination.md)
2. [Filtering & Sorting](02-filtering-sorting.md)
3. [HATEOAS Basics](03-hateoas-basics.md)
4. [OpenAPI/Swagger Spec Basics](04-openapi-swagger-basics.md)
5. [Idempotency](05-idempotency.md)
6. [Rate Limiting](06-rate-limiting.md)
7. [Caching Headers (ETag, Cache-Control)](07-caching-headers.md)
8. [Content Negotiation](08-content-negotiation.md)
9. [Bulk Operations & Batch Requests](09-bulk-operations.md)
10. [Project — Paginated, Cacheable API](10-project-paginated-api.md)

By the end of this level you'll be able to design collection endpoints that
scale (pagination, filtering, sorting), keep clients and servers in sync
efficiently (caching, conditional requests), protect an API from abuse and
retries gone wrong (rate limiting, idempotency), speak more than one wire
format when needed (content negotiation), and document the whole surface in
an OpenAPI spec other engineers and tools can consume directly.
