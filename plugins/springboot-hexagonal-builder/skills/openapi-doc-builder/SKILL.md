---
name: openapi-doc-builder
description: "Generate API documentation using OpenAPI 3.x / Swagger specification. Use this skill whenever the user needs to create API docs, document REST endpoints, generate OpenAPI specs, create Swagger documentation, design API contracts, or produce API reference documentation. Triggers include 'API documentation', 'OpenAPI', 'Swagger', 'API spec', 'API specification', 'REST API docs', 'endpoint documentation', 'API contract', 'API reference', 'document endpoints', 'generate swagger', 'openapi spec', 'openapi yaml', 'openapi json', 'API design', 'documentar API', 'documentación API', 'especificación API', 'contrato API', 'documentar endpoints', 'generar swagger'."
---

# OpenAPI Documentation Builder

Generate comprehensive API documentation using OpenAPI 3.x specification. Produces YAML specs, endpoint references, and interactive-ready documentation.

## Workflow

1. **Gather context** — Understand the API scope, technology stack, and existing endpoints
2. **Select documentation profile** — Choose depth based on project needs
3. **Analyze endpoints** — Identify resources, operations, schemas, and security
4. **Apply best practices** — Validate against OpenAPI standards and API design guidelines
5. **Generate outputs** — Produce OpenAPI spec, endpoint reference, and integration examples

## Step 1: Gather Context

Before documenting, collect essential information. If the user provides a codebase, analyze controllers, routes, DTOs, and existing annotations to pre-fill as much as possible.

### Essential Information

| Information | Why It Matters | How to Get It |
|-------------|---------------|---------------|
| **Project name and version** | Identifies the API in the spec | Ask or infer from pom.xml, package.json, etc. |
| **Base URL / servers** | Defines where the API is hosted | Ask or infer from config files |
| **Authentication method** | Determines security schemes | Ask or analyze security config (JWT, OAuth2, API Key) |
| **Resources / entities** | Core of the API surface | Analyze controllers/routes or ask |
| **Existing annotations** | Pre-populated documentation | Scan for `@Operation`, `@ApiResponse`, `@Schema`, `@RequestMapping`, etc. |
| **Response format** | Standard envelope or direct | Ask or analyze existing responses |
| **Error handling strategy** | Consistent error schemas | Ask or analyze exception handlers |

If the user provides a general description, work with what they give and ask only for critical missing pieces. Default to **OpenAPI 3.1.0** and **YAML** format if not specified.

### Codebase Analysis Strategy

When analyzing an existing codebase, scan in this order:

1. **Controllers / Routes** — Identify all endpoint handlers, HTTP methods, and paths
2. **DTOs / Request-Response models** — Map to OpenAPI schemas
3. **Security configuration** — Identify auth mechanisms
4. **Exception handlers** — Map to error response schemas
5. **Validation annotations** — Map to schema constraints (`@NotNull` → `required`, `@Size` → `minLength`/`maxLength`, `@Pattern` → `pattern`)
6. **Existing OpenAPI annotations** — Preserve any `@Operation`, `@Tag`, `@Schema` metadata already in code

## Step 2: Select Documentation Profile

| Profile | When to Use | Outputs |
|---------|-------------|---------|
| **Quick** | Internal APIs, prototyping | OpenAPI spec (paths + schemas) |
| **Standard** | Most projects | OpenAPI spec + endpoint reference doc + basic examples |
| **Comprehensive** | Public APIs, regulated environments | All Standard outputs + integration guide + SDK examples + changelog template |

Default to **Standard**. Suggest Quick for internal/prototype APIs and Comprehensive for public-facing or partner APIs.

## Step 3: Analyze Endpoints

### Resource Identification

For each resource, document:

| Field | Description |
|-------|-------------|
| **Tag** | Logical grouping (e.g., `Users`, `Orders`, `Authentication`) |
| **Base path** | Resource path prefix (e.g., `/api/v1/users`) |
| **Operations** | HTTP methods supported (GET, POST, PUT, PATCH, DELETE) |
| **Description** | What this resource represents in the domain |

### Endpoint Definition

For each endpoint, capture:

| Field | Description |
|-------|-------------|
| **Operation ID** | Unique identifier, camelCase (e.g., `getUserById`, `createOrder`) |
| **Summary** | Short description (< 80 chars) shown in API explorers |
| **Description** | Detailed explanation including business rules |
| **Parameters** | Path, query, header parameters with types and constraints |
| **Request body** | Schema, content type, required fields, examples |
| **Responses** | Status codes, schemas, descriptions for success and error cases |
| **Security** | Which security scheme(s) apply |
| **Tags** | Resource grouping |

### Schema Definition

For each data model (DTO, entity, request/response body):

| Field | Description |
|-------|-------------|
| **Name** | PascalCase (e.g., `UserResponse`, `CreateOrderRequest`) |
| **Description** | What this schema represents |
| **Properties** | Name, type, format, constraints, description |
| **Required fields** | List of mandatory properties |
| **Examples** | Realistic sample data |

## Step 4: Apply Best Practices

### API Documentation Checklist

Apply these rules to every specification. If a rule is intentionally violated, document why.

- [ ] **Consistent naming** — camelCase for properties, PascalCase for schemas, kebab-case for paths, camelCase for operationId
- [ ] **Versioning** — API version in path (`/api/v1/`) or header, reflected in spec `info.version`
- [ ] **Meaningful HTTP methods** — GET (read), POST (create), PUT (full replace), PATCH (partial update), DELETE (remove)
- [ ] **Proper status codes** — 200 (OK), 201 (Created), 204 (No Content), 400 (Bad Request), 401 (Unauthorized), 403 (Forbidden), 404 (Not Found), 409 (Conflict), 422 (Unprocessable Entity), 500 (Internal Server Error)
- [ ] **Consistent error format** — Standardized error response schema across all endpoints
- [ ] **Pagination** — Document pagination parameters and response metadata for list endpoints
- [ ] **Security schemes** — Every secured endpoint references the appropriate security scheme
- [ ] **Examples** — Every request/response includes at least one realistic example
- [ ] **Descriptions** — Every operation, parameter, and schema property has a description
- [ ] **Content types** — Explicitly declare `application/json` (or other media types) for request/response bodies

### Naming Conventions

| Element | Convention | Example |
|---------|-----------|---------|
| Paths | kebab-case, plural nouns | `/api/v1/order-items` |
| Path parameters | camelCase | `/users/{userId}` |
| Query parameters | camelCase | `?pageSize=20&sortBy=createdAt` |
| Schema names | PascalCase | `CreateUserRequest`, `UserResponse` |
| Properties | camelCase | `firstName`, `createdAt`, `isActive` |
| Operation IDs | camelCase, verb + noun | `listUsers`, `getUserById`, `createOrder` |
| Tags | PascalCase, plural | `Users`, `Orders`, `Authentication` |
| Enum values | UPPER_SNAKE_CASE | `PENDING`, `IN_PROGRESS`, `COMPLETED` |

Load [references/openapi-syntax.md](references/openapi-syntax.md) for complete OpenAPI 3.x syntax reference.
Load [references/common-patterns.md](references/common-patterns.md) for common API documentation patterns (pagination, filtering, error handling, auth).
Load [references/rest-principles.md](references/rest-principles.md) for REST architectural constraints, resource-oriented design, HTTP semantics, idempotency, and API maturity model.

## Step 5: Generate Outputs

### 5.1 OpenAPI Specification (YAML)

Generate a complete, valid OpenAPI 3.x spec. Structure:

```yaml
openapi: 3.1.0
info:
  title: <Project Name> API
  description: |
    <Brief API description>
  version: 1.0.0
  contact:
    name: <Team Name>
    email: <contact email>

servers:
  - url: http://localhost:8080/api/v1
    description: Local development
  - url: https://api.example.com/v1
    description: Production

tags:
  - name: Users
    description: User management operations
  - name: Orders
    description: Order processing operations

security:
  - bearerAuth: []

paths:
  /users:
    get:
      tags: [Users]
      operationId: listUsers
      summary: List all users
      description: Returns a paginated list of users. Supports filtering by status and searching by name or email.
      parameters:
        - $ref: '#/components/parameters/PageParam'
        - $ref: '#/components/parameters/PageSizeParam'
        - name: status
          in: query
          description: Filter by user status
          schema:
            $ref: '#/components/schemas/UserStatus'
        - name: search
          in: query
          description: Search by name or email (case-insensitive, partial match)
          schema:
            type: string
            minLength: 2
            example: "john"
      responses:
        '200':
          description: Paginated list of users
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/PagedUserResponse'
              example:
                content:
                  - id: "550e8400-e29b-41d4-a716-446655440000"
                    email: "john.doe@example.com"
                    fullName: "John Doe"
                    status: ACTIVE
                    createdAt: "2026-01-15T10:30:00Z"
                page:
                  number: 0
                  size: 20
                  totalElements: 150
                  totalPages: 8
        '401':
          $ref: '#/components/responses/Unauthorized'

    post:
      tags: [Users]
      operationId: createUser
      summary: Create a new user
      description: Creates a new user account. Email must be unique across the system.
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/CreateUserRequest'
            example:
              email: "jane.doe@example.com"
              fullName: "Jane Doe"
              password: "SecureP@ss123"
      responses:
        '201':
          description: User created successfully
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/UserResponse'
          headers:
            Location:
              description: URL of the newly created user
              schema:
                type: string
                example: /api/v1/users/550e8400-e29b-41d4-a716-446655440001
        '400':
          $ref: '#/components/responses/BadRequest'
        '409':
          $ref: '#/components/responses/Conflict'

  /users/{userId}:
    get:
      tags: [Users]
      operationId: getUserById
      summary: Get user by ID
      parameters:
        - $ref: '#/components/parameters/UserIdParam'
      responses:
        '200':
          description: User details
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/UserResponse'
        '404':
          $ref: '#/components/responses/NotFound'

components:
  securitySchemes:
    bearerAuth:
      type: http
      scheme: bearer
      bearerFormat: JWT
      description: JWT token obtained from the /auth/login endpoint

  parameters:
    PageParam:
      name: page
      in: query
      description: Page number (zero-based)
      schema:
        type: integer
        minimum: 0
        default: 0
    PageSizeParam:
      name: size
      in: query
      description: Number of items per page
      schema:
        type: integer
        minimum: 1
        maximum: 100
        default: 20
    UserIdParam:
      name: userId
      in: path
      required: true
      description: Unique user identifier (UUID)
      schema:
        type: string
        format: uuid

  schemas:
    UserStatus:
      type: string
      enum: [ACTIVE, INACTIVE, SUSPENDED]
      description: User account status

    CreateUserRequest:
      type: object
      required: [email, fullName, password]
      properties:
        email:
          type: string
          format: email
          maxLength: 255
          description: User email address (must be unique)
        fullName:
          type: string
          minLength: 2
          maxLength: 100
          description: User full name
        password:
          type: string
          format: password
          minLength: 8
          maxLength: 64
          description: Account password (min 8 chars, must include uppercase, lowercase, number, and special char)

    UserResponse:
      type: object
      properties:
        id:
          type: string
          format: uuid
          description: Unique user identifier
        email:
          type: string
          format: email
          description: User email address
        fullName:
          type: string
          description: User full name
        status:
          $ref: '#/components/schemas/UserStatus'
        createdAt:
          type: string
          format: date-time
          description: Account creation timestamp
        updatedAt:
          type: string
          format: date-time
          description: Last modification timestamp

    PagedUserResponse:
      type: object
      properties:
        content:
          type: array
          items:
            $ref: '#/components/schemas/UserResponse'
        page:
          $ref: '#/components/schemas/PageMetadata'

    PageMetadata:
      type: object
      properties:
        number:
          type: integer
          description: Current page number (zero-based)
        size:
          type: integer
          description: Number of items per page
        totalElements:
          type: integer
          format: int64
          description: Total number of items across all pages
        totalPages:
          type: integer
          description: Total number of pages

    ErrorResponse:
      type: object
      required: [status, error, message, timestamp]
      properties:
        status:
          type: integer
          description: HTTP status code
          example: 400
        error:
          type: string
          description: Error category
          example: "Bad Request"
        message:
          type: string
          description: Human-readable error description
          example: "Validation failed"
        timestamp:
          type: string
          format: date-time
          description: When the error occurred
        path:
          type: string
          description: Request path that caused the error
          example: "/api/v1/users"
        details:
          type: array
          items:
            $ref: '#/components/schemas/FieldError'
          description: Detailed field-level errors (for validation failures)

    FieldError:
      type: object
      properties:
        field:
          type: string
          description: Field that failed validation
          example: "email"
        message:
          type: string
          description: Validation error message
          example: "must be a valid email address"
        rejectedValue:
          description: The value that was rejected
          example: "not-an-email"

  responses:
    BadRequest:
      description: Invalid request parameters or body
      content:
        application/json:
          schema:
            $ref: '#/components/schemas/ErrorResponse'
          example:
            status: 400
            error: "Bad Request"
            message: "Validation failed"
            timestamp: "2026-01-15T10:30:00Z"
            path: "/api/v1/users"
            details:
              - field: "email"
                message: "must be a valid email address"
                rejectedValue: "not-an-email"
    Unauthorized:
      description: Missing or invalid authentication token
      content:
        application/json:
          schema:
            $ref: '#/components/schemas/ErrorResponse'
          example:
            status: 401
            error: "Unauthorized"
            message: "Invalid or expired JWT token"
            timestamp: "2026-01-15T10:30:00Z"
            path: "/api/v1/users"
    Forbidden:
      description: Insufficient permissions
      content:
        application/json:
          schema:
            $ref: '#/components/schemas/ErrorResponse'
          example:
            status: 403
            error: "Forbidden"
            message: "You do not have permission to access this resource"
            timestamp: "2026-01-15T10:30:00Z"
            path: "/api/v1/users/admin-panel"
    NotFound:
      description: Resource not found
      content:
        application/json:
          schema:
            $ref: '#/components/schemas/ErrorResponse'
          example:
            status: 404
            error: "Not Found"
            message: "User not found with id: 550e8400-e29b-41d4-a716-446655440099"
            timestamp: "2026-01-15T10:30:00Z"
            path: "/api/v1/users/550e8400-e29b-41d4-a716-446655440099"
    Conflict:
      description: Resource conflict (e.g., duplicate entry)
      content:
        application/json:
          schema:
            $ref: '#/components/schemas/ErrorResponse'
          example:
            status: 409
            error: "Conflict"
            message: "User with email john.doe@example.com already exists"
            timestamp: "2026-01-15T10:30:00Z"
            path: "/api/v1/users"
```

**Spec guidelines:**
- Declare paths in logical resource order
- Use `$ref` extensively to avoid duplication — reusable parameters, schemas, and responses go in `components`
- Every operation must have `operationId`, `summary`, `tags`, and at least one response
- Include `example` or `examples` for all schemas and responses
- Use `format` hints: `email`, `uuid`, `date-time`, `uri`, `password`, `int32`, `int64`
- Document constraints: `minimum`, `maximum`, `minLength`, `maxLength`, `pattern`, `minItems`, `maxItems`

### 5.2 Endpoint Reference Document (Standard and Comprehensive profiles)

Generate a human-readable endpoint reference in Markdown:

```markdown
## Users

### List Users

`GET /api/v1/users`

Returns a paginated list of users.

**Query Parameters:**

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| `page` | integer | No | 0 | Page number (zero-based) |
| `size` | integer | No | 20 | Items per page (max 100) |
| `status` | string | No | — | Filter by status: `ACTIVE`, `INACTIVE`, `SUSPENDED` |
| `search` | string | No | — | Search by name or email (min 2 chars) |

**Response:** `200 OK`

| Field | Type | Description |
|-------|------|-------------|
| `content` | UserResponse[] | Array of user objects |
| `page.number` | integer | Current page number |
| `page.size` | integer | Items per page |
| `page.totalElements` | integer | Total items |
| `page.totalPages` | integer | Total pages |

**Example:**

\```bash
curl -X GET "https://api.example.com/v1/users?page=0&size=20&status=ACTIVE" \
  -H "Authorization: Bearer <token>"
\```

**Errors:** `401` Unauthorized | `403` Forbidden
```

### 5.3 Integration Guide (Comprehensive profile)

For public-facing APIs, generate an integration guide covering:

1. **Authentication** — How to obtain and use tokens, token refresh flow
2. **Rate limiting** — Limits, headers (`X-RateLimit-Limit`, `X-RateLimit-Remaining`, `X-RateLimit-Reset`), retry strategy
3. **Pagination** — How to iterate through paginated results
4. **Error handling** — How to parse error responses, common error codes and remediation
5. **Webhooks** — Event types, payload format, signature verification (if applicable)
6. **SDK examples** — Code snippets in relevant languages (Java, Python, JavaScript/TypeScript, cURL)

### 5.4 Changelog Template (Comprehensive profile)

Provide a versioning and changelog structure:

```markdown
# API Changelog

## [1.1.0] - 2026-04-01

### Added
- `GET /api/v1/users/export` — Export users to CSV

### Changed
- `GET /api/v1/users` — Added `search` query parameter for name/email search

### Deprecated
- `GET /api/v1/users?name=` — Use `search` parameter instead (removal in v2.0)

### Removed
- None

### Fixed
- `POST /api/v1/orders` — Fixed 500 error when `items` array was empty
```

## Output Location

Write outputs to `docs/api/` by default (or the location the user specifies):

| File | Profile | Content |
|------|---------|---------|
| `docs/api/openapi-<project>.yaml` | All | Complete OpenAPI 3.x specification |
| `docs/api/endpoint-reference-<project>.md` | Standard+ | Human-readable endpoint reference |
| `docs/api/integration-guide-<project>.md` | Comprehensive | Authentication, rate limiting, SDK examples |
| `docs/api/changelog-<project>.md` | Comprehensive | API versioning and changelog template |

## Adapting to Technology Stack

### Spring Boot (Java/Kotlin)

Scan for and leverage:
- `@RestController`, `@RequestMapping`, `@GetMapping`, `@PostMapping`, etc.
- `@Operation`, `@ApiResponse`, `@Parameter`, `@Schema` (springdoc-openapi)
- `@Valid`, `@NotNull`, `@Size`, `@Pattern`, `@Email` (Bean Validation)
- `ResponseEntity<>` return types for status code inference
- `@ControllerAdvice` / `@ExceptionHandler` for error response schemas
- `Pageable` parameters for pagination pattern detection
- `@PreAuthorize`, `@Secured` for security requirements

### Node.js / Express

Scan for:
- Route definitions (`app.get()`, `router.post()`, etc.)
- Middleware chains for auth and validation
- Joi/Zod/Yup schemas for request validation → map to OpenAPI schemas
- Express error middleware for error response format
- Swagger JSDoc comments (`@swagger` / `@openapi`)

### Python / FastAPI / Django

Scan for:
- FastAPI: `@app.get()` decorators, Pydantic models (auto-mapped to schemas), `Depends()` for auth
- Django REST: `ViewSet`, `Serializer`, `permission_classes`
- Type annotations for automatic schema inference

### .NET / ASP.NET Core

Scan for:
- `[ApiController]`, `[HttpGet]`, `[HttpPost]`, route attributes
- `[ProducesResponseType]` for response documentation
- `[Authorize]` for security requirements
- `FluentValidation` rules for schema constraints
- Swashbuckle/NSwag annotations

## Adapting to API Style

| API Style | Considerations |
|-----------|---------------|
| **REST** | Standard path-based resources, CRUD operations, HATEOAS links (if used) |
| **REST + WebFlux** | Same structure, note reactive types (`Mono<>`, `Flux<>`) for schema extraction |
| **GraphQL** | Document the REST wrapper or gateway; for pure GraphQL, suggest schema documentation tools instead |
| **gRPC Gateway** | Document the HTTP/JSON transcoding layer using OpenAPI |
| **Event-driven** | Document webhook endpoints and async callback patterns using `callbacks` in OpenAPI |

## References

- [references/openapi-syntax.md](references/openapi-syntax.md) — Complete OpenAPI 3.x structure, data types, and advanced features
- [references/common-patterns.md](references/common-patterns.md) — Pagination, filtering, sorting, error handling, authentication, file upload, and versioning patterns
- [references/rest-principles.md](references/rest-principles.md) — REST architectural constraints, resource-oriented design, HTTP methods and semantics, idempotency, HATEOAS, Richardson maturity model, and common anti-patterns
