---
name: code-review
description: Review code for quality, readability, and best practices.
allowed-tools: Read, Grep
---

# Code Review Skill

## Checklist

### Code Quality
- [ ] Clear variable and function names
- [ ] No duplicated code (DRY principle)
- [ ] Single responsibility principle
- [ ] Proper error handling
- [ ] Comments where needed (complex logic)

### Security
- [ ] No exposed secrets or API keys
- [ ] Input validation present
- [ ] No SQL/NoSQL injection risks
- [ ] Secure authentication/authorization

### Performance
- [ ] No N+1 query problems
- [ ] Efficient algorithms used
- [ ] Proper caching strategy
- [ ] No memory leaks

### Testing
- [ ] Unit tests present
- [ ] Edge cases covered
- [ ] Test coverage adequate

## Review Format
Organize feedback by priority:
1. **Critical** (must fix): Security issues, data corruption risks
2. **Warnings** (should fix): Performance issues, code quality
3. **Suggestions** (consider): Best practices, style improvements
