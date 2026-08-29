# 01 · What Is REST? Constraints & Principles

<span class="level-badge">Level 1 · Entry</span>

REST (**RE**presentational **S**tate **T**ransfer) is an architectural style
for networked applications, described by Roy Fielding in his 2000 doctoral
dissertation. It is not a protocol, a standard, or a library — it's a set of
constraints that, when followed together, make a web API scalable, simple to
consume, and easy to evolve over time.

An API that follows these constraints is commonly called "RESTful." Most
public APIs you'll use (GitHub, Stripe, Twitter/X, Spotify) call themselves
REST APIs, though in practice few of them satisfy every constraint strictly.
Understanding the constraints still matters, because they explain *why* REST
APIs are designed the way they are.

## The six constraints

### 1. Client-server

The client (browser, mobile app, another service) and the server are
separate, independently evolvable components connected only by the
interface between them. The client doesn't need to know how the server
stores data; the server doesn't need to know how the client renders it.

```
[Client] ---- HTTP request ----> [Server]
[Client] <--- HTTP response ---- [Server]
```

This separation lets a team rewrite the entire backend (say, swap Postgres
for MongoDB) without touching the client, as long as the API contract stays
the same.

### 2. Statelessness

Every request from a client to a server must contain **all** the information
needed to understand and process it. The server must not store any client
session state between requests.

```text
BAD (stateful):
Request 1: POST /login          -> server remembers "user 42 is logged in"
Request 2: GET /my-orders       -> server looks up "who's logged in?" from memory

GOOD (stateless):
Request 1: POST /login          -> server returns a token
Request 2: GET /my-orders
           Authorization: Bearer <token>   -> server verifies token, no memory needed
```

Statelessness means any server instance can handle any request — critical
for horizontal scaling and load balancing, since you don't need "sticky
sessions" pinning a client to one server.

!!! note "Statelessness applies to the server, not your data"
    The database can absolutely have state (that's the whole point). What
    can't exist is *per-client session memory in the API server itself*.

### 3. Cacheability

Responses must explicitly (or implicitly, by convention) state whether they
are cacheable. This lets clients, or intermediaries like CDNs and proxies,
reuse a prior response for a later identical request instead of hitting the
server again — improving performance and reducing load.

```
GET /products/42
Cache-Control: public, max-age=3600
```

That header tells any well-behaved cache it can reuse this response for up
to an hour without re-asking the server. You'll cover caching headers like
`ETag` and `Cache-Control` in depth in Level 2.

### 4. Uniform interface

This is the constraint that most defines "RESTfulness." It has four
sub-parts:

- **Resource identification in requests** — resources (a user, an order, a
  product) are identified by a stable identifier, typically a URL, e.g.
  `/users/42`.
- **Manipulation of resources through representations** — a client that
  holds a *representation* of a resource (e.g. a JSON document with fields
  and links) has enough information to modify or delete it.
- **Self-descriptive messages** — each message includes enough information
  to describe how to process it (e.g. the `Content-Type: application/json`
  header tells the recipient how to parse the body).
- **Hypermedia as the engine of application state (HATEOAS)** — ideally, a
  response includes links to related actions/resources, so a client can
  navigate the API by following links rather than hardcoding URLs. Very few
  real-world APIs do this fully; you'll look at it properly in Level 2.

### 5. Layered system

A client generally cannot tell whether it's connected directly to the end
server or to an intermediary (load balancer, API gateway, cache, proxy)
along the way — and it shouldn't need to. Each layer only knows about the
layer immediately next to it.

```
Client -> CDN -> Load Balancer -> API Gateway -> App Server -> Database
```

This lets you insert caching layers, rate-limiting gateways, or
authentication proxies in front of your API without the client's code
changing at all.

### 6. Code-on-demand (optional)

The server can optionally extend client functionality by transferring
executable code — the classic example is a browser downloading and running
JavaScript. This is the only *optional* constraint; most JSON-over-HTTP APIs
don't use it at all, and that's fine.

## Why these constraints matter in practice

| Constraint | What it buys you |
|---|---|
| Client-server | Independent evolution of frontend/backend |
| Statelessness | Horizontal scalability, simpler servers, resilience to server restarts |
| Cacheability | Lower latency, less server load |
| Uniform interface | Predictable, learnable API surface |
| Layered system | Flexible infrastructure (gateways, caches, LBs) without client changes |
| Code-on-demand | Optional extensibility (mostly a browser-web concept) |

## A concrete example

Imagine a bookshelf API (you'll build a real spec for this in the Level 1
capstone). A RESTful design exposes **resources** (books, authors) via
**URLs**, manipulated with standard **HTTP methods**:

```
GET    /books        -> list books
GET    /books/17      -> get book 17
POST   /books        -> create a book
PUT    /books/17      -> replace book 17
PATCH  /books/17      -> partially update book 17
DELETE /books/17      -> delete book 17
```

Compare that to a **non**-RESTful, RPC-style design that ignores the
uniform interface constraint:

```
POST /getBooks
POST /getBookById?id=17
POST /createNewBook
POST /updateBookRecord
POST /removeBook?id=17
```

Both "work," but the first is predictable — once you know the pattern, you
can guess the URL for any resource and action. The second requires you to
memorize every endpoint name individually.

## Exercise

1. Pick an API you use often (GitHub's, Spotify's, or any public API with
   documentation). Find its docs online and identify:
   - At least 3 resource URLs it exposes.
   - Which HTTP methods it uses for each.
   - Whether responses include a `Cache-Control` header (check with the curl
     command below).
2. Run this against any public HTTPS endpoint you have access to (this
   example targets a public test API and was reasoned through, not executed
   live — status codes and headers shown are representative of what
   `httpbin.org` actually returns):

```bash
curl -i https://httpbin.org/get
```

   Expected response shape:

```http
HTTP/1.1 200 OK
Content-Type: application/json
Content-Length: 258

{
  "args": {},
  "headers": {
    "Accept": "*/*",
    "Host": "httpbin.org"
  },
  "origin": "203.0.113.42",
  "url": "https://httpbin.org/get"
}
```

3. Write one sentence for each of the six constraints explaining whether the
   API you picked satisfies it, partially satisfies it, or ignores it.
