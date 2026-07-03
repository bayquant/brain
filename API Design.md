---
tags: [api, rest, architecture, design]
---
# API Design

API design is about defining a **contract** between a service and its consumers — resources, operations, data shapes, and error semantics — that stays stable and predictable as the system evolves.

---

## CORE CONCEPTS

```
Resource — a noun the API exposes (user, order, invoice)
└── Representation — the JSON/XML shape returned for that resource

Endpoint — URI + HTTP method combination
└── Contract — request shape, response shape, status codes, error format

Versioning — how breaking changes are introduced without breaking existing clients
Idempotency — whether repeating a request has the same effect as doing it once
```

- **Resource** — the thing being manipulated, named with a noun (`/orders`), not a verb (`/getOrders`)
- **Collection vs. item** — `/orders` (collection) vs `/orders/{id}` (single item)
- **Contract-first** — design the schema (OpenAPI, etc.) before writing implementation code, so client and server teams can work in parallel

---

## REST BASICS

### RESOURCE NAMING

```
GET    /orders           # list orders
POST   /orders           # create an order
GET    /orders/{id}      # fetch one order
PUT    /orders/{id}      # replace an order
PATCH  /orders/{id}      # partially update an order
DELETE /orders/{id}      # delete an order

GET    /orders/{id}/items       # nested collection
POST   /orders/{id}/items       # add an item to an order
```

- Use plural nouns for collections: `/orders`, not `/order`
- Nest resources only one level deep where possible — deep nesting (`/orders/{id}/items/{id}/discounts/{id}`) becomes hard to route and reason about
- Avoid verbs in URIs; the HTTP method already carries the verb

### HTTP METHODS AND IDEMPOTENCY

| Method | Purpose | Idempotent | Safe (no side effects) |
|---|---|---|---|
| GET    | read           | yes | yes |
| POST   | create / action | no  | no |
| PUT    | replace        | yes | no |
| PATCH  | partial update | no* | no |
| DELETE | remove         | yes | no |

*PATCH can be made idempotent depending on the patch format (e.g. JSON Merge Patch is idempotent; a "increment counter" patch is not).

### STATUS CODES

```
200 OK                  — success, response body included
201 Created              — resource created, Location header points to it
204 No Content           — success, no response body (common for DELETE)

400 Bad Request          — malformed request, client error
401 Unauthorized         — missing/invalid authentication
403 Forbidden            — authenticated, but not allowed
404 Not Found            — resource does not exist
409 Conflict             — request conflicts with current state (e.g. duplicate)
422 Unprocessable Entity — well-formed but semantically invalid (validation errors)
429 Too Many Requests    — rate limited

500 Internal Server Error — unhandled server-side failure
503 Service Unavailable   — server temporarily can't handle the request
```

---

## REQUEST AND RESPONSE SHAPE

### CONSISTENT ENVELOPE

```json
{
  "data": { "id": "123", "status": "shipped" },
  "meta": { "requestId": "abc-123" }
}
```

For collections, include pagination metadata alongside the array rather than returning a bare array — a bare array can't be extended later without a breaking change.

```json
{
  "data": [ { "id": "1" }, { "id": "2" } ],
  "meta": { "page": 1, "perPage": 20, "total": 134 }
}
```

### ERROR FORMAT

Keep error shape consistent across every endpoint so clients can handle errors generically.

```json
{
  "error": {
    "code": "validation_failed",
    "message": "email is not a valid address",
    "field": "email"
  }
}
```

---

## PAGINATION

```
# Offset-based — simple, but can skip/duplicate items if data changes mid-page
GET /orders?page=2&perPage=20

# Cursor-based — stable under concurrent writes, preferred for large/live datasets
GET /orders?cursor=eyJpZCI6MTIzfQ&limit=20
```

---

## VERSIONING

```
# URI versioning — most visible, easiest for clients to reason about
GET /v1/orders
GET /v2/orders

# Header versioning — keeps URIs stable, less discoverable
GET /orders
Accept: application/vnd.myapi.v2+json
```

- Prefer **additive, backward-compatible changes** (new optional fields, new endpoints) over bumping a version
- Bump the version only for breaking changes: removing/renaming a field, changing a field's type, changing status code semantics
- Deprecate old versions with a `Sunset` header and a published timeline before removal

---

## AUTHENTICATION AND AUTHORIZATION

```
# Bearer token (OAuth2 / JWT) — most common for APIs
Authorization: Bearer <token>

# API key — simpler, common for server-to-server or low-stakes public APIs
X-API-Key: <key>
```

- Authentication (who you are) is separate from authorization (what you can do) — a valid token can still yield `403 Forbidden` on a specific resource
- Never put secrets or tokens in the URL — they leak into logs, browser history, and referrer headers

---

## IDEMPOTENCY FOR NON-IDEMPOTENT OPERATIONS

`POST` isn't idempotent by default, which is a problem for retries (network timeout, client retry logic). Fix with a client-supplied idempotency key.

```
POST /payments
Idempotency-Key: 7c9e6679-7425-40de-944b-e07fc1f90ae7
```

The server stores the key with the result of the first request; a retry with the same key returns the original response instead of creating a duplicate.

---

## RATE LIMITING

```
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 42
X-RateLimit-Reset: 1719792000

429 Too Many Requests
Retry-After: 30
```

---

## PATTERNS

### FILTERING, SORTING, FIELD SELECTION

```
GET /orders?status=shipped&sort=-createdAt&fields=id,status,total
```

- `-` prefix on `sort` indicates descending order
- `fields` lets clients request a subset of a representation, reducing payload size

### PARTIAL RESPONSE FOR EXPENSIVE FIELDS

Keep expensive-to-compute fields opt-in rather than always included.

```
GET /orders/{id}?expand=items,customer
```

### BULK OPERATIONS

Avoid N sequential requests for bulk changes — provide a batch endpoint instead.

```
POST /orders/bulk
{ "operations": [ { "op": "update", "id": "1", "status": "shipped" }, ... ] }
```

### HATEOAS (OPTIONAL, OFTEN SKIPPED IN PRACTICE)

Embedding links to related actions/resources in the response so clients discover the API at runtime instead of hardcoding URIs.

```json
{
  "id": "123",
  "status": "pending",
  "links": {
    "self": "/orders/123",
    "cancel": "/orders/123/cancel"
  }
}
```

---

## COMMON PITFALLS

- Returning `200 OK` for errors with an `"error": true` field buried in the body — breaks HTTP semantics and client error handling
- Leaking internal database IDs, stack traces, or schema details in error messages
- Making breaking changes without a version bump (renaming/removing fields, changing types)
- Deep resource nesting that forces clients to know the full parent chain just to reach a child resource
- Inconsistent casing (`snake_case` vs `camelCase`) across endpoints in the same API

See also [[Software Architecture Patterns]] and [[CQRS]].
