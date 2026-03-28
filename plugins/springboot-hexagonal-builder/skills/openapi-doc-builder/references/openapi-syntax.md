# OpenAPI 3.x Syntax Reference

Complete reference for OpenAPI 3.1.0 specification structure, data types, and advanced features.

## Table of Contents

1. [Document Structure](#document-structure)
2. [Info Object](#info-object)
3. [Server Object](#server-object)
4. [Paths and Operations](#paths-and-operations)
5. [Parameters](#parameters)
6. [Request Body](#request-body)
7. [Responses](#responses)
8. [Components and Reuse](#components-and-reuse)
9. [Schema Object and Data Types](#schema-object-and-data-types)
10. [Security Schemes](#security-schemes)
11. [Advanced Features](#advanced-features)

---

## Document Structure

Top-level structure of an OpenAPI 3.1.0 document:

```yaml
openapi: 3.1.0          # Required — spec version
info: {}                 # Required — API metadata
servers: []              # API server URLs
tags: []                 # Logical groupings
security: []             # Global security requirements
paths: {}                # Required — endpoint definitions
components: {}           # Reusable schemas, parameters, responses
webhooks: {}             # Webhook definitions (3.1.0+)
```

## Info Object

```yaml
info:
  title: My API                          # Required
  description: |                         # Markdown supported
    API for managing resources.
  version: 1.0.0                         # Required — API version (not OpenAPI version)
  termsOfService: https://example.com/terms
  contact:
    name: API Support
    url: https://example.com/support
    email: support@example.com
  license:
    name: Apache 2.0
    url: https://www.apache.org/licenses/LICENSE-2.0
```

## Server Object

```yaml
servers:
  - url: http://localhost:8080/api/v1
    description: Local development

  - url: https://api.example.com/{version}
    description: Production
    variables:
      version:
        default: v1
        enum: [v1, v2]
        description: API version
```

## Paths and Operations

### Path Item

```yaml
paths:
  /resources:
    summary: Resource collection operations
    description: Manage the resource collection
    get:    {}   # List resources
    post:   {}   # Create resource
  /resources/{resourceId}:
    get:    {}   # Get single resource
    put:    {}   # Replace resource
    patch:  {}   # Partial update
    delete: {}   # Delete resource
```

### Operation Object

```yaml
get:
  tags: [Resources]
  operationId: listResources          # Unique across the entire spec
  summary: List all resources         # Short (< 80 chars)
  description: |                      # Detailed, markdown supported
    Returns a paginated list of resources.
  deprecated: false
  parameters: []
  requestBody: {}
  responses: {}
  security:
    - bearerAuth: []
  callbacks: {}
  servers: []                         # Override global servers
```

## Parameters

### Parameter Locations

| Location | `in` value | Description |
|----------|-----------|-------------|
| Path | `path` | Part of the URL path (`/users/{userId}`) — always required |
| Query | `query` | After `?` in URL (`?page=0&size=20`) |
| Header | `header` | HTTP request header (`X-Request-Id`) |
| Cookie | `cookie` | HTTP cookie |

### Parameter Object

```yaml
parameters:
  - name: userId
    in: path
    required: true                    # Always true for path params
    description: Unique user identifier
    schema:
      type: string
      format: uuid
    example: "550e8400-e29b-41d4-a716-446655440000"

  - name: status
    in: query
    required: false
    description: Filter by status
    schema:
      type: string
      enum: [ACTIVE, INACTIVE, SUSPENDED]
    example: ACTIVE

  - name: tags
    in: query
    description: Filter by multiple tags
    schema:
      type: array
      items:
        type: string
    style: form                       # Serialization style
    explode: true                     # ?tags=a&tags=b vs ?tags=a,b

  - name: X-Request-Id
    in: header
    required: false
    description: Correlation ID for request tracing
    schema:
      type: string
      format: uuid
```

### Serialization Styles

| Location | Style | Explode | Example |
|----------|-------|---------|---------|
| query | `form` (default) | true | `?color=blue&color=red` |
| query | `form` | false | `?color=blue,red` |
| path | `simple` (default) | false | `/users/3,4,5` |
| path | `label` | false | `/users/.3.4.5` |
| path | `matrix` | false | `/users/;id=3,4,5` |
| header | `simple` (default) | false | `X-Values: 3,4,5` |

## Request Body

```yaml
requestBody:
  required: true
  description: User data to create
  content:
    application/json:
      schema:
        $ref: '#/components/schemas/CreateUserRequest'
      example:
        email: "john@example.com"
        fullName: "John Doe"
      examples:
        basic:
          summary: Basic user
          value:
            email: "john@example.com"
            fullName: "John Doe"
        admin:
          summary: Admin user
          value:
            email: "admin@example.com"
            fullName: "Admin User"
            role: ADMIN

    # Multipart file upload
    multipart/form-data:
      schema:
        type: object
        properties:
          file:
            type: string
            format: binary
            description: File to upload
          description:
            type: string
        required: [file]
      encoding:
        file:
          contentType: image/png, image/jpeg
```

## Responses

```yaml
responses:
  '200':
    description: Successful response             # Required
    content:
      application/json:
        schema:
          $ref: '#/components/schemas/UserResponse'
        example:
          id: "550e8400-e29b-41d4-a716-446655440000"
          email: "john@example.com"
    headers:
      X-RateLimit-Remaining:
        description: Number of requests remaining
        schema:
          type: integer

  '201':
    description: Resource created
    headers:
      Location:
        description: URL of the created resource
        schema:
          type: string
          format: uri

  '204':
    description: Successful operation with no content

  '301':
    description: Resource permanently moved
    headers:
      Location:
        schema:
          type: string
          format: uri

  # Default response for unhandled status codes
  default:
    description: Unexpected error
    content:
      application/json:
        schema:
          $ref: '#/components/schemas/ErrorResponse'
```

## Components and Reuse

### Referencing Components

Use `$ref` to reference any reusable component:

```yaml
# In paths
parameters:
  - $ref: '#/components/parameters/PageParam'
responses:
  '401':
    $ref: '#/components/responses/Unauthorized'
requestBody:
  $ref: '#/components/requestBodies/CreateUser'

# In schemas
properties:
  address:
    $ref: '#/components/schemas/Address'
```

### Component Categories

```yaml
components:
  schemas: {}           # Data models
  parameters: {}        # Reusable parameters
  responses: {}         # Reusable responses
  requestBodies: {}     # Reusable request bodies
  headers: {}           # Reusable headers
  securitySchemes: {}   # Authentication methods
  links: {}             # Relationships between operations
  callbacks: {}         # Webhook definitions
  pathItems: {}         # Reusable path items (3.1.0+)
```

## Schema Object and Data Types

### Primitive Types

| Type | Format | Description | Example |
|------|--------|-------------|---------|
| `string` | — | Plain text | `"hello"` |
| `string` | `date` | ISO 8601 date | `"2026-01-15"` |
| `string` | `date-time` | ISO 8601 datetime | `"2026-01-15T10:30:00Z"` |
| `string` | `email` | Email address | `"user@example.com"` |
| `string` | `uuid` | UUID v4 | `"550e8400-..."` |
| `string` | `uri` | URI | `"https://example.com"` |
| `string` | `password` | Masked in docs | `"********"` |
| `string` | `binary` | Binary file | — |
| `string` | `byte` | Base64-encoded | — |
| `integer` | `int32` | 32-bit signed | `42` |
| `integer` | `int64` | 64-bit signed | `9223372036854775807` |
| `number` | `float` | 32-bit float | `3.14` |
| `number` | `double` | 64-bit float | `3.141592653589793` |
| `boolean` | — | True/false | `true` |
| `null` | — | Null value (3.1.0) | `null` |

### String Constraints

```yaml
type: string
minLength: 2
maxLength: 100
pattern: "^[a-zA-Z0-9_]+$"
enum: [ACTIVE, INACTIVE, SUSPENDED]
```

### Numeric Constraints

```yaml
type: integer
minimum: 0
maximum: 100
exclusiveMinimum: 0      # 3.1.0: value (not boolean)
exclusiveMaximum: 100
multipleOf: 5
```

### Array Schema

```yaml
type: array
items:
  $ref: '#/components/schemas/Item'
minItems: 1
maxItems: 50
uniqueItems: true
```

### Object Schema

```yaml
type: object
required: [email, fullName]
properties:
  email:
    type: string
    format: email
  fullName:
    type: string
  metadata:
    type: object
    additionalProperties:
      type: string          # Map<String, String>
minProperties: 1
maxProperties: 10
```

### Composition — allOf, oneOf, anyOf

```yaml
# allOf — Merge schemas (inheritance/mixins)
AdminUser:
  allOf:
    - $ref: '#/components/schemas/UserResponse'
    - type: object
      required: [permissions]
      properties:
        permissions:
          type: array
          items:
            type: string

# oneOf — Exactly one schema matches (polymorphism)
PaymentMethod:
  oneOf:
    - $ref: '#/components/schemas/CreditCard'
    - $ref: '#/components/schemas/BankTransfer'
    - $ref: '#/components/schemas/DigitalWallet'
  discriminator:
    propertyName: type
    mapping:
      credit_card: '#/components/schemas/CreditCard'
      bank_transfer: '#/components/schemas/BankTransfer'
      digital_wallet: '#/components/schemas/DigitalWallet'

# anyOf — One or more schemas match
SearchResult:
  anyOf:
    - $ref: '#/components/schemas/UserResult'
    - $ref: '#/components/schemas/OrderResult'
```

### Discriminator

Used with `oneOf`/`anyOf` to indicate which schema variant is in use:

```yaml
discriminator:
  propertyName: type       # Property that determines the variant
  mapping:                 # Optional — explicit type-to-schema mapping
    dog: '#/components/schemas/Dog'
    cat: '#/components/schemas/Cat'
```

### Nullable Fields (3.1.0)

```yaml
# OpenAPI 3.1.0 — use type array
type: [string, "null"]

# OpenAPI 3.0.x — use nullable keyword
type: string
nullable: true
```

### readOnly and writeOnly

```yaml
properties:
  id:
    type: string
    format: uuid
    readOnly: true        # Only in responses, ignored in requests
  password:
    type: string
    writeOnly: true       # Only in requests, hidden in responses
```

## Security Schemes

### Bearer Token (JWT)

```yaml
securitySchemes:
  bearerAuth:
    type: http
    scheme: bearer
    bearerFormat: JWT
```

### API Key

```yaml
securitySchemes:
  apiKey:
    type: apiKey
    in: header              # header, query, or cookie
    name: X-API-Key
```

### OAuth 2.0

```yaml
securitySchemes:
  oauth2:
    type: oauth2
    flows:
      authorizationCode:
        authorizationUrl: https://auth.example.com/authorize
        tokenUrl: https://auth.example.com/token
        refreshUrl: https://auth.example.com/refresh
        scopes:
          read:users: Read user information
          write:users: Create and modify users
          admin: Full administrative access
      clientCredentials:
        tokenUrl: https://auth.example.com/token
        scopes:
          read:users: Read user information
```

### Applying Security

```yaml
# Global — applies to all operations
security:
  - bearerAuth: []

# Per-operation — override global
paths:
  /public/health:
    get:
      security: []           # No auth required (override global)
  /admin/users:
    delete:
      security:
        - oauth2: [admin]    # Requires admin scope
```

## Advanced Features

### Links (HATEOAS)

Define relationships between operations:

```yaml
responses:
  '201':
    description: User created
    content:
      application/json:
        schema:
          $ref: '#/components/schemas/UserResponse'
    links:
      GetUserById:
        operationId: getUserById
        parameters:
          userId: '$response.body#/id'
        description: Get the created user by ID
```

### Callbacks (Webhooks)

Define callback URLs that the API will call:

```yaml
post:
  operationId: createSubscription
  callbacks:
    onEvent:
      '{$request.body#/callbackUrl}':
        post:
          summary: Webhook notification
          requestBody:
            content:
              application/json:
                schema:
                  $ref: '#/components/schemas/WebhookEvent'
          responses:
            '200':
              description: Callback processed
```

### Webhooks (3.1.0)

Top-level webhook definitions:

```yaml
webhooks:
  orderCompleted:
    post:
      summary: Order completed notification
      tags: [Webhooks]
      requestBody:
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/OrderCompletedEvent'
      responses:
        '200':
          description: Webhook processed
```

### Tags with External Docs

```yaml
tags:
  - name: Users
    description: User management
    externalDocs:
      description: User management guide
      url: https://docs.example.com/users
```

### External Documentation

```yaml
externalDocs:
  description: Full API documentation
  url: https://docs.example.com
```
