---
name: commit-messages
description: Generate clear, conventional commit messages following team standards. Use when creating git commits to ensure consistency with project conventions including type prefixes (feat/fix/docs/etc), scope identification, imperative mood, and proper issue references. Apply when user requests to commit changes or when git commit hooks trigger.
---

# Commit Message Generation

Generate conventional commit messages that follow team standards for clarity and consistency.

## Conventional Commits Format

```
<type>(<scope>): <subject>

<body>

<footer>
```

### Type Selection

Choose the appropriate type based on the nature of changes:

- **feat**: New feature or capability added
- **fix**: Bug fix or error correction
- **docs**: Documentation-only changes (README, comments, API docs)
- **style**: Code style changes (formatting, whitespace, no logic changes)
- **refactor**: Code restructuring without changing external behavior
- **perf**: Performance improvements
- **test**: Adding or updating tests
- **chore**: Build process, tooling, dependency updates

### Scope Guidelines

The scope identifies which part of the codebase is affected:

- Use module/package names: `auth`, `api`, `frontend`, `mobile`
- Be specific but concise: `user-service`, `payment-flow`
- Omit scope if changes span multiple areas or are global

### Subject Line

- Keep under 50 characters
- Use imperative mood: "add feature" not "added feature" or "adds feature"
- Don't capitalize first letter
- No period at the end
- Be specific about what changed

### Body (Optional but Recommended)

Include when the commit needs explanation beyond the subject:

- Explain the motivation for the change (the "why")
- Describe what problem it solves
- List key implementation details as bullet points
- Use present tense
- Wrap at 72 characters

### Footer (Optional)

- Reference issues: `Closes #123`, `Fixes #456`, `Relates to #789`
- Note breaking changes: `BREAKING CHANGE: description`
- Multiple issue references: `Closes #123, #456`

## Workflow

1. **Analyze changes**: Review git diff to understand scope and impact
2. **Identify type**: Determine primary change type (feat/fix/refactor/etc)
3. **Determine scope**: Identify affected module or component
4. **Write subject**: Craft concise, imperative description
5. **Add body if needed**: Explain motivation and key details for non-trivial changes
6. **Reference issues**: Link to relevant issue tracker items

## Examples

### Feature Addition
```
feat(api): add user authentication endpoint

Implement JWT-based authentication system to replace session-based auth:
- Add POST /api/v1/auth/login endpoint
- Implement password hashing with bcrypt
- Include rate limiting (5 attempts per minute)
- Return access token and refresh token

This provides stateless authentication needed for mobile app.

Closes #123
```

### Bug Fix
```
fix(frontend): resolve memory leak in component cleanup

Properly unsubscribe from observables in useEffect cleanup function.
The subscription was persisting after component unmount, causing
memory accumulation during navigation.

Fixes #456
```

### Refactoring
```
refactor(auth): extract token validation into middleware

Move JWT validation logic from individual route handlers into
reusable middleware to reduce code duplication and improve
maintainability.

No functional changes to API behavior.
```

### Documentation
```
docs(api): update authentication endpoint documentation

Add request/response examples and error code descriptions
to OpenAPI spec for all auth endpoints.
```

### Performance Improvement
```
perf(database): add compound index for user queries

Add index on (status, created_at) to optimize dashboard query.
Reduces query time from 450ms to 12ms for active user list.

Related to #789
```

## Common Patterns

### Multiple Related Changes
When changes span related areas, choose the most significant type:
```
feat(payment): implement checkout flow

- Add payment processing endpoint
- Create payment confirmation page
- Update order status tracking
- Add payment error handling

Closes #234, #235, #236
```

### Breaking Changes
Always document breaking changes in footer:
```
feat(api): change user ID format to UUID

BREAKING CHANGE: User IDs changed from integer to UUID string.
Clients must update type definitions and ID parsing logic.
Migration guide: docs/migrations/uuid-migration.md

Closes #567
```

### Quick Fixes
Even small fixes deserve proper context:
```
fix(frontend): correct typo in error message

Change "Authentification" to "Authentication" in login error display.
```

## Quality Checklist

Before finalizing the commit message:

- [ ] Type accurately reflects the change
- [ ] Scope is specific and relevant
- [ ] Subject is under 50 characters
- [ ] Subject uses imperative mood
- [ ] Body explains "why" not just "what" (for non-trivial changes)
- [ ] Issues are properly referenced
- [ ] Breaking changes are documented in footer

## Project-Specific Conventions

Adapt to project needs:
- Check existing commit history for team patterns
- Follow established scope naming conventions
- Match the level of detail used in recent commits
- Respect any emoji conventions if team uses them
