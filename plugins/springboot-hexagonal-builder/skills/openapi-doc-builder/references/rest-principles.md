# REST API Design Principles

Foundational principles and constraints for designing RESTful APIs. Use this reference to validate API designs and guide documentation decisions.

## Table of Contents

1. [REST Architectural Constraints](#rest-architectural-constraints)
2. [Resource-Oriented Design](#resource-oriented-design)
3. [HTTP Methods and Semantics](#http-methods-and-semantics)
4. [HTTP Status Codes](#http-status-codes)
5. [URI Design](#uri-design)
6. [Content Negotiation](#content-negotiation)
7. [Statelessness](#statelessness)
8. [Idempotency and Safety](#idempotency-and-safety)
9. [HATEOAS and Resource Linking](#hateoas-and-resource-linking)
10. [API Maturity Model (Richardson)](#api-maturity-model-richardson)
11. [Common Anti-Patterns](#common-anti-patterns)

---

## REST Architectural Constraints

REST (Representational State Transfer) defines six architectural constraints. A truly RESTful API should adhere to all of them.

| Constraint | Description | Impact on API Design |
|------------|-------------|---------------------|
| **Client-Server** | Separation of concerns between UI and data storage | API is independent of any specific client; multiple clients can consume the same API |
| **Stateless** | Each request contains all information needed to process it | No server-side sessions; authentication token sent with every request |
| **Cacheable** | Responses must declare themselves cacheable or non-cacheable | Use `Cache-Control`, `ETag`, `Last-Modified` headers appropriately |
| **Uniform Interface** | Standardized way to interact with resources | Consistent URI structure, HTTP methods, media types, and response formats |
| **Layered System** | Client cannot tell if connected directly to the server | Enables load balancers, gateways, CDNs, and proxies transparently |
| **Code on Demand** *(optional)* | Server can extend client functionality by transferring executable code | Rarely used in REST APIs; more common in web applications |

### Uniform Interface Sub-Constraints

The uniform interface is the most distinctive constraint. It comprises four principles:

1. **Resource identification** — Each resource is identified by a URI (`/api/v1/users/123`)
2. **Resource manipulation through representations** — Clients manipulate resources by sending representations (JSON, XML) along with metadata
3. **Self-descriptive messages** — Each message includes enough information to describe how to process it (Content-Type, methods, status codes)
4. **Hypermedia as the engine of application state (HATEOAS)** — Responses include links to related actions and resources

## Resource-Oriented Design

### What Is a Resource?

A resource is any concept that can be addressed — a domain entity, a collection, a process, or a computed value. Resources are **nouns**, not verbs.

### Resource Types

| Type | Description | URI Pattern | Example |
|------|-------------|-------------|---------|
| **Document** | Single instance of a resource | `/{collection}/{id}` | `/users/42` |
| **Collection** | Server-managed set of resources | `/{collection}` | `/users` |
| **Store** | Client-managed set of resources | `/{store}` | `/favorites` |
| **Controller** | Procedural action (use sparingly) | `/{resource}/{action}` | `/users/42/activate` |

### Resource Modeling Guidelines

- **Identify domain entities** — Users, Orders, Products, Payments, etc.
- **Model relationships** — Nested resources for strong ownership (`/users/42/orders`), top-level resources for independent entities (`/orders`)
- **Avoid deep nesting** — Maximum 2 levels deep (`/users/42/orders/7` is OK; `/users/42/orders/7/items/3/reviews` is too deep — flatten to `/order-items/3/reviews`)
- **Use sub-resources for actions on resources** — `POST /orders/42/cancel` instead of `POST /cancel-order`
- **Differentiate between resources and representations** — A `User` resource may have multiple representations: `UserSummary`, `UserDetail`, `UserAdmin`

## HTTP Methods and Semantics

### Method Definitions

| Method | Purpose | Request Body | Response Body | Idempotent | Safe |
|--------|---------|-------------|---------------|------------|------|
| **GET** | Retrieve resource(s) | No | Yes | Yes | Yes |
| **POST** | Create a resource or trigger an action | Yes | Yes | No | No |
| **PUT** | Full replacement of a resource | Yes | Optional | Yes | No |
| **PATCH** | Partial update of a resource | Yes | Yes | No* | No |
| **DELETE** | Remove a resource | No | Optional | Yes | No |
| **HEAD** | Same as GET but without response body | No | No | Yes | Yes |
| **OPTIONS** | Describe communication options | No | Yes | Yes | Yes |

*PATCH can be idempotent depending on the implementation (e.g., JSON Merge Patch is idempotent, JSON Patch may not be).

### Method Selection Guide

| Action | Correct Method | Wrong Approach |
|--------|---------------|----------------|
| Get a list of users | `GET /users` | `POST /getUsers` |
| Get a single user | `GET /users/42` | `GET /user?id=42` |
| Create a user | `POST /users` | `PUT /users/42` (unless client controls ID) |
| Full update of a user | `PUT /users/42` | `POST /users/42/update` |
| Update user's email only | `PATCH /users/42` | `PUT /users/42` (with all fields) |
| Delete a user | `DELETE /users/42` | `POST /users/42/delete` |
| Search users | `GET /users?q=john` | `POST /users/search` (unless query is complex) |
| Activate a user | `POST /users/42/activate` | `PATCH /users/42` with `{status: "ACTIVE"}` (both valid, controller pattern is clearer for business actions) |

### PUT vs PATCH

| Aspect | PUT | PATCH |
|--------|-----|-------|
| Semantics | Full replacement | Partial update |
| Missing fields | Set to null/default | Left unchanged |
| Request body | Complete resource representation | Only changed fields |
| Idempotent | Always | Depends on patch format |
| When to use | Client has full resource state | Client updates specific fields |

## HTTP Status Codes

### Success Codes

| Code | Name | When to Use |
|------|------|-------------|
| **200** | OK | Successful GET, PUT, PATCH, or DELETE that returns data |
| **201** | Created | Successful POST that creates a resource. Include `Location` header with URL of new resource |
| **202** | Accepted | Request accepted for async processing. Return operation status URL |
| **204** | No Content | Successful DELETE or PUT/PATCH with no response body |

### Client Error Codes

| Code | Name | When to Use |
|------|------|-------------|
| **400** | Bad Request | Malformed syntax, invalid JSON, type mismatches |
| **401** | Unauthorized | Missing or invalid authentication credentials |
| **403** | Forbidden | Valid credentials but insufficient permissions |
| **404** | Not Found | Resource does not exist |
| **405** | Method Not Allowed | HTTP method not supported on this endpoint |
| **406** | Not Acceptable | Server cannot produce response matching `Accept` header |
| **409** | Conflict | State conflict (duplicate resource, version mismatch, business rule violation) |
| **410** | Gone | Resource permanently deleted (use instead of 404 when deletion is known) |
| **415** | Unsupported Media Type | Request `Content-Type` not supported |
| **422** | Unprocessable Entity | Syntactically valid but semantically invalid (business validation failures) |
| **429** | Too Many Requests | Rate limit exceeded. Include `Retry-After` header |

### Server Error Codes

| Code | Name | When to Use |
|------|------|-------------|
| **500** | Internal Server Error | Unexpected server failure. Never expose stack traces |
| **502** | Bad Gateway | Upstream service returned an invalid response |
| **503** | Service Unavailable | Server temporarily overloaded or in maintenance. Include `Retry-After` header |
| **504** | Gateway Timeout | Upstream service did not respond in time |

### Status Code Decision Guide

```
Request received
├── Is the request well-formed (valid JSON, correct types)?
│   ├── No → 400 Bad Request
│   └── Yes → Is the client authenticated?
│       ├── No → 401 Unauthorized
│       └── Yes → Is the client authorized?
│           ├── No → 403 Forbidden
│           └── Yes → Does the resource exist?
│               ├── No → 404 Not Found
│               └── Yes → Is the operation valid?
│                   ├── No (business rule violation) → 409 Conflict or 422 Unprocessable
│                   └── Yes → Process request
│                       ├── Created → 201 Created
│                       ├── No content to return → 204 No Content
│                       ├── Async processing → 202 Accepted
│                       └── Success with data → 200 OK
```

## URI Design

### Core Rules

| Rule | Good | Bad |
|------|------|-----|
| Use nouns, not verbs | `/users` | `/getUsers`, `/createUser` |
| Use plural nouns for collections | `/users`, `/orders` | `/user`, `/order` |
| Use kebab-case for multi-word paths | `/order-items` | `/orderItems`, `/order_items` |
| Use camelCase for query parameters | `?sortBy=createdAt` | `?sort_by=created_at` |
| No trailing slashes | `/users` | `/users/` |
| No file extensions | `/users/42` | `/users/42.json` |
| No CRUD in URIs | `DELETE /users/42` | `/users/42/delete` |
| Hierarchical relationships via nesting | `/users/42/orders` | `/orders?userId=42` (both valid, nesting implies ownership) |

### Versioning in URIs

```
# Recommended: Path versioning
/api/v1/users
/api/v2/users

# Alternative: Header versioning
GET /api/users
Accept: application/vnd.myapi.v2+json

# Alternative: Query parameter
/api/users?version=2
```

Path versioning is the most explicit and cache-friendly. Use it as default unless the project has specific reasons for another strategy.

### Collection Operations

| Operation | URI | Method | Description |
|-----------|-----|--------|-------------|
| List | `/users` | GET | Retrieve collection (with pagination, filtering, sorting) |
| Create | `/users` | POST | Create new resource in collection |
| Read | `/users/{id}` | GET | Retrieve single resource |
| Replace | `/users/{id}` | PUT | Full update of resource |
| Update | `/users/{id}` | PATCH | Partial update of resource |
| Delete | `/users/{id}` | DELETE | Remove resource |

### Non-CRUD Operations

Not all operations map cleanly to CRUD. Use these patterns for actions:

| Scenario | Pattern | Example |
|----------|---------|---------|
| State transition | `POST /{resource}/{id}/{action}` | `POST /orders/42/cancel` |
| Batch operation | `POST /{resource}/batch` | `POST /users/batch` |
| Complex search | `POST /{resource}/search` | `POST /products/search` (when query is too complex for GET query params) |
| Aggregate/computed | `GET /{resource}/stats` | `GET /orders/stats` |
| Current user | `GET /me` or `GET /users/me` | Alias for the authenticated user's resource |

## Content Negotiation

### Request Headers

| Header | Purpose | Example |
|--------|---------|---------|
| `Content-Type` | Format of request body | `application/json` |
| `Accept` | Desired response format | `application/json` |
| `Accept-Language` | Preferred language for messages | `es, en;q=0.9` |
| `Accept-Encoding` | Supported compression | `gzip, deflate` |

### Best Practices

- **Default to `application/json`** for both request and response
- **Always declare `Content-Type`** in responses
- **Return 406 Not Acceptable** if the server cannot produce the requested format
- **Return 415 Unsupported Media Type** if the request body format is not supported
- **Support `application/json` at minimum**; add `application/xml`, `text/csv`, etc. only when needed

## Statelessness

### Principles

- **Every request is self-contained** — Server does not store client context between requests
- **Authentication via tokens** — JWT, API keys, or OAuth tokens sent with every request
- **No server-side sessions** — Do not use session IDs or cookies for API state management
- **Client manages state** — Client tracks current page, selected filters, navigation context

### Benefits for API Design

| Benefit | Impact |
|---------|--------|
| Scalability | Any server instance can handle any request — enables horizontal scaling |
| Reliability | Server crashes don't lose client state; client can retry on any instance |
| Cacheability | Stateless responses are easier to cache at CDN/proxy level |
| Simplicity | No session synchronization between server nodes |

### Exceptions

Some scenarios require server-side state. Document these clearly:

- **Long-running operations** — Use async patterns with operation status resources
- **Multi-step workflows** — Model as resources with state (`/orders` with status field) rather than sessions
- **Real-time connections** — WebSocket connections maintain state by nature; document the connection lifecycle

## Idempotency and Safety

### Definitions

- **Safe** — The method does not modify server state (GET, HEAD, OPTIONS)
- **Idempotent** — Calling the method multiple times produces the same result as calling it once (GET, PUT, DELETE, HEAD, OPTIONS)

### Idempotency Matrix

| Method | Safe | Idempotent | Explanation |
|--------|------|------------|-------------|
| GET | Yes | Yes | Reading data never changes it |
| HEAD | Yes | Yes | Same as GET without body |
| OPTIONS | Yes | Yes | Describes allowed methods |
| PUT | No | Yes | Replacing with the same data yields the same result |
| DELETE | No | Yes | Deleting an already-deleted resource yields the same state |
| POST | No | No | Each call may create a new resource |
| PATCH | No | Depends | JSON Merge Patch is idempotent; JSON Patch operations may not be |

### Idempotency Keys

For non-idempotent operations (POST), support idempotency keys to prevent duplicate processing:

```
POST /api/v1/payments
Idempotency-Key: 550e8400-e29b-41d4-a716-446655440000
Content-Type: application/json

{
  "amount": 100.00,
  "currency": "USD"
}
```

**Document idempotency keys when:**
- The API processes payments or financial transactions
- Duplicate operations could cause data corruption
- Network retries are expected (mobile clients, unreliable networks)

## HATEOAS and Resource Linking

### Principle

Responses should include links to related actions and resources, allowing clients to navigate the API dynamically rather than hardcoding URLs.

### Link Format (HAL style)

```json
{
  "id": "42",
  "email": "john@example.com",
  "status": "ACTIVE",
  "_links": {
    "self": { "href": "/api/v1/users/42" },
    "orders": { "href": "/api/v1/users/42/orders" },
    "deactivate": { "href": "/api/v1/users/42/deactivate", "method": "POST" }
  }
}
```

### When to Use HATEOAS

| Scenario | Recommendation |
|----------|---------------|
| Public API with many consumers | Recommended — reduces client coupling to URL structure |
| Internal microservice API | Optional — services typically share a contract |
| API with complex state transitions | Recommended — links communicate available actions per state |
| Simple CRUD API | Optional — adds complexity with limited benefit |

### Documenting HATEOAS

When the API uses HATEOAS, document:
- **Link relations** — What each link name means (e.g., `self`, `next`, `orders`, `cancel`)
- **Available links per state** — Which links appear based on resource state (e.g., `cancel` only appears when order status is `PENDING`)
- **Link format** — HAL, JSON:API, or custom format
- **Discoverability** — Root endpoint (`GET /api`) that lists all available resources

## API Maturity Model (Richardson)

The Richardson Maturity Model classifies REST APIs by how well they leverage HTTP. Use it to assess and guide API design.

| Level | Name | Description | Example |
|-------|------|-------------|---------|
| **Level 0** | The Swamp of POX | Single URI, single method (POST), RPC-style | `POST /api` with action in body |
| **Level 1** | Resources | Multiple URIs (resources), but single method | `POST /users`, `POST /orders` |
| **Level 2** | HTTP Verbs | Multiple URIs + proper HTTP methods and status codes | `GET /users`, `POST /users`, `DELETE /users/42` |
| **Level 3** | Hypermedia Controls | Level 2 + HATEOAS — responses include navigational links | Responses contain `_links` |

**Target Level 2 as minimum** for any REST API. Level 3 is ideal for public APIs but adds complexity.

### Level 2 Checklist (Minimum Standard)

- [ ] Resources identified by URIs (nouns, not verbs)
- [ ] Correct HTTP methods for each operation
- [ ] Appropriate status codes (not just 200 for everything)
- [ ] Content negotiation via `Accept` and `Content-Type`
- [ ] Stateless communication
- [ ] Consistent error response format

## Common Anti-Patterns

Avoid these mistakes when designing and documenting APIs:

| Anti-Pattern | Problem | Correct Approach |
|-------------|---------|-----------------|
| **Verbs in URIs** | `POST /createUser`, `GET /getUsers` | Use HTTP methods: `POST /users`, `GET /users` |
| **Singular collection names** | `/user`, `/order` | Use plural: `/users`, `/orders` |
| **Ignoring HTTP methods** | Everything is POST | Map operations to correct methods |
| **200 for everything** | `200 { "error": "Not Found" }` | Use proper status codes: `404 Not Found` |
| **Nested too deep** | `/a/1/b/2/c/3/d/4` | Flatten: max 2 levels of nesting |
| **Exposing internal models** | DB columns as API fields | Design API schemas independently from DB |
| **Missing pagination** | `GET /users` returns 100K records | Always paginate collections |
| **No versioning** | Breaking changes break all clients | Version from day one |
| **Chatty APIs** | 10 calls to render one page | Provide composed endpoints or allow field expansion (`?expand=orders`) |
| **Ignoring caching** | No cache headers on read endpoints | Use `Cache-Control`, `ETag` for GET responses |
| **RPC tunneling** | `POST /api { "action": "getUser", "id": 42 }` | Resource-oriented design with proper methods |
| **Missing `Location` header** | POST returns 201 without resource URL | Always return `Location` on 201 Created |
| **Inconsistent naming** | Mix of camelCase, snake_case, kebab-case | Pick one convention per context and stick to it |
