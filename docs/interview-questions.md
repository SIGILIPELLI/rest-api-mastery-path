---
title: "REST API Interview Questions and Answers (178 Questions)"
description: "178 REST API interview questions with answers and explanations, from beginner to advanced, organised by level. Free, from the REST API Mastery Path."
---

# REST API Interview Questions and Answers

**178 questions** with the correct answer and a short explanation, organised from beginner (Level 1) to advanced (Level 4). Every question comes from a lesson in the free REST API Mastery Path, so you can open the lesson to go deeper.

!!! tip "Practise, don't just read"
    Each lesson also ends with an interactive quiz, and the [final quiz](quiz.md) tests all four levels. Try to answer each question below before reading the answer.

## Level 1 · Entry

### [01 · What Is REST? Constraints &amp; Principles](level-1/01-what-is-rest.md)

**Q1.** What is REST?

**Answer:** An architectural style defined by constraints, described by Roy Fielding in his 2000 dissertation

Few public APIs satisfy every constraint strictly.

**Q2.** What does statelessness mean for the API server?

**Answer:** Every request carries all the information needed, and the server stores no per-client session state between requests

The database can have state; the API server just can't hold per-client session memory.

**Q3.** Why does statelessness matter for scaling?

**Answer:** Any server instance can handle any request, so no sticky sessions are needed

It makes horizontal scaling and load balancing easy.

**Q4.** What does the client-server constraint allow?

**Answer:** Rewriting the backend (e.g. swapping Postgres for MongoDB) without touching the client if the API contract stays the same

They are independently evolvable components.

### [02 · HTTP Methods &amp; Status Codes](level-1/02-http-methods-status-codes.md)

**Q5.** Which methods are both safe and idempotent?

**Answer:** GET, HEAD and OPTIONS

PUT and DELETE are idempotent but not safe.

**Q6.** What does idempotent mean?

**Answer:** Calling it once has the same effect as calling it many times

DELETE /books/17 five times still ends with book 17 gone, though later responses may be 404.

**Q7.** Why is GET /books/17/delete an anti-pattern?

**Answer:** It violates safety, so prefetchers, crawlers and back-button caching could trigger deletes

Use DELETE for deletions.

**Q8.** Is PATCH guaranteed idempotent by the HTTP spec?

**Answer:** No: 'set field X to Y' is idempotent but 'increment X by 1' is not

It depends on the operation.

### [03 · URL &amp; Resource Design](level-1/03-url-resource-design.md)

**Q9.** Which URL design is preferred?

**Answer:** DELETE /orders/482 rather than POST /deleteOrder?id=482

URLs identify resources (nouns); the HTTP method carries the action.

**Q10.** How should a genuine non-CRUD action be modelled?

**Answer:** Either a controlled verb segment like POST /orders/482/cancel, or a sub-resource like POST /password-reset-requests; pick one convention

What to avoid is verbs for standard CRUD.

**Q11.** How deep should nesting go?

**Answer:** Avoid more than 2–3 levels; give deep children their own filterable top-level collection

For example GET /reviews/9/comments or GET /comments/4.

**Q12.** Should collections be singular or plural?

**Answer:** Plural, consistently (/books)

It reads naturally for both the collection and singular forms.

### [04 · Request/Response Anatomy](level-1/04-request-response-anatomy.md)

**Q13.** What are the four parts of an HTTP message?

**Answer:** Start line, headers, a blank line, and an optional body

The blank line tells the parser the headers are done.

**Q14.** Are HTTP header names case-sensitive?

**Answer:** No: content-type equals Content-Type

Header order mostly doesn't matter either.

**Q15.** Which should clients check: the status code or the reason phrase?

**Answer:** The numeric status code; the reason phrase is informational only

For example 201 versus 'Created'.

**Q16.** What does the request line contain?

**Answer:** The method, the path (not the full URL) and the HTTP version

The host is a separate header.

### [05 · Query Params vs Path Params](level-1/05-query-vs-path-params.md)

**Q17.** What is the rule of thumb for a path parameter?

**Answer:** If removing the value would leave you unable to say which resource you mean, it's a path parameter

For example /books/17.

**Q18.** Which is the better design?

**Answer:** GET /books/17 rather than GET /books?id=17

17 identifies the resource; it isn't a filter.

**Q19.** What are query parameters for?

**Answer:** Filtering, sorting, paginating or optional modifiers on a collection

If removing it leaves a meaningful, broader request, it's a query parameter.

**Q20.** How should GET /authors/3/books?sort=-published_year&amp;limit=10 be read?

**Answer:** Path parameter authorId=3 picks which author's books; sort and limit are query parameters

Most real endpoints use both together.

### [06 · Using curl, httpie &amp; Postman](level-1/06-curl-httpie-postman.md)

**Q21.** What does `curl -i` do?

**Answer:** Prepends the response headers to the output

-v goes further and shows the request curl sent.

**Q22.** What does `-w "%{http_code}\n"` help with?

**Answer:** Checking success or failure in shell scripts without parsing the body

It prints the status code.

**Q23.** How do you send query values safely with curl?

**Answer:** -G with --data-urlencode, which encodes values like 'Ursula K. Le Guin'

-G turns the data into a query string.

**Q24.** When is curl preferred over httpie?

**Answer:** For scripting and CI, since it's preinstalled almost everywhere

httpie reads more like English and pretty-prints JSON; Postman suits saved collections and teams.

### [07 · Authentication Basics (API Keys &amp; Bearer Tokens)](level-1/07-authentication-basics.md)

**Q25.** What does authentication answer versus authorization?

**Answer:** Authentication: who is calling; authorization: what they are allowed to do

The lesson expands authorization with OAuth2 in Level 3.

**Q26.** Why avoid API keys in query parameters?

**Answer:** URLs get logged by proxies, browser history and server logs, leaking the secret

Prefer a header such as X-API-Key.

**Q27.** What does a leaked API key mean?

**Answer:** It's valid until manually revoked; keys carry no built-in expiry or scoping

Never embed a secret key in client-side JS or a mobile app binary.

**Q28.** Why must Bearer tokens only be sent over HTTPS?

**Answer:** Whoever holds the token can use it, with no additional proof

Sent as Authorization: Bearer &lt;token&gt;.

### [08 · Error Handling Conventions](level-1/08-error-handling-conventions.md)

**Q29.** What are the three ingredients of a good error response?

**Answer:** The right HTTP status, a consistent structured body, and a human-readable message

Generic tooling and monitoring check the status first.

**Q30.** Why make error.code a stable machine-readable string?

**Answer:** Clients can switch on it without parsing English, and it won't change when the message wording does

For example validation_failed.

**Q31.** Why is error.details an array?

**Answer:** To report multiple validation failures at once instead of making the client fix and resubmit one at a time

For example both email and age issues in one 422.

**Q32.** What is RFC 9457?

**Answer:** The IETF Problem Details standard for HTTP APIs, with application/problem+json, obsoleting RFC 7807

It includes fields like type and title.

### [09 · API Versioning Basics](level-1/09-versioning-basics.md)

**Q33.** Which change is breaking?

**Answer:** Removing a field, renaming a field, changing a field's type, or making validation stricter

Rule of thumb: if unmodified client code could start erroring, it's breaking.

**Q34.** What are the pros of URL path versioning (/v1/books/17)?

**Answer:** Very visible and explicit, and trivial to route versions to different backends

The con: the same resource has two URLs, a pragmatic tradeoff Stripe and GitHub accept.

**Q35.** Why does versioning matter at all?

**Answer:** You can't change a field's meaning or remove an endpoint when consumers you don't control depend on it

It lets old consumers keep working until they migrate.

**Q36.** How does header versioning differ?

**Answer:** The URL stays the same and the client requests a version through a header like X-API-Version: 2

It keeps the URL as the single stable identifier.

### [10 · Capstone — Bookshelf API Spec](level-1/10-capstone-bookshelf-api.md)

**Q37.** How is 'a book on a shelf' modelled in the capstone?

**Answer:** As the nested path /v1/shelves/{shelfId}/books rather than a separate shelf-item ID

It avoids inventing an ID the client must track.

**Q38.** What do unauthenticated requests receive?

**Answer:** 401 Unauthorized with a structured error body (code 'unauthorized')

All endpoints require a bearer token.

**Q39.** What is the deliverable of the capstone?

**Answer:** A precise, well-documented API design (spec), not a running server

Exactly as you'd hand it to another engineer to implement.

**Q40.** Which URL design points does the capstone highlight?

**Answer:** The /v1/ prefix, plural resource names, path parameters identifying resources, and nesting for the relationship

They come from modules 3 and 9.

## Level 2 · Intermediate

### [01 · Pagination](level-2/01-pagination.md)

**Q41.** What are the two problems with offset/limit pagination at scale?

**Answer:** OFFSET scans and discards rows so it slows as offset grows, and inserts or deletes between requests cause page drift

A client may see an item twice or skip one.

**Q42.** How does cursor-based pagination work?

**Answer:** The server returns an opaque cursor (often base64 of the last sort key) and the next request asks for items after it, e.g. WHERE id &gt; 60

It's an indexed range query, fast at any depth.

**Q43.** What is the main limitation of cursors?

**Answer:** You can't jump directly to page 5, only traverse forward or backward

They're also slightly more complex to implement.

**Q44.** What is page-number pagination?

**Answer:** Offset/limit in disguise: offset = (page - 1) * per_page, with the same tradeoffs

It has a more human-readable query shape.

### [02 · Filtering &amp; Sorting](level-2/02-filtering-sorting.md)

**Q45.** How are multiple filter parameters combined?

**Answer:** Implicitly AND-ed together

One query parameter per filterable field.

**Q46.** What does ?sort=-published_date,title mean?

**Answer:** Sort by published_date descending, then title ascending as a tiebreaker

Compound sort keys matter for stable pagination.

**Q47.** What does ?genre=scifi,fantasy usually mean?

**Answer:** genre is scifi OR fantasy

Some APIs repeat the parameter instead; both are common.

**Q48.** Why should you pick one convention for range filters?

**Answer:** Mixing price_gte on one resource and min_price on another makes an API feel unfinished

Stripe uses created[gte]; suffix operators are also valid.

### [03 · HATEOAS Basics](level-2/03-hateoas-basics.md)

**Q49.** What is HATEOAS?

**Answer:** Hypermedia As The Engine Of Application State: clients navigate the API by following links returned in responses

It's the most-cited, least-implemented part of Fielding's uniform interface constraint.

**Q50.** What does a missing cancel link in an order response tell the client?

**Answer:** Cancellation isn't currently allowed, so the client needn't copy 'only pending orders can be cancelled'

The server decides which actions are available.

**Q51.** Which are two widely used hypermedia formats?

**Answer:** HAL (application/hal+json) and JSON:API (application/vnd.api+json)

They standardise where links live in the payload so generic clients can parse them.

**Q52.** What do most real-world APIs do?

**Answer:** Skip a formal spec and use a plain _links or links object, HAL-ish

The lesson shows a minimal ad-hoc convention.

### [04 · OpenAPI/Swagger Spec Basics](level-2/04-openapi-swagger-basics.md)

**Q53.** What is OpenAPI?

**Answer:** A machine-readable specification (YAML or JSON) describing endpoints, request/response shapes and auth

It was formerly called Swagger.

**Q54.** What can tools generate from an OpenAPI document?

**Answer:** Interactive docs, client SDKs, server stubs and test suites

The spec becomes a single source of truth instead of drifting hand-written docs.

**Q55.** What does $ref: '#/components/schemas/Book' do?

**Answer:** Reuses a shared schema defined once under components

Book and NewBook are shared definitions.

**Q56.** How is the path parameter {id} declared in the spec?

**Answer:** As a parameter with in: path and required: true

Query parameters use in: query.

### [05 · Idempotency](level-2/05-idempotency.md)

**Q57.** Why is POST risky to retry?

**Answer:** It typically creates a new resource, so a retry after a lost response can create duplicates

For example two orders instead of one.

**Q58.** Which patch is idempotent?

**Answer:** {"price": 15} (set an absolute value), unlike {"price_delta": -5}

Applying the delta twice decreases the price by 10.

**Q59.** How does the idempotency-key pattern work?

**Answer:** The client sends a unique token per logical operation; the server remembers processed keys and returns the original result instead of repeating the side effect

It's used for payments and orders, where duplicates are costly.

**Q60.** Which methods are idempotent by definition?

**Answer:** GET, PUT, DELETE, HEAD and OPTIONS

PATCH isn't guaranteed.

### [06 · Rate Limiting](level-2/06-rate-limiting.md)

**Q61.** What is the boundary-burst problem of the fixed window algorithm?

**Answer:** A client can send 200 requests in the 2 seconds straddling a boundary, well above 100 per minute

Sliding windows smooth this at the cost of more bookkeeping.

**Q62.** How does a token bucket behave?

**Answer:** Tokens refill at a steady rate up to a cap, each request uses one; it allows short bursts while enforcing a long-run average

A bucket of 20 with refill 10/s lets 20 requests through instantly, then 10 more after 1 s.

**Q63.** What does the leaky bucket do?

**Answer:** Processes (or discards) queued requests at a constant rate, trading burst tolerance for steady output

It's common in network traffic shaping.

**Q64.** What does RateLimit-Reset: 37 mean?

**Answer:** Seconds until the window or bucket resets

GitHub's X-RateLimit-Reset is a Unix timestamp instead.

### [07 · Caching Headers (ETag, Cache-Control)](level-2/07-caching-headers.md)

**Q65.** What does Cache-Control: no-cache actually mean?

**Answer:** It may be cached but must be revalidated with the server before reuse

It's misleadingly named; no-store means do not cache anywhere.

**Q66.** What does the server return to If-None-Match when the ETag is unchanged?

**Answer:** 304 Not Modified with no body

The client keeps using its cached copy, saving the payload transfer.

**Q67.** What does private mean in Cache-Control?

**Answer:** Only the end client may cache it, not a shared proxy or CDN (typical for per-user data)

public allows any cache.

**Q68.** What is an ETag?

**Answer:** An opaque fingerprint of a resource's current state, often a content hash

Last-Modified is a coarser, second-level alternative.

### [08 · Content Negotiation](level-2/08-content-negotiation.md)

**Q69.** What does Accept: application/json;q=0.9, application/xml;q=0.5, */*;q=0.1 express?

**Answer:** Prefer JSON, XML is an acceptable fallback, anything as a last resort

The server picks the highest-q type it can produce.

**Q70.** What should a server return if it can't produce any acceptable format?

**Answer:** 406 Not Acceptable

Many JSON-only APIs skip strict 406 handling, which is fine if documented.

**Q71.** What does 415 Unsupported Media Type signal?

**Answer:** The request's Content-Type isn't supported

Accept negotiates the response; Content-Type describes the request body.

**Q72.** Which header handles language negotiation?

**Answer:** Accept-Language

It uses the same mechanism as Accept.

### [09 · Bulk Operations &amp; Batch Requests](level-2/09-bulk-operations.md)

**Q73.** Why have bulk endpoints?

**Answer:** One request per item costs round trips, overhead and more chances to partially fail

500 records would mean 500 round trips.

**Q74.** What status does a bulk request with mixed outcomes return?

**Answer:** 207 Multi-Status with a per-item result

The request as a whole doesn't have one status; check each item.

**Q75.** Why is partial success the norm in bulk operations?

**Answer:** One bad item shouldn't fail the other 499

The example returns 201, 201 and 422 for the three items.

**Q76.** What should each item in a bulk result carry?

**Answer:** Its own status and either data or an error

The client then inspects each item.

### [10 · Project — Paginated, Cacheable API](level-2/10-project-paginated-api.md)

**Q77.** Why does the project use cursor pagination?

**Answer:** The collection grows continuously as users add books

The response carries next_cursor and has_more.

**Q78.** How are unrecognised sort or filter fields handled?

**Answer:** Validated against an allowlist; unknown fields return 400 with invalid_sort_field or invalid_filter_field

The allowlist also protects the database.

**Q79.** What is If-Match used for on writes?

**Answer:** Optimistic concurrency: the write succeeds only if the ETag still matches

It prevents lost updates.

**Q80.** What does a retried POST with the same Idempotency-Key return?

**Answer:** The original 201 response, without creating a duplicate book

The server remembers processed keys.

## Level 3 · Advanced

### [01 · OAuth2 Flows](level-3/01-oauth2-flows.md)

**Q81.** What is the point of OAuth2?

**Answer:** A third-party app gets a scoped, revocable access token without ever seeing the user's password

It's the framework for how a token gets issued.

**Q82.** Why does the Authorization Code flow use two exchanges?

**Answer:** The code is exposed in the redirect URL but is single-use and short-lived; the token exchange happens server-to-server with a secret the browser never sees

A leaked code alone is useless without the client secret.

**Q83.** What is the state parameter for?

**Answer:** A CSRF defense: the client verifies the same random value comes back in the redirect

Without it an attacker could trick a victim into completing an attacker-initiated flow.

**Q84.** Who are the four OAuth2 actors?

**Answer:** Resource owner, client, authorization server and resource server

PKCE is required for public clients like mobile apps and SPAs.

### [02 · Webhooks](level-3/02-webhooks.md)

**Q85.** How do webhooks differ from polling?

**Answer:** The server pushes an event to a registered URL when something happens, with no wasted polling requests

The cost: the receiver needs a publicly reachable, always-on endpoint.

**Q86.** How does a receiver verify a webhook?

**Answer:** Recompute an HMAC signature over the raw body with the shared secret and compare

Otherwise anyone who finds the URL could POST fake events.

**Q87.** Why use hmac.compare_digest instead of ==?

**Answer:** Constant-time comparison avoids leaking information through timing

The handler returns 401 for an invalid signature.

**Q88.** What should a webhook receiver do quickly?

**Answer:** Respond (e.g. 200) fast and process heavy work separately

Slow responses cause the sender to retry.

### [03 · API Gateways Basics](level-3/03-api-gateways-basics.md)

**Q89.** What is an API gateway?

**Answer:** The single entry point in front of services that routes, secures and shapes traffic

Clients don't need to know every backend's address.

**Q90.** Why validate tokens once at the gateway?

**Answer:** Individual services don't each re-implement JWT verification

It's authentication at the edge.

**Q91.** What does TLS termination at the gateway mean?

**Answer:** The gateway holds the certificate; internal traffic can run plain HTTP inside a trusted network

It's one of the gateway's jobs.

**Q92.** Which of these is a gateway responsibility?

**Answer:** Routing, auth, rate limiting, transformation and observability

For example /v1/orders/* goes to orders-service.

### [04 · GraphQL vs REST](level-3/04-graphql-vs-rest.md)

**Q93.** What problems does GraphQL address compared with REST?

**Answer:** Over-fetching, under-fetching (chains of calls) and client-driven response shape

One request returns exactly the fields asked for.

**Q94.** Why does GraphQL defeat standard HTTP caching?

**Answer:** A single POST /graphql endpoint isn't cacheable by URL, so caching must be rebuilt at the application layer

For example persisted queries and a normalized client cache.

**Q95.** Why do GraphQL servers need query complexity analysis?

**Answer:** One query can ask for deeply nested, expensive data, unlike a REST endpoint with predictable cost

It prevents abuse.

**Q96.** What is the GraphQL schema?

**Answer:** A typed definition of the data the client can query, e.g. type User { id: ID! name: String! }

The exclamation mark means non-null.

### [05 · API Testing Strategy](level-3/05-api-testing-strategy.md)

**Q97.** What does the API testing pyramid say?

**Answer:** Many fast unit tests at the bottom, then contract and integration tests, with few slow end-to-end tests on top

Most bugs should be caught at the bottom layers.

**Q98.** What does an integration test catch that a unit test can't?

**Answer:** A wrong status code or JSON shape, or a route that isn't wired up, against a real test DB and the real HTTP layer

It also verifies the row landed in the DB.

**Q99.** How does consumer-driven contract testing (Pact) work?

**Answer:** The consumer records what it expects; the provider's CI replays that contract before every deploy

It catches breaking changes without needing both full codebases.

**Q100.** What is the purpose of end-to-end tests?

**Answer:** To catch what only shows up when the whole system runs together

They are few and slow.

### [06 · Backward Compatibility &amp; Deprecation](level-3/06-backward-compatibility-deprecation.md)

**Q101.** Which is a non-breaking change?

**Answer:** Adding an optional response field or a new endpoint

Well-behaved clients ignore what they don't recognise.

**Q102.** What do the Deprecation and Sunset headers do?

**Answer:** Let tooling detect deprecated usage and the retirement date without humans reading changelogs

A Link header can point to the migration guide.

**Q103.** Why is URL-path versioning the most common?

**Answer:** It's visible, cacheable per version and trivial to route at the gateway

Header versioning is harder to test in a browser and easy to forget.

**Q104.** How should v1 be retired?

**Answer:** Ship v2 alongside, announce with a concrete sunset date, then remove after the runway

Bundle breaking changes into one release rather than drip-feeding.

### [07 · API Documentation Best Practices](level-3/07-api-documentation.md)

**Q105.** Why generate reference docs from the OpenAPI spec?

**Answer:** They stay accurate because they're generated, not hand-written

Guides and tutorials are hand-written.

**Q106.** What is a changelog's role?

**Answer:** A dated, human-readable list of changes linked to migration guides for breaking changes

It's part of the documentation stack.

**Q107.** What does an interactive playground allow?

**Answer:** Making a real call from the browser, such as Swagger UI, Redoc or Postman collections

It lowers the barrier to a first request.

**Q108.** What is the key doc goal described?

**Answer:** Usable by a developer integrating for the first time at 11pm with no one to ask

Guides are task-oriented.

### [08 · gRPC &amp; Alternative Protocols](level-3/08-grpc-alternatives.md)

**Q109.** What does gRPC use?

**Answer:** Compact binary Protocol Buffers over HTTP/2, with services defined in a .proto file

It generates strongly typed client and server code in many languages.

**Q110.** Why does gRPC catch contract mismatches earlier than REST/JSON?

**Answer:** The .proto file is the single source of truth generated on both sides, so mismatches fail at compile time

With JSON, a malformed body fails at runtime.

**Q111.** Why does REST still win for public and browser-facing APIs?

**Answer:** Browsers can't natively speak gRPC, REST/JSON is debuggable with curl, and public integrators expect REST

gRPC-Web exists but adds a proxy layer.

**Q112.** Where is gRPC the common choice?

**Answer:** Internal service-to-service calls in microservices

Both sides control their codegen pipeline within one company.

### [09 · Monitoring &amp; Observability for APIs](level-3/09-monitoring-observability.md)

**Q113.** What are the four golden signals?

**Answer:** Latency, traffic, errors and saturation

Saturation is the leading indicator before latency and errors spike.

**Q114.** Why alert on error rate rather than raw error count?

**Answer:** 12 errors in 18,432 requests (0.07%) is very different from 12 in 40 (30%)

The rate tells you how bad it is.

**Q115.** What does distributed tracing show?

**Answer:** How one request fans out across services under a trace_id, revealing e.g. a slow Postgres query in orders-service

Without it, 'the dashboard is slow' is a mystery.

**Q116.** Why use structured JSON logs with request_id?

**Answer:** They're queryable, and request_id threads through every log line and downstream call for a request

The request_id is generated at the gateway.

### [10 · Project — OAuth2-Secured API with Webhooks](level-3/10-project-oauth2-webhooks.md)

**Q117.** Which OAuth2 flow does the project use?

**Answer:** Authorization Code with PKCE

The token exchange includes code_verifier.

**Q118.** What happens when a token has only books:read and tries POST /v1/books?

**Answer:** 403 Forbidden (insufficient scope)

The test test_requires_write_scope asserts 403.

**Q119.** Where is the OAuth2 token signature validated?

**Answer:** Once at the gateway, before the request reaches books-service or webhooks-service

The gateway applies jwt and rate-limiting plugins.

**Q120.** How are webhook deliveries made reliable and trustworthy?

**Answer:** Signed with the webhook's secret via X-Signature and retried with exponential backoff on failure

The subscriber receives a secret when registering.

## Level 4 · Master

### [01 · Microservices API Design Patterns](level-4/01-microservices-api-patterns.md)

**Q121.** What does database-per-service mean?

**Answer:** Each microservice owns its database and no service reaches into another's tables

orders-service calls users-service's API for a user's name.

**Q122.** What is the API composition pattern?

**Answer:** A gateway or BFF layer merges data from multiple services into one response

The client sees one endpoint like /v1/orders/42/summary.

**Q123.** What is a Backend-for-Frontend (BFF)?

**Answer:** A layer per client type (mobile, web) shaping responses, such as a lean mobile payload versus a full web one

Different clients need different shapes from the same services.

**Q124.** Why does database-per-service make services independently deployable?

**Answer:** No one can break another team's queries by changing a table they don't own

It's the key ownership rule.

### [02 · API Security Hardening (CORS, Injection, OWASP API Top 10)](level-4/02-api-security-hardening.md)

**Q125.** What does BOLA (Broken Object Level Authorization) mean?

**Answer:** Checking that a token is valid but not that this user owns this specific resource

The fix: every lookup filters by the authenticated user as well as the ID.

**Q126.** What is the safe way to handle a user-supplied sort parameter?

**Answer:** Validate against an allowlist of known-safe columns, and use parameterized queries

?sort=id;DROP+TABLE+orders is catastrophic when interpolated directly.

**Q127.** What is the correct CORS configuration?

**Answer:** An explicit origin allowlist (e.g. https://app.example.com) with Vary: Origin, not reflecting any Origin

Reflecting Origin recreates the hole even though browsers reject * with credentials.

**Q128.** What happens to a preflight from an origin not on the allowlist?

**Answer:** It's rejected, so the browser never sends the real request

The example returns 403 for evil.com.

### [03 · Performance at Scale](level-4/03-performance-at-scale.md)

**Q129.** Why does a public product listing with Cache-Control: public, max-age=300 scale well?

**Answer:** It's served from the CDN edge for 5 minutes, so the origin never sees most of the traffic

Private per-user caching still hits the origin but skips the database.

**Q130.** Why invalidate the Redis cache on write, not just by expiry?

**Answer:** So the next read repopulates fresh data instead of serving stale data until the TTL

update_product deletes the key.

**Q131.** What does connection pooling solve?

**Answer:** Opening a DB connection per request is expensive (TCP handshake, auth); a pool reuses warm connections

Too small a pool makes requests queue; too large overwhelms the database. Size by measurement.

**Q132.** Why return 202 Accepted for a long report job?

**Answer:** The client gets an immediate response with a job ID, then polls or receives a webhook when it's done

Generating the report might take two minutes.

### [04 · Designing Public Developer-Facing APIs](level-4/04-public-developer-apis.md)

**Q133.** Why is the raw API key shown only once at creation?

**Answer:** Only its prefix and metadata are kept retrievable, so a leaked key can be identified and revoked without storing the plaintext

Keys are scoped and revocable independently.

**Q134.** What does an sk_test_ key route to?

**Answer:** A fully isolated sandbox with fake payment processing and resettable data

The key prefix signals the environment.

**Q135.** Why generate SDKs from the OpenAPI spec?

**Answer:** Every language binding stays in sync automatically as the spec changes

Hand-maintained SDKs drift out of date.

**Q136.** Why publish rate limit headers for public APIs?

**Answer:** Third-party developers build automated systems against them and can't just be told by your team

For example X-RateLimit-Limit, Remaining and Reset.

### [05 · API Governance &amp; Design Reviews](level-4/05-api-governance.md)

**Q137.** What is a design-first review process?

**Answer:** Teams submit an OpenAPI spec for review before writing implementation code

Catching inconsistency at spec stage is far cheaper than migrating later.

**Q138.** What does the style guide excerpt say about money?

**Answer:** Integer cents with a _cents suffix (total_cents: 4250), never a float

Timestamps are ISO 8601 UTC with an _at suffix.

**Q139.** What does a written style guide change about reviews?

**Answer:** Subjective comments become objective checks against a section, giving faster reviews and less friction

It's the actual standard.

**Q140.** Why does governance matter with dozens of teams?

**Answer:** Without coordination APIs diverge in pagination, error formats and auth, instead of feeling like one coherent platform

The checklist covers pagination, errors, naming, versioning and scopes.

### [06 · Multi-Region &amp; High Availability APIs](level-4/06-multi-region-ha.md)

**Q141.** What are the three reasons for multi-region?

**Answer:** Latency, availability and regulatory data residency

A Singapore customer on a US-only API pays about 200 ms of network transit.

**Q142.** How do active-passive and active-active differ?

**Answer:** Active-passive has one region serving with a replicated idle standby; active-active has several serving live traffic at once

Active-active is harder (cross-region consistency) but wastes no capacity.

**Q143.** What happens to a region failing health checks?

**Answer:** It's automatically removed from DNS/routing so traffic reroutes within seconds

GeoDNS routes customers to their nearest healthy region.

**Q144.** What's the hard part of multi-region?

**Answer:** The database and data consistency; stateless app servers replicate trivially

There is a trade-off between consistency and write latency.

### [07 · Event-Driven APIs &amp; Async Patterns](level-4/07-event-driven-apis.md)

**Q145.** What is the difference between a request and an event?

**Answer:** A request says 'do this and tell me what happened'; an event says 'this happened' with no expectation of who reacts

orders-service publishes order.confirmed without knowing its consumers.

**Q146.** Why do brokers durably store events?

**Answer:** A consumer that's down processes the backlog when back, instead of losing the event as a failed direct HTTP call would

Examples: Kafka, SQS, RabbitMQ.

**Q147.** Why must consumers be idempotent?

**Answer:** Brokers give at-least-once delivery, so the same event may arrive twice

Check processed_events before handling.

**Q148.** What does the outbox pattern solve?

**Answer:** Losing events on a crash between a DB write and a publish, by writing the event in the same transaction and relaying it separately

Either both commit or neither does.

### [08 · API Monetization &amp; Developer Portals](level-4/08-api-monetization.md)

**Q149.** Why record usage metering asynchronously?

**Answer:** So metering adds no latency to the actual response

It uses the event-driven pattern from module 7.

**Q150.** How does quota exhaustion differ from rate limiting?

**Answer:** It's a billing boundary that resets on the billing cycle, and the fix is upgrading a plan rather than waiting 60 seconds

Both can return 429, but the semantics differ.

**Q151.** Why offer a real-time usage dashboard?

**Answer:** It avoids a customer discovering a huge bill at month end with no warning

It returns calls_used, calls_included and projected overage.

**Q152.** What does a developer portal bundle?

**Answer:** Sign-up and key generation, docs, usage dashboard, billing and plan upgrades, and support links

It makes a paid API self-service.

### [09 · Zero-Downtime Deployments &amp; Migrations](level-4/09-zero-downtime-migrations.md)

**Q153.** What is the danger window in a rolling deploy?

**Answer:** Old and new code serve traffic simultaneously, so a change incompatible with the previous version breaks requests

It lasts seconds to minutes.

**Q154.** What are the three phases of expand-contract?

**Answer:** Expand (add the new column and write both), migrate (backfill and switch reads), contract (drop the old column)

Each is an independent, individually safe deploy.

**Q155.** Why backfill all rows before switching reads?

**Answer:** Otherwise old rows read as null

Switch reads only after every row is backfilled.

**Q156.** How does a rolling deployment protect against a bad build?

**Answer:** New instances are health-checked before receiving traffic, and a failing one halts the rollout

The load balancer only routes to healthy instances.

### [10 · Capstone Project](level-4/10-capstone-project.md)

**Q157.** What is the capstone, OrderFlow?

**Answer:** A design exercise: a public, monetized, multi-region order/inventory/payments API at tens of thousands of requests per second

Work through each decision as in a real architecture review and justify it.

**Q158.** What is expected of each decision?

**Answer:** Be able to justify it, as in an architecture review

It pulls together every module from all four levels.

**Q159.** What do the capstone's requirements span?

**Answer:** All four levels: REST design, pagination, auth, webhooks, gateways, security, performance, governance, multi-region, events, monetization and zero-downtime releases

The platform is monetized as a paid product.

**Q160.** What scale does the brief set?

**Answer:** Tens of thousands of requests per second across multiple continents

That scale drives the multi-region and performance choices.

## Mixed questions across all levels

**Q161.** A mobile client POSTs /payments, the network times out, and it retries the same request. How should the API prevent a double charge?

**Answer:** Require an Idempotency-Key header; return the stored original result when the key repeats

Verified by simulation: the second call with the same key returns the original result (status 200 versus 201 for a new key) and creates no second record.

**Q162.** A client's list request used offset pagination: page 1 returned ids [1, 2, 3]. Before page 2, a new item was inserted at the front. What does page 2 (offset 3) return?

**Answer:** [3, 4, 5]: item 3 appears twice (page drift)

Offsets shift when rows are inserted or deleted. Cursor pagination (WHERE id &gt; last_id) avoids this.

**Q163.** A token bucket has capacity 3 and refills 1 token per second. Requests arrive at t = 0, 0, 0, 0, 1, 1, 2 seconds. Which are allowed (1) or rejected (0)?

**Answer:** 1, 1, 1, 0, 1, 0, 1

The burst of three drains the bucket, the 4th is rejected, then one token returns per second. Verified by simulation.

**Q164.** `GET /v1/orders/999` with user A's valid token returns user B's order. Which vulnerability is this, and what is the fix?

**Answer:** Broken Object Level Authorization (BOLA); filter every lookup by the authenticated user as well as the ID

Checking only that a token is valid isn't enough. Returning 404 for others' resources also avoids confirming they exist.

**Q165.** A client sends `If-None-Match: "b7e2f9"` and the resource hasn't changed. What does the server return?

**Answer:** 304 Not Modified with no body

The client keeps using its cached copy. If-Match (not If-None-Match) is for optimistic concurrency on writes.

**Q166.** A client sends `If-Match: "old-etag"` on a PUT but the resource has changed since. What should the server return?

**Answer:** 412 Precondition Failed, so the client doesn't overwrite someone else's change

If-Match gives optimistic concurrency and prevents lost updates.

**Q167.** You must rename the `total` column to `total_amount` on a live API with rolling deploys. What is the safe order?

**Answer:** Expand (add column, write both), migrate (backfill, switch reads), contract (drop the old column), each as a separate deploy

During a rolling deploy old and new code run together, so every step must be compatible with both versions.

**Q168.** An endpoint accepts `?sort=` and builds `ORDER BY` from it. Which fix is correct?

**Answer:** Validate against an allowlist of known columns (and use parameterised queries elsewhere)

Never interpolate user input into SQL, even for ORDER BY.

**Q169.** A public API returns `429 Too Many Requests` with `RateLimit-Reset: 37`. What should a well-behaved client do?

**Answer:** Wait about 37 seconds (the time until the window resets) before retrying

RateLimit-Reset here means seconds until the window or bucket resets. A Retry-After header may also be present.

**Q170.** A webhook receiver gets a POST with an X-Webhook-Signature header. What is the right verification?

**Answer:** Recompute the HMAC-SHA256 of the raw body with the shared secret and compare in constant time

Verify the raw bytes. Parsing and re-serialising can change them, and == leaks timing information.

**Q171.** A report takes two minutes to generate. Which design is best?

**Answer:** POST returns 202 Accepted with a job id; the client polls the job or receives a webhook

Long work belongs in a background job, not in the request.

**Q172.** Which change to an existing v1 response is breaking?

**Answer:** Renaming a field or changing its type (for example 42 to "42")

Rule of thumb: if unmodified client code could start erroring, it's breaking.

**Q173.** Which headers help clients prepare for retiring v1?

**Answer:** Deprecation and Sunset (with a Link to the migration guide)

They let tooling detect deprecated usage and the removal date automatically.

**Q174.** An orders service needs the customer's name, which lives in another team's database. What is the microservices-correct approach?

**Answer:** Call the users service's API (or consume its events); never query its tables

Database-per-service keeps services independently deployable.

**Q175.** After the orders service writes the order row, it crashes before publishing `order.confirmed`. Which pattern avoids lost events?

**Answer:** The transactional outbox: write the event row in the same DB transaction and relay it separately

Both writes commit or neither does; a relay publishes from the outbox.

**Q176.** A broker delivers the same event twice after a rebalance. What must the consumer be?

**Answer:** Idempotent: it records processed event ids and skips duplicates

Most brokers give at-least-once delivery.

**Q177.** A browser app on https://app.example.com calls your API. Which CORS setup is right?

**Answer:** Allow that explicit origin (with Vary: Origin), not a reflected or wildcard origin with credentials

Reflecting any origin recreates the hole the same-origin policy is protecting against.

**Q178.** Which protocol suits internal service-to-service calls with strong contracts and streaming, and which suits public browser-facing APIs?

**Answer:** gRPC internally; REST/JSON publicly

Browsers can't natively speak gRPC, and public integrators expect REST.

## Practical coding questions

### Cursor pagination

Implement list_books(limit, cursor=None) over BOOKS (ids 1..7). The cursor is the base64 (urlsafe) of JSON {"id": last_id}. Return {data: ids, has_more, next_cursor}. Fetch limit+1 rows to know whether more exist. Then page through with limit 3.

Starter code:

```
import base64, json
BOOKS = [{"id": i, "title": f"Book {i}"} for i in range(1, 8)]

def encode_cursor(last_id):
    return base64.urlsafe_b64encode(json.dumps({"id": last_id}).encode()).decode()

def decode_cursor(cursor):
    return json.loads(base64.urlsafe_b64decode(cursor.encode()))["id"]

def list_books(limit, cursor=None):
    # your code
    pass

p1 = list_books(3)
p2 = list_books(3, p1["next_cursor"])
p3 = list_books(3, p2["next_cursor"])
print(p1["data"], p1["has_more"], decode_cursor(p1["next_cursor"]))
print(p2["data"], p2["has_more"])
print(p3["data"], p3["has_more"], p3["next_cursor"])
```

Expected output:

```text
[1, 2, 3] True 3
[4, 5, 6] True
[7] False None
```

??? success "Show a solution"

    ```
    import base64, json
    BOOKS = [{"id": i, "title": f"Book {i}"} for i in range(1, 8)]

    def encode_cursor(last_id):
        return base64.urlsafe_b64encode(json.dumps({"id": last_id}).encode()).decode()

    def decode_cursor(cursor):
        return json.loads(base64.urlsafe_b64decode(cursor.encode()))["id"]

    def list_books(limit, cursor=None):
        after = decode_cursor(cursor) if cursor else 0
        rows = [b for b in BOOKS if b["id"] > after][: limit + 1]
        page = rows[:limit]
        has_more = len(rows) > limit
        return {"data": [b["id"] for b in page], "has_more": has_more,
                "next_cursor": encode_cursor(page[-1]["id"]) if has_more else None}

    p1 = list_books(3)
    p2 = list_books(3, p1["next_cursor"])
    p3 = list_books(3, p2["next_cursor"])
    print(p1["data"], p1["has_more"], decode_cursor(p1["next_cursor"]))
    print(p2["data"], p2["has_more"])
    print(p3["data"], p3["has_more"], p3["next_cursor"])
    ```

    Fetching limit+1 rows tells you whether another page exists without a count query. The cursor encodes the last id, so the next page is WHERE id &gt; last_id, which stays consistent when rows are inserted or deleted. Verified by running the solution: output matches.

### Verify a webhook signature

Write sign(body) that returns "sha256=" + the hex HMAC-SHA256 of the raw body using the shared secret, and verify(body, header) using a constant-time comparison. Print the signature length and prefix, then verify a good signature, a tampered body and a wrong signature.

Starter code:

```
import hmac, hashlib
SECRET = "whsec_abc123"

def sign(body: bytes) -> str:
    # your code
    pass

def verify(body: bytes, header: str) -> bool:
    # your code
    pass

body = b'{"event": "book.added", "id": "evt_1"}'
good = sign(body)
print(len(good), good[:7])
print(verify(body, good), verify(body + b" ", good), verify(body, "sha256=" + "0" * 64))
```

Expected output:

```text
71 sha256=
True False False
```

??? success "Show a solution"

    ```
    import hmac, hashlib
    SECRET = "whsec_abc123"
    def sign(body: bytes) -> str:
        return "sha256=" + hmac.new(SECRET.encode(), body, hashlib.sha256).hexdigest()
    def verify(body: bytes, header: str) -> bool:
        return hmac.compare_digest(sign(body), header)
    body = b'{"event": "book.added", "id": "evt_1"}'
    good = sign(body)
    print(len(good), good[:7])
    print(verify(body, good), verify(body + b" ", good), verify(body, "sha256=" + "0" * 64))
    ```

    The hex SHA-256 digest is 64 characters, plus the 7-character prefix, so 71. Any change to the body changes the signature, and compare_digest avoids timing leaks. Verified by running the solution: output matches.

---

More free practice: the [REST API Mastery Path](index.md) curriculum, the [Interview Prep Pack](https://learnmasterypath.com/interview-pack/) and the [free resume ATS check](https://learnmasterypath.com/resume/).
