---
name: documentation
description: Generate comprehensive technical documentation including API documentation (OpenAPI/Swagger), component documentation (props, usage, examples), and architecture decision records (ADRs). Use when documenting endpoints, creating component libraries, explaining system design decisions, or maintaining technical specifications. Apply when code needs clear documentation for team collaboration and knowledge sharing.
---

# Technical Documentation Generation

Create clear, comprehensive documentation that enables team collaboration and knowledge sharing.

## Documentation Types

### 1. API Documentation (OpenAPI/Swagger)

Document REST APIs using OpenAPI 3.x specification format.

**When to create**: For every API endpoint to ensure contract clarity.

**Format**:

```yaml
openapi: 3.0.0
info:
  title: User Management API
  version: 1.0.0
  description: API for user authentication and management

paths:
  /api/v1/users:
    get:
      summary: List all users
      description: Retrieve paginated list of users with optional filtering
      tags:
        - Users
      parameters:
        - name: page
          in: query
          description: Page number for pagination
          required: false
          schema:
            type: integer
            default: 1
            minimum: 1
        - name: limit
          in: query
          description: Number of items per page
          required: false
          schema:
            type: integer
            default: 20
            maximum: 100
        - name: status
          in: query
          description: Filter by user status
          required: false
          schema:
            type: string
            enum: [active, inactive, pending]
      responses:
        '200':
          description: Successfully retrieved user list
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/UserListResponse'
              example:
                users:
                  - id: 1
                    email: "user@example.com"
                    status: "active"
                total: 100
                page: 1
                limit: 20
        '401':
          description: Unauthorized - Invalid or missing authentication token
        '500':
          description: Internal server error

    post:
      summary: Create new user
      description: Register a new user account
      tags:
        - Users
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/CreateUserRequest'
            example:
              email: "newuser@example.com"
              password: "securePassword123"
              name: "John Doe"
      responses:
        '201':
          description: User created successfully
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/User'
        '400':
          description: Bad request - Validation error
          content:
            application/json:
              schema:
                type: object
                properties:
                  error:
                    type: string
                  details:
                    type: array
                    items:
                      type: string
        '409':
          description: Conflict - Email already exists

components:
  schemas:
    User:
      type: object
      required:
        - id
        - email
        - status
      properties:
        id:
          type: integer
          description: Unique user identifier
        email:
          type: string
          format: email
          description: User email address
        name:
          type: string
          description: User full name
        status:
          type: string
          enum: [active, inactive, pending]
          description: Current user status
        createdAt:
          type: string
          format: date-time
          description: Account creation timestamp

    UserListResponse:
      type: object
      properties:
        users:
          type: array
          items:
            $ref: '#/components/schemas/User'
        total:
          type: integer
        page:
          type: integer
        limit:
          type: integer

    CreateUserRequest:
      type: object
      required:
        - email
        - password
      properties:
        email:
          type: string
          format: email
        password:
          type: string
          minLength: 8
        name:
          type: string

  securitySchemes:
    BearerAuth:
      type: http
      scheme: bearer
      bearerFormat: JWT

security:
  - BearerAuth: []
```

**Key elements**:
- Clear endpoint summaries and descriptions
- Complete parameter documentation (type, required, validation)
- Response schemas for all status codes
- Request body examples
- Reusable component schemas
- Authentication/security requirements

### 2. Component Documentation (Frontend)

Document reusable UI components for team consistency.

**Vue 3 Example**:

```vue
<script setup lang="ts">
/**
 * UserProfile Component
 *
 * Displays user profile information with edit capability.
 * Handles loading states and error display automatically.
 *
 * @component
 *
 * @example
 * <UserProfile
 *   :user="currentUser"
 *   :editable="true"
 *   @update="handleUserUpdate"
 * />
 */

interface User {
  id: number
  name: string
  email: string
  avatar?: string
}

interface Props {
  /** User object to display */
  user: User
  /** Enable editing mode (default: false) */
  editable?: boolean
  /** Show avatar image (default: true) */
  showAvatar?: boolean
}

interface Emits {
  /** Emitted when user data is updated */
  (e: 'update', user: User): void
  /** Emitted when avatar is clicked */
  (e: 'avatar-click'): void
}

const props = withDefaults(defineProps<Props>(), {
  editable: false,
  showAvatar: true
})

const emit = defineEmits<Emits>()
</script>

<template>
  <!-- Component template -->
</template>

<style scoped>
/* Component styles */
</style>
```

**React/TypeScript Example**:

```typescript
/**
 * UserProfile Component
 *
 * Displays user profile information with optional editing.
 *
 * @component
 * @example
 * ```tsx
 * <UserProfile
 *   user={currentUser}
 *   editable={true}
 *   onUpdate={handleUpdate}
 * />
 * ```
 */

interface User {
  id: number
  name: string
  email: string
  avatar?: string
}

interface UserProfileProps {
  /** User object to display */
  user: User
  /** Enable editing mode */
  editable?: boolean
  /** Show avatar image */
  showAvatar?: boolean
  /** Callback when user data is updated */
  onUpdate?: (user: User) => void
  /** Callback when avatar is clicked */
  onAvatarClick?: () => void
}

export const UserProfile: React.FC<UserProfileProps> = ({
  user,
  editable = false,
  showAvatar = true,
  onUpdate,
  onAvatarClick
}) => {
  // Component implementation
}
```

**Documentation checklist**:
- [ ] Component purpose and use case
- [ ] Props/properties with types and descriptions
- [ ] Default values clearly indicated
- [ ] Event emissions documented
- [ ] Usage examples provided
- [ ] Edge cases and limitations noted

### 3. Architecture Decision Records (ADR)

Document significant architectural decisions for future reference.

**Template**:

```markdown
# ADR 001: Use JWT for Authentication

## Status

Accepted

Date: 2024-01-15

## Context

The application requires stateless authentication to support:
- Multiple frontend clients (web, mobile)
- Horizontal scaling without shared session storage
- API access for third-party integrations

Current session-based authentication:
- Requires sticky sessions or shared session storage (Redis)
- Complicates deployment and scaling
- Doesn't support API token access patterns

## Decision

Implement JWT (JSON Web Tokens) for authentication with:
- Access tokens: Short-lived (15 minutes), used for API requests
- Refresh tokens: Long-lived (7 days), used to obtain new access tokens
- Token storage: HttpOnly cookies for web, secure storage for mobile

## Consequences

### Positive

- **Stateless**: No server-side session storage required
- **Scalable**: Easy horizontal scaling without sticky sessions
- **Flexible**: Same tokens work across web, mobile, and API clients
- **Performance**: Reduced database queries (token validation via signature)

### Negative

- **Revocation complexity**: Cannot instantly revoke tokens before expiration
  - Mitigation: Short access token lifetime (15 min) + token blacklist for critical cases
- **Token size**: JWTs larger than session IDs in cookies
  - Impact: Minimal (~200 bytes extra per request)
- **Clock skew**: Server time differences can cause validation issues
  - Mitigation: Use `nbf` (not before) with clock skew tolerance

### Neutral

- Requires implementing refresh token rotation for security
- Need to add token blacklist table for forced logout
- Must handle token expiration gracefully in client apps

## Implementation Notes

- Use RS256 algorithm (asymmetric) for better security
- Store public key in frontend for client-side validation
- Include user permissions in token claims to avoid DB queries
- Implement automatic token refresh 5 minutes before expiration

## Alternatives Considered

### Session-based authentication + Redis
- Rejected: Adds infrastructure complexity and Redis as single point of failure
- Still requires handling distributed session synchronization

### OAuth 2.0 only
- Rejected: Overkill for current needs, complex implementation
- May reconsider when adding social login

## References

- [JWT Best Practices](https://tools.ietf.org/html/rfc8725)
- [OWASP Authentication Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html)
```

**ADR Structure**:
1. **Title**: Decision summary with unique number
2. **Status**: Proposed / Accepted / Deprecated / Superseded
3. **Context**: Problem being solved, background, constraints
4. **Decision**: What was decided and key implementation details
5. **Consequences**: Positive, negative, and neutral impacts
6. **Alternatives Considered**: Other options evaluated and why rejected
7. **References**: Relevant documentation and resources

**When to create an ADR**:
- Technology/framework selection
- Architecture pattern adoption
- Database schema design
- Security approach decisions
- Significant refactoring plans
- API design conventions

## Documentation Best Practices

### 1. Keep Documentation Close to Code

```
src/
├── components/
│   ├── UserProfile/
│   │   ├── UserProfile.vue
│   │   └── README.md        # Component-specific docs
├── api/
│   ├── users.ts
│   └── users.openapi.yaml   # API spec
└── docs/
    └── adr/
        ├── 001-jwt-auth.md
        └── 002-database-selection.md
```

### 2. Use Clear Examples

Always include practical, copy-pasteable examples:

```typescript
// ❌ BAD: Vague description
// Fetches user data

// ✅ GOOD: Clear purpose with example
/**
 * Fetches user profile with related posts and comments.
 *
 * @example
 * const user = await fetchUserProfile(123)
 * console.log(user.posts.length) // Number of user posts
 */
```

### 3. Document Edge Cases and Limitations

```typescript
/**
 * Validates email format.
 *
 * @param email - Email address to validate
 * @returns true if valid format
 *
 * @remarks
 * - Uses basic regex, doesn't verify deliverability
 * - Accepts international characters in domain
 * - Maximum length: 254 characters (RFC 5321)
 *
 * @example
 * validateEmail("user@example.com") // true
 * validateEmail("invalid.email")    // false
 */
```

### 4. Maintain Documentation

- Update docs with code changes (same PR)
- Mark deprecated APIs clearly
- Remove outdated documentation
- Regular documentation audits

## Tools and Automation

### Generate API Docs from FastAPI

```python
# Automatically generates OpenAPI spec at /docs
from fastapi import FastAPI

app = FastAPI(
    title="My API",
    description="API documentation",
    version="1.0.0"
)

# Export spec to file
import json
with open("openapi.json", "w") as f:
    json.dump(app.openapi(), f, indent=2)
```

### Generate Component Docs

- **Vue**: Use vue-docgen for automatic prop extraction
- **React**: Use react-docgen or Storybook
- **TypeScript**: Leverage JSDoc with TypeDoc

## Summary

Good documentation:
- Lives close to the code it documents
- Includes practical examples
- Explains the "why" not just the "what"
- Stays up-to-date with code changes
- Documents decisions and trade-offs
