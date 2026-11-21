---
title: "Final Quiz"
description: "Test what you have learned across all four levels of REST API Mastery Path with a 18-question final quiz."
---

# Final Quiz

18 questions drawn from all four levels of this course. You need **70% (13 out of 18)** to pass —
you can retake it as many times as you like. Your best result is saved in this browser.

<div id="mp-quiz-root"></div>

<script type="application/json" id="mp-quiz-data">
[
  {
    "question": "A mobile client POSTs /payments, the network times out, and it retries the same request. How should the API prevent a double charge?",
    "options": [
      "Reject all retries with 429",
      "Make POST idempotent by changing it to GET",
      "Ask the user to wait a minute",
      "Require an Idempotency-Key header; return the stored original result when the key repeats"
    ],
    "correctIndex": 3,
    "explanation": "Verified by simulation: the second call with the same key returns the original result (status 200 versus 201 for a new key) and creates no second record."
  },
  {
    "question": "A client's list request used offset pagination: page 1 returned ids [1, 2, 3]. Before page 2, a new item was inserted at the front. What does page 2 (offset 3) return?",
    "options": [
      "[3, 4, 5]: item 3 appears twice (page drift)",
      "An error",
      "[0, 1, 2]",
      "[4, 5, 6]"
    ],
    "correctIndex": 0,
    "explanation": "Offsets shift when rows are inserted or deleted. Cursor pagination (WHERE id > last_id) avoids this."
  },
  {
    "question": "A token bucket has capacity 3 and refills 1 token per second. Requests arrive at t = 0, 0, 0, 0, 1, 1, 2 seconds. Which are allowed (1) or rejected (0)?",
    "options": [
      "1, 1, 1, 0, 0, 0, 0",
      "1, 1, 1, 1, 1, 1, 1",
      "1, 0, 0, 0, 1, 0, 0",
      "1, 1, 1, 0, 1, 0, 1"
    ],
    "correctIndex": 3,
    "explanation": "The burst of three drains the bucket, the 4th is rejected, then one token returns per second. Verified by simulation."
  },
  {
    "question": "`GET /v1/orders/999` with user A's valid token returns user B's order. Which vulnerability is this, and what is the fix?",
    "options": [
      "CORS misconfiguration; add Vary: Origin",
      "SQL injection; use parameterised queries",
      "CSRF; add a state parameter",
      "Broken Object Level Authorization (BOLA); filter every lookup by the authenticated user as well as the ID"
    ],
    "correctIndex": 3,
    "explanation": "Checking only that a token is valid isn't enough. Returning 404 for others' resources also avoids confirming they exist."
  },
  {
    "question": "A client sends `If-None-Match: \"b7e2f9\"` and the resource hasn't changed. What does the server return?",
    "options": [
      "304 Not Modified with no body",
      "200 with the full body",
      "204 No Content",
      "412 Precondition Failed"
    ],
    "correctIndex": 0,
    "explanation": "The client keeps using its cached copy. If-Match (not If-None-Match) is for optimistic concurrency on writes."
  },
  {
    "question": "A client sends `If-Match: \"old-etag\"` on a PUT but the resource has changed since. What should the server return?",
    "options": [
      "412 Precondition Failed, so the client doesn't overwrite someone else's change",
      "201 Created",
      "304 Not Modified",
      "200 and overwrite"
    ],
    "correctIndex": 0,
    "explanation": "If-Match gives optimistic concurrency and prevents lost updates."
  },
  {
    "question": "You must rename the `total` column to `total_amount` on a live API with rolling deploys. What is the safe order?",
    "options": [
      "Drop and re-create the table",
      "Rename it in one migration",
      "Stop the API for the migration window",
      "Expand (add column, write both), migrate (backfill, switch reads), contract (drop the old column), each as a separate deploy"
    ],
    "correctIndex": 3,
    "explanation": "During a rolling deploy old and new code run together, so every step must be compatible with both versions."
  },
  {
    "question": "An endpoint accepts `?sort=` and builds `ORDER BY` from it. Which fix is correct?",
    "options": [
      "Trust the ORM",
      "Strip quotes from the input",
      "Escape semicolons only",
      "Validate against an allowlist of known columns (and use parameterised queries elsewhere)"
    ],
    "correctIndex": 3,
    "explanation": "Never interpolate user input into SQL, even for ORDER BY."
  },
  {
    "question": "A public API returns `429 Too Many Requests` with `RateLimit-Reset: 37`. What should a well-behaved client do?",
    "options": [
      "Switch to a different API key",
      "Retry 37 times",
      "Retry immediately",
      "Wait about 37 seconds (the time until the window resets) before retrying"
    ],
    "correctIndex": 3,
    "explanation": "RateLimit-Reset here means seconds until the window or bucket resets. A Retry-After header may also be present."
  },
  {
    "question": "A webhook receiver gets a POST with an X-Webhook-Signature header. What is the right verification?",
    "options": [
      "Recompute the HMAC-SHA256 of the raw body with the shared secret and compare in constant time",
      "Check that the request came from a known IP only",
      "Parse the JSON and trust the event id",
      "Compare the signature with == on the parsed body"
    ],
    "correctIndex": 0,
    "explanation": "Verify the raw bytes. Parsing and re-serialising can change them, and == leaks timing information."
  },
  {
    "question": "A report takes two minutes to generate. Which design is best?",
    "options": [
      "Return 200 with an empty body",
      "POST returns 202 Accepted with a job id; the client polls the job or receives a webhook",
      "Hold the connection open for two minutes",
      "Return 504 and ask the client to retry"
    ],
    "correctIndex": 1,
    "explanation": "Long work belongs in a background job, not in the request."
  },
  {
    "question": "Which change to an existing v1 response is breaking?",
    "options": [
      "Renaming a field or changing its type (for example 42 to \"42\")",
      "Adding a new optional query parameter",
      "Adding an optional field",
      "Adding a new endpoint"
    ],
    "correctIndex": 0,
    "explanation": "Rule of thumb: if unmodified client code could start erroring, it's breaking."
  },
  {
    "question": "Which headers help clients prepare for retiring v1?",
    "options": [
      "Accept and Content-Type",
      "Deprecation and Sunset (with a Link to the migration guide)",
      "Allow and Vary",
      "ETag and Last-Modified"
    ],
    "correctIndex": 1,
    "explanation": "They let tooling detect deprecated usage and the removal date automatically."
  },
  {
    "question": "An orders service needs the customer's name, which lives in another team's database. What is the microservices-correct approach?",
    "options": [
      "Add a read-only DB connection to users_db",
      "Call the users service's API (or consume its events); never query its tables",
      "Copy the tables nightly by hand",
      "Join across databases"
    ],
    "correctIndex": 1,
    "explanation": "Database-per-service keeps services independently deployable."
  },
  {
    "question": "After the orders service writes the order row, it crashes before publishing `order.confirmed`. Which pattern avoids lost events?",
    "options": [
      "Retry the whole HTTP request",
      "The transactional outbox: write the event row in the same DB transaction and relay it separately",
      "Publish first, then write the row",
      "Ignore it"
    ],
    "correctIndex": 1,
    "explanation": "Both writes commit or neither does; a relay publishes from the outbox."
  },
  {
    "question": "A broker delivers the same event twice after a rebalance. What must the consumer be?",
    "options": [
      "Single-threaded",
      "Stateless and ignoring ids",
      "Exactly-once by default",
      "Idempotent: it records processed event ids and skips duplicates"
    ],
    "correctIndex": 3,
    "explanation": "Most brokers give at-least-once delivery."
  },
  {
    "question": "A browser app on https://app.example.com calls your API. Which CORS setup is right?",
    "options": [
      "Access-Control-Allow-Origin: * with credentials",
      "Allow that explicit origin (with Vary: Origin), not a reflected or wildcard origin with credentials",
      "Disable CORS",
      "Reflect whatever Origin arrives"
    ],
    "correctIndex": 1,
    "explanation": "Reflecting any origin recreates the hole the same-origin policy is protecting against."
  },
  {
    "question": "Which protocol suits internal service-to-service calls with strong contracts and streaming, and which suits public browser-facing APIs?",
    "options": [
      "SOAP for both",
      "REST internally; gRPC publicly",
      "GraphQL internally only",
      "gRPC internally; REST/JSON publicly"
    ],
    "correctIndex": 3,
    "explanation": "Browsers can't natively speak gRPC, and public integrators expect REST."
  }
]
</script>

## Coding challenges

Try these on your own computer, then compare your output. Nothing you write is sent anywhere.

### Cursor pagination

Implement list_books(limit, cursor=None) over BOOKS (ids 1..7). The cursor is the base64 (urlsafe) of JSON {"id": last_id}. Return {data: ids, has_more, next_cursor}. Fetch limit+1 rows to know whether more exist. Then page through with limit 3.

**Starter code**

```python
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

**Expected output**

```text
[1, 2, 3] True 3
[4, 5, 6] True
[7] False None
```

<details><summary>Show solution and explanation</summary>

<pre><code>import base64, json
BOOKS = [{&quot;id&quot;: i, &quot;title&quot;: f&quot;Book {i}&quot;} for i in range(1, 8)]

def encode_cursor(last_id):
    return base64.urlsafe_b64encode(json.dumps({&quot;id&quot;: last_id}).encode()).decode()

def decode_cursor(cursor):
    return json.loads(base64.urlsafe_b64decode(cursor.encode()))[&quot;id&quot;]

def list_books(limit, cursor=None):
    after = decode_cursor(cursor) if cursor else 0
    rows = [b for b in BOOKS if b[&quot;id&quot;] &gt; after][: limit + 1]
    page = rows[:limit]
    has_more = len(rows) &gt; limit
    return {&quot;data&quot;: [b[&quot;id&quot;] for b in page], &quot;has_more&quot;: has_more,
            &quot;next_cursor&quot;: encode_cursor(page[-1][&quot;id&quot;]) if has_more else None}

p1 = list_books(3)
p2 = list_books(3, p1[&quot;next_cursor&quot;])
p3 = list_books(3, p2[&quot;next_cursor&quot;])
print(p1[&quot;data&quot;], p1[&quot;has_more&quot;], decode_cursor(p1[&quot;next_cursor&quot;]))
print(p2[&quot;data&quot;], p2[&quot;has_more&quot;])
print(p3[&quot;data&quot;], p3[&quot;has_more&quot;], p3[&quot;next_cursor&quot;])
</code></pre>

<p>Fetching limit+1 rows tells you whether another page exists without a count query. The cursor encodes the last id, so the next page is WHERE id &gt; last_id, which stays consistent when rows are inserted or deleted. Verified by running the solution: output matches.</p>
</details>

### Verify a webhook signature

Write sign(body) that returns "sha256=" + the hex HMAC-SHA256 of the raw body using the shared secret, and verify(body, header) using a constant-time comparison. Print the signature length and prefix, then verify a good signature, a tampered body and a wrong signature.

**Starter code**

```python
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

**Expected output**

```text
71 sha256=
True False False
```

<details><summary>Show solution and explanation</summary>

<pre><code>import hmac, hashlib
SECRET = &quot;whsec_abc123&quot;
def sign(body: bytes) -&gt; str:
    return &quot;sha256=&quot; + hmac.new(SECRET.encode(), body, hashlib.sha256).hexdigest()
def verify(body: bytes, header: str) -&gt; bool:
    return hmac.compare_digest(sign(body), header)
body = b&#x27;{&quot;event&quot;: &quot;book.added&quot;, &quot;id&quot;: &quot;evt_1&quot;}&#x27;
good = sign(body)
print(len(good), good[:7])
print(verify(body, good), verify(body + b&quot; &quot;, good), verify(body, &quot;sha256=&quot; + &quot;0&quot; * 64))
</code></pre>

<p>The hex SHA-256 digest is 64 characters, plus the 7-character prefix, so 71. Any change to the body changes the signature, and compare_digest avoids timing leaks. Verified by running the solution: output matches.</p>
</details>
