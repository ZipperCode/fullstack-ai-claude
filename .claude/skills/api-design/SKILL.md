---
name: api-design
description: Guide REST API design following best practices and team standards. Use when designing, implementing, or reviewing REST APIs to ensure adherence to modern API design principles including: (1) RESTful resource naming and HTTP method usage, (2) Consistent response formats and error handling, (3) Authentication and authorization patterns, (4) Pagination, filtering, and sorting strategies, (5) API versioning approaches, (6) Rate limiting and security best practices, (7) Request/response validation, (8) OpenAPI/Swagger documentation. Apply when creating new endpoints, refactoring existing APIs, or when asked about API architecture decisions.
---

# REST API Design Guidelines

## Core RESTful Principles

### Resource Naming Conventions

**Use nouns, not verbs:**

```
✅ Good:
GET    /api/v1/users
POST   /api/v1/users
GET    /api/v1/users/:id
PUT    /api/v1/users/:id
PATCH  /api/v1/users/:id
DELETE /api/v1/users/:id

❌ Bad:
GET    /api/v1/getUsers
POST   /api/v1/createUser
GET    /api/v1/getUserById/:id
```

**Use plural nouns for collections:**

```
✅ Good: /api/v1/users, /api/v1/products
❌ Bad: /api/v1/user, /api/v1/product
```

**Nested resources for relationships:**

```
GET    /api/v1/users/:userId/posts           # Get user's posts
POST   /api/v1/users/:userId/posts           # Create post for user
GET    /api/v1/users/:userId/posts/:postId   # Get specific post
DELETE /api/v1/users/:userId/posts/:postId   # Delete user's post
```

**Limit nesting to 2 levels maximum:**

```
✅ Good: /api/v1/users/:userId/posts/:postId
❌ Bad: /api/v1/users/:userId/posts/:postId/comments/:commentId/likes
      → Use: /api/v1/comments/:commentId/likes
```

### HTTP Methods

| Method | Purpose | Idempotent | Safe | Request Body | Response Body |
|--------|---------|------------|------|--------------|---------------|
| GET | Retrieve resource(s) | ✅ | ✅ | ❌ | ✅ |
| POST | Create resource | ❌ | ❌ | ✅ | ✅ |
| PUT | Replace entire resource | ✅ | ❌ | ✅ | ✅ |
| PATCH | Update partial resource | ❌ | ❌ | ✅ | ✅ |
| DELETE | Delete resource | ✅ | ❌ | ❌ | ✅ |

### HTTP Status Codes

**Success Codes (2xx):**
- `200 OK` - Successful GET, PUT, PATCH, DELETE
- `201 Created` - Successful POST (resource created)
- `204 No Content` - Successful DELETE (no response body)

**Client Error Codes (4xx):**
- `400 Bad Request` - Invalid request body/parameters
- `401 Unauthorized` - Authentication required or failed
- `403 Forbidden` - Authenticated but not authorized
- `404 Not Found` - Resource doesn't exist
- `409 Conflict` - Resource conflict (e.g., duplicate email)
- `422 Unprocessable Entity` - Validation errors
- `429 Too Many Requests` - Rate limit exceeded

**Server Error Codes (5xx):**
- `500 Internal Server Error` - Unexpected server error
- `502 Bad Gateway` - Invalid upstream response
- `503 Service Unavailable` - Server overloaded or down
- `504 Gateway Timeout` - Upstream timeout

## Request/Response Formats

### Consistent Response Structure

**Success Response:**

```json
{
  "success": true,
  "data": {
    "id": "user_123",
    "email": "john@example.com",
    "name": "John Doe",
    "createdAt": "2024-01-04T10:30:00Z"
  },
  "meta": {
    "timestamp": "2024-01-04T10:30:00Z",
    "requestId": "req_abc123"
  }
}
```

**Collection Response with Pagination:**

```json
{
  "success": true,
  "data": [
    {"id": "user_1", "name": "John"},
    {"id": "user_2", "name": "Jane"}
  ],
  "pagination": {
    "page": 1,
    "limit": 20,
    "total": 150,
    "totalPages": 8,
    "hasNext": true,
    "hasPrevious": false
  },
  "meta": {
    "timestamp": "2024-01-04T10:30:00Z",
    "requestId": "req_abc124"
  }
}
```

**Error Response:**

```json
{
  "success": false,
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Request validation failed",
    "details": [
      {
        "field": "email",
        "message": "Must be a valid email address",
        "value": "invalid-email"
      },
      {
        "field": "age",
        "message": "Must be at least 18",
        "value": 15
      }
    ]
  },
  "meta": {
    "timestamp": "2024-01-04T10:30:00Z",
    "requestId": "req_abc125"
  }
}
```

### Standard Error Codes

```typescript
enum ErrorCode {
  // Validation (4xx)
  VALIDATION_ERROR = "VALIDATION_ERROR",
  INVALID_REQUEST = "INVALID_REQUEST",
  MISSING_PARAMETER = "MISSING_PARAMETER",

  // Authentication & Authorization (401, 403)
  UNAUTHORIZED = "UNAUTHORIZED",
  INVALID_TOKEN = "INVALID_TOKEN",
  TOKEN_EXPIRED = "TOKEN_EXPIRED",
  FORBIDDEN = "FORBIDDEN",
  INSUFFICIENT_PERMISSIONS = "INSUFFICIENT_PERMISSIONS",

  // Resource (404, 409)
  NOT_FOUND = "NOT_FOUND",
  ALREADY_EXISTS = "ALREADY_EXISTS",
  CONFLICT = "CONFLICT",

  // Rate Limiting (429)
  RATE_LIMIT_EXCEEDED = "RATE_LIMIT_EXCEEDED",

  // Server (5xx)
  INTERNAL_ERROR = "INTERNAL_ERROR",
  SERVICE_UNAVAILABLE = "SERVICE_UNAVAILABLE",
  DATABASE_ERROR = "DATABASE_ERROR"
}
```

## Pagination, Filtering, Sorting

### Pagination Strategies

**Offset-based (simple, for small datasets):**

```
GET /api/v1/users?page=2&limit=20

Response includes:
{
  "data": [...],
  "pagination": {
    "page": 2,
    "limit": 20,
    "total": 150,
    "totalPages": 8
  }
}
```

**Cursor-based (recommended for large datasets):**

```
GET /api/v1/users?cursor=eyJpZCI6MTIzfQ&limit=20

Response includes:
{
  "data": [...],
  "pagination": {
    "nextCursor": "eyJpZCI6MTQzfQ",
    "previousCursor": "eyJpZCI6MTAzfQ",
    "hasNext": true,
    "hasPrevious": true
  }
}
```

### Filtering

```
GET /api/v1/users?status=active&role=admin
GET /api/v1/products?category=electronics&minPrice=100&maxPrice=500
GET /api/v1/posts?authorId=user_123&tags=tech,programming

# Date range filtering
GET /api/v1/orders?startDate=2024-01-01&endDate=2024-01-31

# Search
GET /api/v1/users?search=john
```

### Sorting

```
GET /api/v1/users?sort=createdAt           # Ascending (default)
GET /api/v1/users?sort=-createdAt          # Descending (- prefix)
GET /api/v1/products?sort=price,-rating    # Multiple fields
```

### Field Selection (Sparse Fieldsets)

```
GET /api/v1/users?fields=id,name,email
GET /api/v1/products?fields=id,name,price&include=category,reviews
```

## Authentication & Authorization

### Authentication Patterns

**Bearer Token (JWT):**

```http
GET /api/v1/users/me
Authorization: Bearer eyJhbGciOiJIUzI1NiIs...

Response:
{
  "success": true,
  "data": {
    "id": "user_123",
    "email": "john@example.com",
    "roles": ["user", "admin"]
  }
}
```

**API Key:**

```http
GET /api/v1/data
X-API-Key: sk_live_abc123xyz

# Or as query parameter (less secure, avoid if possible)
GET /api/v1/data?apiKey=sk_live_abc123xyz
```

### Authorization Header Format

```
Authorization: Bearer <token>
Authorization: Basic <base64(username:password)>
Authorization: API-Key <api-key>
```

### Protected Endpoints Pattern

```typescript
// Middleware checks
GET /api/v1/users/me              → Requires: authenticated
GET /api/v1/admin/users           → Requires: authenticated + admin role
POST /api/v1/users/:id/suspend    → Requires: authenticated + (admin OR owner)
```

## API Versioning

### URL Versioning (Recommended)

```
/api/v1/users
/api/v2/users

Pros: Clear, easy to route, supports multiple versions simultaneously
Cons: URL changes
```

### Header Versioning

```http
GET /api/users
Accept: application/vnd.myapi.v2+json

Pros: Clean URLs
Cons: Less discoverable, harder to test
```

### Breaking Changes

**What requires a new version:**
- Removing fields from response
- Changing field types
- Changing endpoint behavior
- Removing endpoints
- Changing authentication

**What doesn't require a new version:**
- Adding new optional fields
- Adding new endpoints
- Adding new optional query parameters
- Bug fixes

## Rate Limiting

### Rate Limit Headers

```http
HTTP/1.1 200 OK
X-RateLimit-Limit: 1000          # Max requests per window
X-RateLimit-Remaining: 950       # Remaining requests
X-RateLimit-Reset: 1704362400    # Unix timestamp when limit resets
Retry-After: 3600                # Seconds until retry (on 429)
```

### Rate Limit Response (429)

```json
{
  "success": false,
  "error": {
    "code": "RATE_LIMIT_EXCEEDED",
    "message": "API rate limit exceeded",
    "details": {
      "limit": 1000,
      "remaining": 0,
      "resetAt": "2024-01-04T11:00:00Z"
    }
  }
}
```

### Rate Limiting Strategies

```
Per API Key:     1000 requests/hour
Per IP:          100 requests/minute
Per User:        5000 requests/hour
Per Endpoint:    10 requests/second (POST /api/v1/messages)
```

## Request Validation

### Input Validation

```typescript
// User creation request
POST /api/v1/users
{
  "email": "john@example.com",      // Required, valid email
  "password": "SecurePass123!",     // Required, min 8 chars, complexity rules
  "name": "John Doe",                // Required, 2-100 chars
  "age": 25,                         // Optional, >= 18 if provided
  "phone": "+1234567890"             // Optional, valid phone format
}

// Validation response (422)
{
  "success": false,
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Request validation failed",
    "details": [
      {
        "field": "email",
        "message": "Email already exists",
        "code": "DUPLICATE_VALUE"
      },
      {
        "field": "password",
        "message": "Password must contain at least one uppercase letter, one number, and one special character",
        "code": "INVALID_FORMAT"
      }
    ]
  }
}
```

### Query Parameter Validation

```typescript
GET /api/v1/users?page=abc&limit=1000

Response 400:
{
  "success": false,
  "error": {
    "code": "INVALID_REQUEST",
    "message": "Invalid query parameters",
    "details": [
      {
        "field": "page",
        "message": "Must be a positive integer",
        "value": "abc"
      },
      {
        "field": "limit",
        "message": "Must be between 1 and 100",
        "value": 1000
      }
    ]
  }
}
```

## Security Best Practices

### Request Security

```typescript
// Content-Type validation
POST /api/v1/users
Content-Type: application/json    ✅ Accept only expected types

// Input sanitization
{
  "name": "<script>alert('xss')</script>"  ❌ Sanitize HTML/scripts
}

// SQL/NoSQL injection prevention
// Use parameterized queries or ORM
GET /api/v1/users?email=admin' OR '1'='1  ❌ Validate and escape
```

### Response Security Headers

```http
HTTP/1.1 200 OK
Content-Type: application/json; charset=utf-8
X-Content-Type-Options: nosniff
X-Frame-Options: DENY
X-XSS-Protection: 1; mode=block
Strict-Transport-Security: max-age=31536000; includeSubDomains
Content-Security-Policy: default-src 'self'
```

### Sensitive Data

```json
// Never expose in responses:
❌ {
  "password": "hashed_value",
  "passwordHash": "$2b$10$...",
  "apiSecret": "sk_live_...",
  "internalId": "db_12345"
}

✅ {
  "id": "user_123",
  "email": "john@example.com",
  "name": "John Doe"
}

// Use separate endpoints for sensitive operations
POST /api/v1/users/:id/change-password
POST /api/v1/users/:id/regenerate-api-key
```

## CRUD Operation Patterns

### Create (POST)

```http
POST /api/v1/users
Content-Type: application/json

{
  "email": "john@example.com",
  "name": "John Doe"
}

Response 201:
{
  "success": true,
  "data": {
    "id": "user_123",
    "email": "john@example.com",
    "name": "John Doe",
    "createdAt": "2024-01-04T10:30:00Z"
  }
}
Location: /api/v1/users/user_123
```

### Read (GET)

```http
# Single resource
GET /api/v1/users/user_123

Response 200:
{
  "success": true,
  "data": {
    "id": "user_123",
    "email": "john@example.com",
    "name": "John Doe"
  }
}

# Collection
GET /api/v1/users?page=1&limit=20

Response 200:
{
  "success": true,
  "data": [...],
  "pagination": {...}
}
```

### Update (PUT/PATCH)

```http
# Full replacement (PUT)
PUT /api/v1/users/user_123
{
  "email": "john.new@example.com",
  "name": "John Smith"
}

# Partial update (PATCH)
PATCH /api/v1/users/user_123
{
  "name": "John Smith"
}

Response 200:
{
  "success": true,
  "data": {
    "id": "user_123",
    "email": "john.new@example.com",
    "name": "John Smith",
    "updatedAt": "2024-01-04T11:00:00Z"
  }
}
```

### Delete (DELETE)

```http
DELETE /api/v1/users/user_123

Response 204 No Content
# Or 200 with confirmation
{
  "success": true,
  "data": {
    "id": "user_123",
    "deleted": true,
    "deletedAt": "2024-01-04T11:30:00Z"
  }
}
```

## Bulk Operations

```http
# Bulk create
POST /api/v1/users/batch
{
  "users": [
    {"email": "user1@example.com", "name": "User 1"},
    {"email": "user2@example.com", "name": "User 2"}
  ]
}

Response 201:
{
  "success": true,
  "data": {
    "created": 2,
    "failed": 0,
    "items": [...]
  }
}

# Bulk update
PATCH /api/v1/users/batch
{
  "ids": ["user_1", "user_2"],
  "updates": {"status": "active"}
}

# Bulk delete
DELETE /api/v1/users/batch
{
  "ids": ["user_1", "user_2", "user_3"]
}
```

## API Documentation (OpenAPI)

### Basic OpenAPI Spec

```yaml
openapi: 3.0.0
info:
  title: User Management API
  version: 1.0.0
  description: API for managing users

servers:
  - url: https://api.example.com/v1
    description: Production
  - url: https://staging-api.example.com/v1
    description: Staging

paths:
  /users:
    get:
      summary: List all users
      parameters:
        - name: page
          in: query
          schema:
            type: integer
            default: 1
        - name: limit
          in: query
          schema:
            type: integer
            default: 20
            maximum: 100
      responses:
        '200':
          description: Successful response
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/UserListResponse'

    post:
      summary: Create a new user
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/CreateUserRequest'
      responses:
        '201':
          description: User created
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/UserResponse'
        '422':
          description: Validation error
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'

components:
  schemas:
    User:
      type: object
      properties:
        id:
          type: string
        email:
          type: string
          format: email
        name:
          type: string
        createdAt:
          type: string
          format: date-time

  securitySchemes:
    bearerAuth:
      type: http
      scheme: bearer
      bearerFormat: JWT

security:
  - bearerAuth: []
```

## Common Patterns Summary

**✅ Do:**
- Use nouns for resources, plural form
- Use HTTP methods correctly (GET, POST, PUT, PATCH, DELETE)
- Return appropriate status codes
- Implement pagination for collections
- Version your API
- Validate all inputs
- Use consistent response formats
- Document with OpenAPI
- Implement rate limiting
- Use proper authentication/authorization
- Return meaningful error messages

**❌ Don't:**
- Use verbs in URLs (e.g., /getUser, /createPost)
- Return 200 for errors
- Expose sensitive data in responses
- Allow unlimited pagination
- Break backwards compatibility without versioning
- Trust user input without validation
- Use query parameters for sensitive data
- Ignore security headers
- Mix authentication schemes
- Return stack traces in production

## Quick Checklist

- [ ] Resources use plural nouns
- [ ] HTTP methods used correctly
- [ ] Appropriate status codes returned
- [ ] Consistent response format (success/error)
- [ ] Pagination implemented for collections
- [ ] Filtering and sorting supported
- [ ] API versioning strategy in place
- [ ] Authentication/authorization implemented
- [ ] Rate limiting configured
- [ ] Input validation on all endpoints
- [ ] Error responses with meaningful messages
- [ ] Security headers configured
- [ ] Sensitive data not exposed
- [ ] OpenAPI documentation available
- [ ] CORS configured properly
