---
name: api-design
description: Guide REST API design following best practices and team standards.
allowed-tools: Read, Edit, Grep
---

# API Design Skill

## RESTful Principles

### Resource-Based URLs
```
GET    /api/v1/users           # List users
POST   /api/v1/users           # Create user
GET    /api/v1/users/:id       # Get user
PUT    /api/v1/users/:id       # Update user
DELETE /api/v1/users/:id       # Delete user
```

### HTTP Status Codes
- **200 OK**: Successful request
- **201 Created**: Resource created
- **400 Bad Request**: Invalid input
- **401 Unauthorized**: Authentication required
- **403 Forbidden**: Authorization failed
- **404 Not Found**: Resource not found
- **500 Internal Server Error**: Server error

## Response Format
```json
{
  "success": true,
  "data": { "id": "123", "name": "John" },
  "timestamp": "2024-01-04T10:30:00Z"
}
```

## Error Response
```json
{
  "success": false,
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Invalid email format",
    "details": [{"field": "email", "message": "Must be valid"}]
  }
}
```

## Best Practices
- Use nouns for resources, not verbs
- Implement pagination (limit, offset)
- Version your API (/api/v1/)
- Use filtering (?status=active)
- Document with OpenAPI/Swagger
