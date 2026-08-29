# 05 · Query Params vs Path Params

REST APIs pass extra data through three places: path parameters, query
parameters, and the body. Knowing which one to use for a given piece of data
is a quick tell for whether an API is well-designed.

## Path parameters — identifying a specific resource

A path parameter is a variable segment of the URL that identifies *which*
resource you mean. It's part of the resource's identity.

```
GET /books/{bookId}
GET /books/17
```

```
GET /authors/{authorId}/books/{bookId}
GET /authors/3/books/17
```

Rule of thumb: **if removing the value would leave you unable to say which
resource you mean, it's a path parameter.**

## Query parameters — filtering, sorting, paginating, or optional modifiers

A query parameter narrows, shapes, or modifies a request against a
*collection* — it doesn't identify a single resource.

```
GET /books?author=Asimov
GET /books?published_after=1960&published_before=1970
GET /books?sort=-published_year
GET /books?page=2&limit=20
GET /books?fields=title,author
```

Rule of thumb: **if removing the value still leaves a meaningful request
(just a broader/differently-shaped one), it's a query parameter.**

## Side-by-side

```
GOOD: GET /books/17                          (17 identifies the resource)
BAD:  GET /books?id=17                       (17 IS the resource, not a filter)

GOOD: GET /books?author=Asimov               (filters a collection)
BAD:  GET /books/author/Asimov               (looks like a resource path, isn't one)

GOOD: GET /orders/482/items                  (482 identifies which order)
BAD:  GET /items?orderId=482                 (defensible too, but less RESTful — treats
                                                 "items of order 482" as a top-level
                                                 collection with a filter rather than a
                                                 clearly nested resource)
```

That last example is genuinely a judgment call some APIs make differently —
both patterns exist in production. Nesting (`/orders/482/items`) communicates
the relationship more clearly; a flat collection with a filter
(`/items?order_id=482`) is sometimes preferred when items are also
independently addressable outside the context of an order.

## Combining both

Most real endpoints use both together:

```
GET /authors/{authorId}/books?sort=-published_year&limit=10

Path param:  authorId = 3       (which author's books)
Query params: sort, limit        (how to shape that specific list)
```

## URL-encoding query parameters

Query parameter values must be URL-encoded if they contain reserved
characters (spaces, `&`, `=`, `#`, non-ASCII text).

```
Raw value:    Science Fiction & Fantasy
Encoded:      Science%20Fiction%20%26%20Fantasy

GET /books?genre=Science%20Fiction%20%26%20Fantasy
```

`curl` and most HTTP client libraries handle this encoding automatically if
you build the query string with their helpers rather than concatenating raw
strings yourself:

```bash
curl -G https://api.example.com/books \
  --data-urlencode "genre=Science Fiction & Fantasy" \
  --data-urlencode "sort=-published_year"
```

`-G` tells curl to turn `--data-urlencode` arguments into a properly-encoded
query string on a `GET` request (without `-G`, `-d`/`--data-urlencode` would
send a `POST` body instead). This produces:

```
GET /books?genre=Science%20Fiction%20%26%20Fantasy&sort=-published_year HTTP/1.1
```

## Arrays and repeated keys in query strings

There's no single standard for representing a list in a query string —
common conventions:

```
?tag=fiction&tag=classic          (repeated key — most common, works with most frameworks)
?tags=fiction,classic             (comma-separated single value)
?tags[]=fiction&tags[]=classic    (PHP/Rails-style bracket notation)
```

Check the specific API's docs — all three are used in the wild, and getting
it wrong silently produces an empty or partial filter rather than an error
in many implementations, which makes this a common source of "why isn't my
filter working" bugs.

## Optional vs. required, and sensible defaults

Query parameters are almost always optional with sensible defaults; path
parameters are almost never optional (a route without one wouldn't match).

```
GET /books                    -> defaults: page=1, limit=20, sort=id
GET /books?page=2             -> limit and sort still default
GET /books?limit=100&sort=-year
```

A well-documented API states its defaults explicitly rather than leaving
clients to guess.

## Exercise

For a `GET` request against a hypothetical `/orders` API, decide whether
each piece of data belongs as a path parameter, a query parameter, or in the
request body, and explain why:

1. The ID of a specific order you want to retrieve.
2. Filtering orders to only those placed by customer `842`.
3. Only returning orders with `status=shipped`.
4. Requesting page 3 of the results, 25 per page.
5. The new shipping address when updating an order.
6. Sorting orders by `total_amount` descending.

Then write the full request line (method + path + query string, properly
encoded) that combines items 2, 3, 4, and 6 into a single `GET` request.
