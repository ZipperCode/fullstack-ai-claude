---
name: documentation
description: Generate technical documentation (API docs, component docs, architecture docs).
allowed-tools: Read, Write, Grep
---

# Documentation Skill

## Documentation Types

### API Documentation
Use OpenAPI/Swagger format:
```yaml
/api/v1/users:
  get:
    summary: List users
    parameters:
      - name: page
        in: query
        type: integer
    responses:
      200:
        description: Success
        schema: UserListResponse
```

### Component Documentation
```vue
<!--
@component UserProfile
@description Displays user profile information
@prop {User} user - User object
@prop {boolean} editable - Enable editing mode
@example
<UserProfile :user="currentUser" :editable="true" />
-->
```

### Architecture Decisions (ADR)
```markdown
# ADR 001: Use JWT for Authentication

## Status
Accepted

## Context
Need stateless authentication for API

## Decision
Use JWT tokens with 24h expiration

## Consequences
- Pros: Stateless, scalable
- Cons: Cannot revoke before expiration
```
