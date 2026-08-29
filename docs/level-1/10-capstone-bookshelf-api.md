# 10 · Capstone — Bookshelf API Spec

Time to put everything from Level 1 together: design and document a
complete, versioned REST API spec for a small "bookshelf" application — a
place where a user tracks books they own and organizes them into shelves
(e.g. "Currently Reading," "Sci-Fi Favorites"). You won't need to write a
server to complete this — the deliverable is a precise, well-documented API
design, exactly as you'd hand it to another engineer to implement, or as
you'd document a real API for consumers.

## The resources

- **Books** — `id`, `title`, `author`, `isbn`, `published_year`.
- **Shelves** — `id`, `name`, `created_at`.
- **Shelf items** — the relationship "this book is on this shelf," with its
  own `added_at` timestamp.

## 1. URL design (module 3)

```
GET    /v1/books                        list all books
POST   /v1/books                        create a book
GET    /v1/books/{bookId}               get one book
PUT    /v1/books/{bookId}               replace a book
PATCH  /v1/books/{bookId}               partially update a book
DELETE /v1/books/{bookId}               delete a book

GET    /v1/shelves                      list the current user's shelves
POST   /v1/shelves                      create a shelf
GET    /v1/shelves/{shelfId}            get one shelf
PATCH  /v1/shelves/{shelfId}            rename a shelf
DELETE /v1/shelves/{shelfId}            delete a shelf

GET    /v1/shelves/{shelfId}/books      list books on a shelf
POST   /v1/shelves/{shelfId}/books      add a book to a shelf
DELETE /v1/shelves/{shelfId}/books/{bookId}   remove a book from a shelf
```

Notice the `/v1/` prefix (module 9), plural resource names, path parameters
identifying specific resources, and the nested `shelves/{shelfId}/books`
relationship (module 3) rather than inventing a separate "shelf-item ID"
concept the client would need to track.

## 2. Authentication (module 7)

All endpoints require a bearer token, obtained via a (not-detailed-here)
login endpoint:

```
Authorization: Bearer <token>
```

Requests without a valid token receive:

```http
HTTP/1.1 401 Unauthorized
Content-Type: application/json

{ "error": { "code": "unauthorized", "message": "A valid bearer token is required." } }
```

## 3. Full request/response examples

### Create a book

```bash
curl -i -X POST https://api.example.com/v1/books \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"title": "Dune", "author": "Frank Herbert", "isbn": "9780441013593", "published_year": 1965}'
```

```http
HTTP/1.1 201 Created
Content-Type: application/json
Location: /v1/books/101

{
  "id": 101,
  "title": "Dune",
  "author": "Frank Herbert",
  "isbn": "9780441013593",
  "published_year": 1965
}
```

### List books, with filtering and sorting

```bash
curl "https://api.example.com/v1/books?author=Frank+Herbert&sort=-published_year" \
  -H "Authorization: Bearer $TOKEN"
```

```http
HTTP/1.1 200 OK
Content-Type: application/json

{
  "data": [
    { "id": 101, "title": "Dune", "author": "Frank Herbert", "isbn": "9780441013593", "published_year": 1965 }
  ]
}
```

### Create a shelf

```bash
curl -i -X POST https://api.example.com/v1/shelves \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"name": "Sci-Fi Favorites"}'
```

```http
HTTP/1.1 201 Created
Content-Type: application/json
Location: /v1/shelves/9

{ "id": 9, "name": "Sci-Fi Favorites", "created_at": "2026-08-29T10:00:00Z" }
```

### Add a book to a shelf

```bash
curl -i -X POST https://api.example.com/v1/shelves/9/books \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"bookId": 101}'
```

```http
HTTP/1.1 201 Created
Content-Type: application/json

{ "bookId": 101, "shelfId": 9, "added_at": "2026-08-29T10:05:00Z" }
```

### Adding a book that's already on the shelf (conflict)

```bash
curl -i -X POST https://api.example.com/v1/shelves/9/books \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"bookId": 101}'
```

```http
HTTP/1.1 409 Conflict
Content-Type: application/json

{ "error": { "code": "already_on_shelf", "message": "Book 101 is already on shelf 9." } }
```

### Removing a book from a shelf

```bash
curl -i -X DELETE https://api.example.com/v1/shelves/9/books/101 \
  -H "Authorization: Bearer $TOKEN"
```

```http
HTTP/1.1 204 No Content
```

### Requesting a nonexistent book

```bash
curl -i https://api.example.com/v1/books/9999 \
  -H "Authorization: Bearer $TOKEN"
```

```http
HTTP/1.1 404 Not Found
Content-Type: application/json

{ "error": { "code": "not_found", "message": "No book exists with id 9999." } }
```

### Validation error creating a book

```bash
curl -i -X POST https://api.example.com/v1/books \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"title": "", "author": "Frank Herbert"}'
```

```http
HTTP/1.1 422 Unprocessable Entity
Content-Type: application/json

{
  "error": {
    "code": "validation_failed",
    "message": "The request could not be processed due to validation errors.",
    "details": [
      { "field": "title", "issue": "must not be empty" }
    ]
  }
}
```

## 4. Error code reference (module 8)

| `error.code` | Status | Meaning |
|---|---|---|
| `unauthorized` | 401 | Missing or invalid bearer token |
| `not_found` | 404 | The requested book/shelf doesn't exist |
| `already_on_shelf` | 409 | Book is already on the target shelf |
| `validation_failed` | 422 | One or more fields failed validation |
| `internal_error` | 500 | Unexpected server error |

## 5. Versioning note (module 9)

This spec is `v1`. A future breaking change (e.g. splitting `author` into a
separate `authors` resource with its own ID, instead of a plain string)
would ship as `/v2/...`, with `/v1/...` continuing to serve its existing
shape until formally deprecated.

## Deliverable checklist

Confirm your own written spec (based on this worked example, adapted or
extended in your own words) includes:

- [ ] A full endpoint table (method + path + short description) for books,
      shelves, and shelf-book relationships.
- [ ] At least one full request/response example per endpoint, with
      realistic JSON.
- [ ] Explicit authentication requirements.
- [ ] A consistent error response shape, plus a table of the specific error
      codes your API returns.
- [ ] The version prefix in every URL, with a one-sentence note on your
      versioning strategy.

## Exercise

Extend the spec above with one additional feature of your choosing — for
example, a `GET /v1/books/{bookId}/shelves` endpoint (which shelves contain
this book), or a `notes` field a user can attach to a book. Write out its
full endpoint definition, one success response, and one realistic error
response, following the same conventions used throughout this capstone.
