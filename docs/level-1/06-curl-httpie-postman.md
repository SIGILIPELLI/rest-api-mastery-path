# 06 · Using curl, httpie & Postman

You'll spend a large fraction of your API-development life issuing manual
requests to explore, test, and debug. This module covers the three most
common tools: `curl` (universal, scriptable), `httpie` (friendlier syntax),
and Postman (GUI, great for saved collections and teams).

## curl

`curl` ships on essentially every Linux/macOS system and is the lowest
common denominator — if you can only learn one tool, learn this one.

### Basic GET

```bash
curl https://api.example.com/books/17
```

Prints only the response body by default.

### See status code and headers with -i / -v

```bash
curl -i https://api.example.com/books/17
```

```http
HTTP/1.1 200 OK
Content-Type: application/json
Content-Length: 63

{"id": 17, "title": "Dune", "author": "Frank Herbert"}
```

`-i` (include) prepends the response headers. `-v` (verbose) goes further,
also showing the request curl sent (see module 4).

### POST with a JSON body

```bash
curl -i -X POST https://api.example.com/books \
  -H "Content-Type: application/json" \
  -d '{"title": "Foundation", "author": "Isaac Asimov"}'
```

`-X POST` sets the method (curl infers `POST` automatically once you use
`-d`, but being explicit is clearer). `-H` adds a header; `-d` sets the body.

### PATCH / PUT / DELETE

```bash
curl -X PATCH https://api.example.com/books/17 \
  -H "Content-Type: application/json" \
  -d '{"year": 1966}'

curl -X PUT https://api.example.com/books/17 \
  -H "Content-Type: application/json" \
  -d '{"title": "Dune", "author": "Frank Herbert", "year": 1966}'

curl -X DELETE https://api.example.com/books/17
```

### Sending an auth header

```bash
curl https://api.example.com/me \
  -H "Authorization: Bearer eyJhbGciOiJIUzI1NiJ9.eyJzdWIiOiI0MiJ9.xyz"
```

### Query strings safely

```bash
curl -G https://api.example.com/books \
  --data-urlencode "author=Ursula K. Le Guin" \
  --data-urlencode "sort=-year"
```

### Saving a response to a file, and only printing the status code

```bash
curl -o response.json -w "%{http_code}\n" https://api.example.com/books/17
```

`-w "%{http_code}\n"` is invaluable in shell scripts that need to check
success/failure without parsing the body.

## httpie

`httpie` (the `http` command) reads more like plain English and pretty-prints
JSON with syntax highlighting by default — many people prefer it for
interactive exploration, though curl remains more universal for scripting
and CI since it's preinstalled almost everywhere.

### Basic GET

```bash
http GET api.example.com/books/17
```

(the scheme defaults to `http://`; add `https://` explicitly for HTTPS, or
use `https api.example.com/...` as shorthand)

### POST with JSON — no manual escaping needed

```bash
http POST api.example.com/books title="Foundation" author="Isaac Asimov"
```

httpie infers `Content-Type: application/json` automatically and builds the
JSON body from `key=value` arguments — no need to hand-write a JSON string.
Use `key:=value` for non-string JSON values:

```bash
http POST api.example.com/books \
  title="Foundation" \
  year:=1951 \
  in_stock:=true \
  tags:='["sci-fi", "classic"]'
```

### Custom headers and auth

```bash
http GET api.example.com/me "Authorization:Bearer eyJhbGciOiJIUzI1NiJ9..."

# Or with the built-in auth helper for Basic auth:
http -a username:password GET api.example.com/me
```

### Query parameters

```bash
http GET api.example.com/books author=="Ursula K. Le Guin" sort==-year
```

Note `==` for query parameters vs. `=`/`:=` for body fields — this is
httpie's key syntax distinction.

## curl vs httpie — when to use which

| | curl | httpie |
|---|---|---|
| Preinstalled everywhere | Yes | No (needs `pip install httpie` or a package manager) |
| Scripting / CI | Best choice | Works, less common in scripts |
| Interactive exploration | Verbose syntax | Faster to type, colorized output |
| JSON body construction | Manual (hand-write JSON string) | Automatic from key=value args |

## Postman

Postman is a GUI application built around **collections** — saved, organized
groups of requests you can share with a team, run in sequence, and attach
tests to.

### Core workflow

1. Create a new request: pick the method (`GET`/`POST`/...), enter the URL.
2. Add headers in the **Headers** tab (e.g. `Authorization`, `Content-Type`).
3. For a body, go to the **Body** tab, select **raw** + **JSON**, and type
   the JSON payload.
4. Click **Send** — Postman shows status code, response time, response size,
   headers, and a pretty-printed body.
5. Save the request into a **Collection** so it's reusable and shareable.

### Environments and variables

Postman lets you define variables like `{{base_url}}` or `{{auth_token}}`
per **environment** (e.g. "local," "staging," "production"), so the same
collection of requests can target different servers just by switching a
dropdown:

```
{{base_url}}/books/17
Authorization: Bearer {{auth_token}}
```

### Why teams like it over curl for shared work

- A whole collection of requests (covering an entire API) can be exported as
  JSON and version-controlled, or shared via a Postman workspace.
- Postman can auto-generate equivalent `curl` commands from any request
  (**Code** button) — handy for turning exploratory GUI work into a script.
- Built-in test scripts (JavaScript) can assert on status codes and response
  shape, turning a manual collection into a lightweight automated smoke test
  suite.

## Choosing a tool

- **Quick one-off check, or a CI script**: `curl`.
- **Fast interactive exploration, especially with JSON bodies**: `httpie`.
- **Organizing dozens of endpoints across a team, with saved environments**:
  Postman (or its open-source-friendly alternatives like Insomnia).

All three send exactly the same HTTP over the wire — pick based on your
workflow, not on any difference in what's possible.

## Exercise

1. Using `curl`, write the command to `POST` a new resource to
   `https://api.example.com/comments` with a JSON body `{"post_id": 17,
   "text": "Great article!"}`, including the correct `Content-Type` header,
   and printing the response headers.
2. Rewrite that same request using `httpie` syntax.
3. Using `curl`, write a `GET` request to `/search` with query parameters
   `q=rest api` (note the space) and `limit=5`, letting curl handle the
   URL-encoding.
4. In one sentence, explain what advantage a Postman **collection** gives a
   team of 5 backend engineers that a folder of curl one-liners in a text
   file doesn't.
