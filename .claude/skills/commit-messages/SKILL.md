---
name: commit-messages
description: Generate clear, conventional commit messages following team standards.
allowed-tools: Bash, Grep, Read
---

# Commit Message Skill

## Conventional Commits Format
```
<type>(<scope>): <subject>

<body>

<footer>
```

## Types
- **feat**: New feature
- **fix**: Bug fix
- **docs**: Documentation changes
- **style**: Code style (formatting)
- **refactor**: Code refactoring
- **perf**: Performance improvements
- **test**: Adding/updating tests
- **chore**: Build/tooling changes

## Examples
```
feat(api): add user authentication endpoint

- Implement JWT-based authentication
- Add password hashing with bcrypt
- Include rate limiting

Closes #123
```

```
fix(frontend): resolve memory leak in component cleanup

Properly unsubscribe from observables in useEffect cleanup.

Fixes #456
```

## Requirements
- Subject < 50 characters
- Imperative mood ("add" not "added")
- Body explains "why" not "what"
- Reference issues with "Fixes" or "Closes"
