---
name: security-review
description: Review code for security vulnerabilities (OWASP Top 10).
allowed-tools: Read, Grep, Bash
---

# Security Review Skill

## OWASP Top 10 Checklist

### 1. Injection
- [ ] SQL/NoSQL queries use parameterized statements
- [ ] No eval() or exec() with user input
- [ ] Input validation and sanitization

### 2. Broken Authentication
- [ ] Passwords hashed with bcrypt/argon2
- [ ] JWT tokens properly configured
- [ ] Session management secure
- [ ] Rate limiting on auth endpoints

### 3. Sensitive Data Exposure
- [ ] No secrets in code
- [ ] HTTPS enforced
- [ ] Sensitive data encrypted at rest
- [ ] No sensitive info in logs

### 4. XSS Prevention
- [ ] User input escaped/sanitized
- [ ] CSP headers configured
- [ ] Frontend sanitization library used

### 5. Access Control
- [ ] Authorization checks present
- [ ] Principle of least privilege
- [ ] No direct object references

## Severity Levels
- **Critical**: Immediate security risk
- **High**: Significant vulnerability
- **Medium**: Moderate risk
- **Low**: Minor issue
