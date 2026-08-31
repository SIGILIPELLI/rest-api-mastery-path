# 04 · OpenAPI/Swagger Spec Basics

OpenAPI (formerly "Swagger") is a machine-readable specification format for
describing a REST API's endpoints, request/response shapes, and auth
requirements in YAML or JSON. Once an API has an OpenAPI document, tools can
auto-generate interactive docs, client SDKs, server stubs, and test suites
from it — the spec becomes a single source of truth instead of hand-written
docs drifting out of sync with the actual code.

## Anatomy of an OpenAPI document

```yaml
openapi: 3.0.3
info:
  title: Bookshelf API
  version: 1.0.0
  description: A simple API for managing a book collection.

servers:
  - url: https://api.example.com/v1

paths:
  /books:
    get:
      summary: List books
      parameters:
        - name: limit
          in: query
          schema: { type: integer, default: 20 }
        - name: genre
          in: query
          schema: { type: string }
      responses:
        '200':
          description: A paginated list of books
          content:
            application/json:
              schema:
                type: object
                properties:
                  data:
                    type: array
                    items: { $ref: '#/components/schemas/Book' }
                  meta:
                    type: object
                    properties:
                      total: { type: integer }
    post:
      summary: Create a book
      requestBody:
        required: true
        content:
          application/json:
            schema: { $ref: '#/components/schemas/NewBook' }
      responses:
        '201':
          description: Book created
          content:
            application/json:
              schema: { $ref: '#/components/schemas/Book' }
        '422':
          description: Validation error

  /books/{id}:
    get:
      summary: Get a single book
      parameters:
        - name: id
          in: path
          required: true
          schema: { type: integer }
      responses:
        '200':
          description: The book
          content:
            application/json:
              schema: { $ref: '#/components/schemas/Book' }
        '404':
          description: Book not found

components:
  schemas:
    Book:
      type: object
      properties:
        id: { type: integer }
        title: { type: string }
        genre: { type: string }
        price: { type: number, format: float }
      required: [id, title]
    NewBook:
      type: object
      properties:
        title: { type: string }
        genre: { type: string }
        price: { type: number }
      required: [title]
  securitySchemes:
    bearerAuth:
      type: http
      scheme: bearer
      bearerFormat: JWT

security:
  - bearerAuth: []
```

Key sections:

- **`info`** — metadata (title, version, description).
- **`servers`** — base URL(s), useful for switching between staging/prod.
- **`paths`** — every endpoint, HTTP method, parameters, request body, and
  possible responses.
- **`components/schemas`** — reusable data shapes, referenced with `$ref`
  to avoid repeating the `Book` shape in every response that includes one.
- **`components/securitySchemes`** + top-level **`security`** — how clients
  authenticate.

## Documenting parameters and validation

```yaml
parameters:
  - name: price_gte
    in: query
    schema: { type: number, minimum: 0 }
    description: Only return books priced at or above this value.
  - name: sort
    in: query
    schema:
      type: string
      enum: [title, -title, price, -price, published_date, -published_date]
```

An `enum` on the `sort` parameter documents exactly which values are valid
— and tools like Swagger UI render it as a dropdown, which doubles as
built-in input validation guidance for anyone testing the API by hand.

## Generating docs and clients from the spec

Once `openapi.yaml` exists, several things become free:

```bash
# Serve interactive docs (Swagger UI) from the spec
npx @redocly/cli preview-docs openapi.yaml

# Validate the spec itself is well-formed
npx @redocly/cli lint openapi.yaml

# Generate a TypeScript client
npx openapi-typescript openapi.yaml -o api-types.ts
```

Swagger UI turns the YAML into a page where every endpoint is documented
*and* directly callable ("Try it out") from the browser — no separate
Postman collection to maintain.

## Spec-first vs code-first

**Code-first**: write the API, then generate the OpenAPI spec from
annotations in the code (e.g. FastAPI does this automatically from Python
type hints; Spring uses `springdoc-openapi`). The spec always matches the
code because it's derived from it.

```python
# FastAPI: spec is generated automatically from this code
@app.get("/books/{id}", response_model=Book)
def get_book(id: int):
    ...
```

**Spec-first**: write `openapi.yaml` first, review/agree on it with
consumers, then generate server stubs and implement against them. This
front-loads API design discussions before any code exists — valuable for
public or cross-team APIs where a breaking rework after the fact is
expensive.

Neither is universally better: code-first is faster for a small team owning
both ends; spec-first is safer when the API has external consumers who need
to build against a stable contract before the implementation is finished.

## Worked example: adding a new endpoint to an existing spec

Adding `DELETE /books/{id}`:

```yaml
  /books/{id}:
    delete:
      summary: Delete a book
      parameters:
        - name: id
          in: path
          required: true
          schema: { type: integer }
      responses:
        '204':
          description: Book deleted, no content returned
        '404':
          description: Book not found
      security:
        - bearerAuth: []
```

Running `redocly lint` after this change catches, e.g., a missing
`description` or an inconsistent response code before it ships — cheaper
than a consumer discovering the mismatch at runtime.

## Exercise

1. Write the OpenAPI `paths` entry for `PATCH /books/{id}` that accepts a
   partial `NewBook`-shaped body and returns the updated `Book`, plus a
   `404` and `422` response.
2. Add a `components/schemas/Error` schema describing the error envelope
   from Level 1's error-handling module, and reference it from every
   non-2xx response in the spec.
3. Explain, with a concrete scenario, one situation where spec-first is
   clearly better than code-first, and one where the reverse is true.
4. Why does putting `sort` behind an `enum` in the spec catch bugs earlier
   than only validating it in server code?
