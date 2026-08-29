# 03 · URL & Resource Design

Good URL design is what makes an API feel intuitive — a developer should be
able to guess `/orders/482/items` works before ever reading the docs.

## Nouns, not verbs

URLs identify **resources** (things), not actions. The HTTP method already
carries the action.

```
GOOD:  GET /orders/482
BAD:   GET /getOrder?id=482

GOOD:  DELETE /orders/482
BAD:   POST /deleteOrder?id=482
```

## Collections and individual resources

A consistent pattern: the plural collection, then the identifier for one
member of it.

```
GET    /books           -> the collection of all books
POST   /books           -> create a new book in the collection
GET    /books/17        -> one specific book
PUT    /books/17        -> replace book 17
PATCH  /books/17        -> partially update book 17
DELETE /books/17        -> delete book 17
```

Use plural nouns (`/books`, not `/book`) consistently — it reads naturally
for both the collection and singular forms, and avoids inconsistency debates
across the whole API surface.

## Nested resources

When a resource only makes sense in the context of a parent, nest it:

```
GET  /books/17/reviews         -> all reviews for book 17
GET  /books/17/reviews/9       -> review 9 of book 17
POST /books/17/reviews         -> add a new review to book 17
```

Avoid nesting more than 2-3 levels deep — it gets unwieldy and usually means
the "child" resource deserves its own top-level, filterable collection
instead:

```
AVOID: GET /authors/3/books/17/reviews/9/comments/4
BETTER: GET /comments/4        (with a `review_id` field in the body)
        or GET /reviews/9/comments?limit=...
```

## Actions that don't map cleanly to CRUD

Sometimes an operation is a genuine action, not a resource — "activate this
account," "reset this password." Two common, both-acceptable patterns:

```
POST /accounts/42/activate
POST /orders/482/cancel
```

Here the trailing segment is a verb, and that's fine — it's a controlled
exception for operations that are actions on a resource rather than a way to
GET/replace it. What you should avoid is using verbs for standard CRUD
(`/createOrder` instead of `POST /orders`).

Alternative pattern some APIs use: model the action as its own sub-resource
that you create.

```
POST /password-reset-requests
Body: { "email": "ada@example.com" }
```

Either style is defensible — pick one convention and apply it consistently
across your API.

## Query parameters vs. path segments

Use a **path segment** when the value identifies which resource you mean;
use a **query parameter** when it filters, sorts, or paginates a collection.

```
GOOD: GET /books/17                     (17 identifies the resource)
GOOD: GET /books?author=asimov&limit=20  (filtering a collection)
BAD:  GET /books?id=17                  (17 is the resource, should be a path segment)
```

We go deeper on this in module 5.

## Casing and formatting conventions

- Use **lowercase, hyphen-separated** path segments: `/order-items`, not
  `/orderItems` or `/Order_Items`. Hyphens are more URL-friendly and the
  overwhelming industry convention.
- Don't put a trailing slash inconsistently — pick `/books` or `/books/` and
  stick to it (most frameworks normalize this automatically, but be aware of
  it if you're writing raw routing rules).
- Avoid file extensions in the URL (`/books.json`) — use the `Accept` header
  instead (covered in module 4) to negotiate the format.
- Keep identifiers opaque where possible. Sequential integer IDs
  (`/orders/482`) are simple and fine for most internal or B2B APIs; UUIDs
  (`/orders/8f14e45f-...`) avoid leaking information like "how many orders
  exist" and prevent trivial enumeration attacks, which matters more for
  public-facing, security-sensitive resources.

## Versioning in the URL (preview)

Many APIs put a version at the start of the path:

```
GET /v1/books/17
```

We cover the tradeoffs of URL vs. header versioning fully in module 9 — for
now, just know `/v1/...` is the most common and easiest-to-explore approach.

## Worked example: designing a "bookshelf" API's URLs

Resources: books, authors, and a user's personal shelves (collections of
books they own).

```
GET    /books                      list all books
POST   /books                      add a new book
GET    /books/{bookId}             get one book
PUT    /books/{bookId}             replace a book
PATCH  /books/{bookId}             partially update a book
DELETE /books/{bookId}             delete a book

GET    /authors                    list authors
GET    /authors/{authorId}         get one author
GET    /authors/{authorId}/books   books by this author

GET    /shelves                    list the current user's shelves
POST   /shelves                    create a new shelf
GET    /shelves/{shelfId}          get one shelf
GET    /shelves/{shelfId}/books    books on this shelf
POST   /shelves/{shelfId}/books    add a book to this shelf (body: { "bookId": 17 })
DELETE /shelves/{shelfId}/books/{bookId}   remove a book from this shelf
```

Notice `/shelves/{shelfId}/books/{bookId}` for removal — it identifies the
exact "book on this shelf" relationship without inventing a separate,
unnecessary resource ID for that link.

## Exercise

Design the URL scheme (methods + paths only, no bodies needed) for a "task
tracker" API with these resources: **projects**, **tasks** (each belongs to
one project), and **comments** (each belongs to one task). Cover:

1. Listing and creating projects.
2. Listing, creating, retrieving, updating, and deleting tasks within a
   specific project.
3. Adding a comment to a task and listing a task's comments.
4. An action to "mark a task complete" that doesn't fit standard CRUD —
   decide which pattern from this module you'd use and justify it in one
   sentence.
