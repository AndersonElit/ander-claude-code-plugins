# Common API Documentation Patterns

Reusable patterns for API documentation. Each pattern includes the OpenAPI specification, rationale, and variations.

## Table of Contents

1. [Pagination Patterns](#pagination-patterns)
2. [Filtering and Sorting](#filtering-and-sorting)
3. [Error Handling](#error-handling)
4. [Authentication Patterns](#authentication-patterns)
5. [File Upload and Download](#file-upload-and-download)
6. [Versioning Strategies](#versioning-strategies)
7. [Bulk Operations](#bulk-operations)
8. [Long-Running Operations](#long-running-operations)
9. [HATEOAS and Hypermedia](#hateoas-and-hypermedia)
10. [Rate Limiting](#rate-limiting)

---

## Pagination Patterns

### Offset-Based Pagination (Spring Data style)

Most common for traditional REST APIs.

```yaml
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

schemas:
  PageMetadata:
    type: object
    properties:
      number:
        type: integer
        description: Current page number (zero-based)
      size:
        type: integer
        description: Items per page
      totalElements:
        type: integer
        format: int64
        description: Total items across all pages
      totalPages:
        type: integer
        description: Total number of pages
      first:
        type: boolean
        description: Whether this is the first page
      last:
        type: boolean
        description: Whether this is the last page

  PagedResponse:
    type: object
    properties:
      content:
        type: array
        items: {}       # Override with specific type
      page:
        $ref: '#/components/schemas/PageMetadata'
```

### Cursor-Based Pagination

Better for real-time feeds, large datasets, or when data changes frequently.

```yaml
parameters:
  CursorParam:
    name: cursor
    in: query
    description: Opaque cursor for the next page (from previous response)
    schema:
      type: string
  LimitParam:
    name: limit
    in: query
    description: Maximum number of items to return
    schema:
      type: integer
      minimum: 1
      maximum: 100
      default: 20

schemas:
  CursorPageMetadata:
    type: object
    properties:
      nextCursor:
        type: [string, "null"]
        description: Cursor for the next page (null if no more pages)
      previousCursor:
        type: [string, "null"]
        description: Cursor for the previous page
      hasMore:
        type: boolean
        description: Whether more items exist after this page

  CursorPagedResponse:
    type: object
    properties:
      data:
        type: array
        items: {}
      pagination:
        $ref: '#/components/schemas/CursorPageMetadata'
```

**When to use which:**

| Pattern | Use When |
|---------|----------|
| Offset-based | UI has page numbers, data is relatively static, total count is needed |
| Cursor-based | Infinite scroll, real-time feeds, large/frequently changing datasets |

## Filtering and Sorting

### Query Parameter Filtering

```yaml
parameters:
  - name: status
    in: query
    description: Filter by status
    schema:
      type: string
      enum: [ACTIVE, INACTIVE, SUSPENDED]

  - name: createdAfter
    in: query
    description: Filter items created after this date (inclusive)
    schema:
      type: string
      format: date-time

  - name: createdBefore
    in: query
    description: Filter items created before this date (exclusive)
    schema:
      type: string
      format: date-time

  - name: minAmount
    in: query
    description: Minimum amount (inclusive)
    schema:
      type: number
      format: double

  - name: maxAmount
    in: query
    description: Maximum amount (inclusive)
    schema:
      type: number
      format: double
```

### Sorting

```yaml
parameters:
  SortParam:
    name: sort
    in: query
    description: |
      Sort field and direction. Format: `field,direction`.
      Multiple sort criteria separated by `&sort=`.
      Direction: `asc` (default) or `desc`.
    schema:
      type: array
      items:
        type: string
    style: form
    explode: true
    examples:
      single:
        summary: Sort by creation date descending
        value: ["createdAt,desc"]
      multiple:
        summary: Sort by status ascending then name
        value: ["status,asc", "name,desc"]
```

### Search

```yaml
parameters:
  SearchParam:
    name: q
    in: query
    description: |
      Full-text search query. Searches across name, email, and description fields.
      Minimum 2 characters. Case-insensitive.
    schema:
      type: string
      minLength: 2
      maxLength: 200
```

## Error Handling

### Standard Error Response

```yaml
schemas:
  ErrorResponse:
    type: object
    required: [status, error, message, timestamp]
    properties:
      status:
        type: integer
        description: HTTP status code
      error:
        type: string
        description: Error category (e.g., "Bad Request", "Not Found")
      message:
        type: string
        description: Human-readable error description
      timestamp:
        type: string
        format: date-time
        description: When the error occurred
      path:
        type: string
        description: Request path that produced the error
      traceId:
        type: string
        description: Correlation ID for debugging (matches X-Trace-Id header)
      details:
        type: array
        items:
          $ref: '#/components/schemas/FieldError'

  FieldError:
    type: object
    properties:
      field:
        type: string
        description: JSON path to the field (e.g., "items[0].quantity")
      message:
        type: string
        description: Validation error message
      code:
        type: string
        description: Machine-readable error code (e.g., "FIELD_REQUIRED", "INVALID_FORMAT")
      rejectedValue:
        description: The value that was rejected
```

### RFC 7807 Problem Details

Alternative standard for error responses:

```yaml
schemas:
  ProblemDetail:
    type: object
    required: [type, title, status]
    properties:
      type:
        type: string
        format: uri
        description: URI reference identifying the problem type
        example: "https://api.example.com/problems/validation-error"
      title:
        type: string
        description: Short human-readable summary
        example: "Validation Error"
      status:
        type: integer
        description: HTTP status code
        example: 400
      detail:
        type: string
        description: Human-readable explanation specific to this occurrence
        example: "The request body contains 2 validation errors"
      instance:
        type: string
        format: uri
        description: URI reference identifying this specific occurrence
      errors:
        type: array
        items:
          $ref: '#/components/schemas/FieldError'
```

### Common Error Responses (Reusable)

```yaml
responses:
  BadRequest:
    description: Request validation failed
    content:
      application/json:
        schema:
          $ref: '#/components/schemas/ErrorResponse'
        examples:
          validation:
            summary: Validation error
            value:
              status: 400
              error: "Bad Request"
              message: "Validation failed: 2 errors"
              details:
                - field: "email"
                  message: "must be a valid email"
                  code: "INVALID_FORMAT"
                - field: "password"
                  message: "must be at least 8 characters"
                  code: "MIN_LENGTH"
          malformed:
            summary: Malformed JSON
            value:
              status: 400
              error: "Bad Request"
              message: "Malformed JSON in request body"

  Unauthorized:
    description: Authentication required or token invalid/expired
    content:
      application/json:
        schema:
          $ref: '#/components/schemas/ErrorResponse'
    headers:
      WWW-Authenticate:
        description: Authentication scheme
        schema:
          type: string
          example: 'Bearer realm="api", error="invalid_token"'

  Forbidden:
    description: Authenticated but insufficient permissions
    content:
      application/json:
        schema:
          $ref: '#/components/schemas/ErrorResponse'

  NotFound:
    description: Requested resource does not exist
    content:
      application/json:
        schema:
          $ref: '#/components/schemas/ErrorResponse'

  Conflict:
    description: Resource conflict (duplicate, state conflict)
    content:
      application/json:
        schema:
          $ref: '#/components/schemas/ErrorResponse'

  UnprocessableEntity:
    description: Request is syntactically valid but semantically incorrect
    content:
      application/json:
        schema:
          $ref: '#/components/schemas/ErrorResponse'

  TooManyRequests:
    description: Rate limit exceeded
    content:
      application/json:
        schema:
          $ref: '#/components/schemas/ErrorResponse'
    headers:
      Retry-After:
        description: Seconds to wait before retrying
        schema:
          type: integer
      X-RateLimit-Limit:
        description: Maximum requests per window
        schema:
          type: integer
      X-RateLimit-Remaining:
        description: Remaining requests in current window
        schema:
          type: integer
      X-RateLimit-Reset:
        description: Unix timestamp when the rate limit resets
        schema:
          type: integer
```

## Authentication Patterns

### JWT Bearer Token Flow

```yaml
paths:
  /auth/login:
    post:
      tags: [Authentication]
      operationId: login
      summary: Authenticate and obtain JWT tokens
      security: []                  # No auth required
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              required: [email, password]
              properties:
                email:
                  type: string
                  format: email
                password:
                  type: string
                  format: password
      responses:
        '200':
          description: Authentication successful
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/TokenResponse'

  /auth/refresh:
    post:
      tags: [Authentication]
      operationId: refreshToken
      summary: Refresh an expired access token
      security: []
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              required: [refreshToken]
              properties:
                refreshToken:
                  type: string
      responses:
        '200':
          description: New tokens issued
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/TokenResponse'

schemas:
  TokenResponse:
    type: object
    properties:
      accessToken:
        type: string
        description: JWT access token (short-lived)
      refreshToken:
        type: string
        description: Refresh token (long-lived, single-use)
      tokenType:
        type: string
        enum: [Bearer]
        description: Token type (always "Bearer")
      expiresIn:
        type: integer
        description: Access token TTL in seconds
        example: 3600
```

### OAuth 2.0 Scopes per Operation

```yaml
paths:
  /users:
    get:
      security:
        - oauth2: [read:users]
    post:
      security:
        - oauth2: [write:users]
  /admin/users:
    delete:
      security:
        - oauth2: [admin]
```

## File Upload and Download

### Single File Upload

```yaml
/documents:
  post:
    operationId: uploadDocument
    summary: Upload a document
    requestBody:
      required: true
      content:
        multipart/form-data:
          schema:
            type: object
            required: [file]
            properties:
              file:
                type: string
                format: binary
                description: Document file (PDF, DOCX, max 10MB)
              title:
                type: string
                description: Document title
              tags:
                type: array
                items:
                  type: string
          encoding:
            file:
              contentType: application/pdf, application/vnd.openxmlformats-officedocument.wordprocessingml.document
    responses:
      '201':
        description: Document uploaded
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/DocumentResponse'
```

### File Download

```yaml
/documents/{documentId}/download:
  get:
    operationId: downloadDocument
    summary: Download a document
    parameters:
      - name: documentId
        in: path
        required: true
        schema:
          type: string
          format: uuid
    responses:
      '200':
        description: Document file
        content:
          application/octet-stream:
            schema:
              type: string
              format: binary
        headers:
          Content-Disposition:
            description: File download metadata
            schema:
              type: string
              example: 'attachment; filename="report.pdf"'
          Content-Length:
            schema:
              type: integer
```

## Versioning Strategies

### URL Path Versioning (Recommended)

```yaml
servers:
  - url: https://api.example.com/v1
    description: Version 1
  - url: https://api.example.com/v2
    description: Version 2
```

### Header Versioning

```yaml
parameters:
  ApiVersion:
    name: X-API-Version
    in: header
    required: false
    description: API version (defaults to latest)
    schema:
      type: string
      enum: ["1.0", "2.0"]
      default: "2.0"
```

### Content Negotiation Versioning

```yaml
# Client sends: Accept: application/vnd.example.v2+json
responses:
  '200':
    content:
      application/vnd.example.v1+json:
        schema:
          $ref: '#/components/schemas/UserV1'
      application/vnd.example.v2+json:
        schema:
          $ref: '#/components/schemas/UserV2'
```

## Bulk Operations

### Batch Create

```yaml
/users/batch:
  post:
    operationId: createUsersBatch
    summary: Create multiple users in a single request
    requestBody:
      required: true
      content:
        application/json:
          schema:
            type: object
            required: [users]
            properties:
              users:
                type: array
                items:
                  $ref: '#/components/schemas/CreateUserRequest'
                minItems: 1
                maxItems: 100
    responses:
      '200':
        description: Batch result (may contain partial failures)
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/BatchResult'

schemas:
  BatchResult:
    type: object
    properties:
      total:
        type: integer
        description: Total items processed
      succeeded:
        type: integer
        description: Number of successful operations
      failed:
        type: integer
        description: Number of failed operations
      results:
        type: array
        items:
          type: object
          properties:
            index:
              type: integer
              description: Position in the original request array
            status:
              type: string
              enum: [SUCCESS, FAILED]
            id:
              type: string
              description: Created resource ID (if successful)
            error:
              $ref: '#/components/schemas/ErrorResponse'
              description: Error details (if failed)
```

## Long-Running Operations

### Async with Polling

```yaml
/reports:
  post:
    operationId: generateReport
    summary: Start report generation (async)
    responses:
      '202':
        description: Report generation started
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/AsyncOperation'
        headers:
          Location:
            description: URL to poll for status
            schema:
              type: string
              example: /api/v1/operations/op-123

/operations/{operationId}:
  get:
    operationId: getOperationStatus
    summary: Check async operation status
    responses:
      '200':
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/AsyncOperation'

schemas:
  AsyncOperation:
    type: object
    properties:
      operationId:
        type: string
      status:
        type: string
        enum: [PENDING, RUNNING, COMPLETED, FAILED]
      progress:
        type: integer
        minimum: 0
        maximum: 100
        description: Completion percentage
      createdAt:
        type: string
        format: date-time
      completedAt:
        type: string
        format: date-time
      resultUrl:
        type: string
        format: uri
        description: URL to fetch the result (when COMPLETED)
      error:
        $ref: '#/components/schemas/ErrorResponse'
        description: Error details (when FAILED)
```

## HATEOAS and Hypermedia

```yaml
schemas:
  UserResponse:
    type: object
    properties:
      id:
        type: string
        format: uuid
      email:
        type: string
      _links:
        type: object
        properties:
          self:
            $ref: '#/components/schemas/Link'
          orders:
            $ref: '#/components/schemas/Link'
          update:
            $ref: '#/components/schemas/Link'
          delete:
            $ref: '#/components/schemas/Link'

  Link:
    type: object
    properties:
      href:
        type: string
        format: uri
        description: URL of the linked resource
      method:
        type: string
        enum: [GET, POST, PUT, PATCH, DELETE]
        description: HTTP method to use
      title:
        type: string
        description: Human-readable link title
```

## Rate Limiting

### Rate Limit Headers

Document rate limiting in response headers:

```yaml
headers:
  X-RateLimit-Limit:
    description: Maximum number of requests allowed per time window
    schema:
      type: integer
      example: 1000
  X-RateLimit-Remaining:
    description: Number of requests remaining in the current window
    schema:
      type: integer
      example: 999
  X-RateLimit-Reset:
    description: Unix timestamp (seconds) when the rate limit window resets
    schema:
      type: integer
      example: 1706367600
  Retry-After:
    description: Seconds to wait before retrying (only present on 429 responses)
    schema:
      type: integer
      example: 60
```

### Documenting Rate Limits per Tier

Include in the API description or integration guide:

| Tier | Limit | Window | Applies To |
|------|-------|--------|------------|
| Free | 100 requests | Per hour | All endpoints |
| Standard | 1,000 requests | Per hour | All endpoints |
| Premium | 10,000 requests | Per hour | All endpoints |
| Burst | 50 requests | Per second | Write endpoints only |
