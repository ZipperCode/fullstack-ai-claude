---
name: security-review
description: Review code for security vulnerabilities based on OWASP Top 10 standards, including injection attacks, authentication flaws, sensitive data exposure, XSS, and access control issues. Use when reviewing code before deployment, auditing existing applications, implementing authentication/authorization, handling user input, or addressing security incidents. Apply for ensuring secure coding practices, preventing common vulnerabilities, and maintaining compliance with security standards. Critical for protecting applications and user data from security threats.
---

# Security Review

Systematically identify and remediate security vulnerabilities following OWASP Top 10 standards.

## OWASP Top 10 Security Risks

### 1. Injection Attacks

**Risk**: Attackers inject malicious code through untrusted input.

#### SQL/NoSQL Injection

```python
# ❌ BAD: Direct string interpolation (vulnerable)
query = f"SELECT * FROM users WHERE email = '{email}'"
result = db.execute(query)

# ❌ BAD: NoSQL injection risk
db.users.find({"email": email, "password": password})

# ✅ GOOD: Parameterized queries (SQL)
query = "SELECT * FROM users WHERE email = ?"
result = db.execute(query, (email,))

# ✅ GOOD: Proper validation (NoSQL)
from pydantic import BaseModel, EmailStr

class LoginRequest(BaseModel):
    email: EmailStr  # Validates email format
    password: str

# Query uses validated data
db.users.find_one({"email": login.email})
```

#### Command Injection

```python
# ❌ BAD: Unsafe shell command
import os
filename = request.query_params.get("file")
os.system(f"cat {filename}")  # Vulnerable to: file="; rm -rf /"

# ✅ GOOD: Avoid shell commands or use subprocess with list
import subprocess
result = subprocess.run(
    ["cat", filename],  # Array prevents injection
    capture_output=True,
    check=True
)
```

#### Code Injection

```python
# ❌ BAD: eval() with user input
user_code = request.json.get("expression")
result = eval(user_code)  # Extremely dangerous!

# ✅ GOOD: Use safe alternatives
import ast

def safe_eval(expression: str):
    # Only allow simple math expressions
    tree = ast.parse(expression, mode='eval')
    for node in ast.walk(tree):
        if not isinstance(node, (ast.Expression, ast.Num, ast.BinOp, ast.Add, ast.Sub)):
            raise ValueError("Invalid expression")
    return eval(compile(tree, '', 'eval'))
```

**Checklist**:
- [ ] No `eval()` or `exec()` with user input
- [ ] SQL queries use parameterized statements
- [ ] NoSQL queries validate input types
- [ ] Shell commands avoided or use subprocess with arrays
- [ ] Input validation on all user data

### 2. Broken Authentication

**Risk**: Weak authentication allows attackers to compromise accounts.

#### Password Security

```python
# ❌ BAD: Plain text passwords
user.password = password

# ❌ BAD: Weak hashing (MD5, SHA1)
import hashlib
hashed = hashlib.md5(password.encode()).hexdigest()

# ✅ GOOD: bcrypt/argon2 for password hashing
from passlib.context import CryptContext

pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")

def hash_password(password: str) -> str:
    return pwd_context.hash(password)

def verify_password(plain: str, hashed: str) -> bool:
    return pwd_context.verify(plain, hashed)
```

#### JWT Token Security

```python
# ❌ BAD: Weak secret, no expiration
jwt.encode({"user_id": 123}, "secret", algorithm="HS256")

# ✅ GOOD: Strong secret, expiration, proper algorithm
from datetime import datetime, timedelta
import secrets

JWT_SECRET = secrets.token_urlsafe(64)  # Generate strong secret
JWT_ALGORITHM = "HS256"
ACCESS_TOKEN_EXPIRE_MINUTES = 15

def create_access_token(data: dict):
    to_encode = data.copy()
    expire = datetime.utcnow() + timedelta(minutes=ACCESS_TOKEN_EXPIRE_MINUTES)
    to_encode.update({"exp": expire, "iat": datetime.utcnow()})
    return jwt.encode(to_encode, JWT_SECRET, algorithm=JWT_ALGORITHM)
```

#### Rate Limiting

```python
from slowapi import Limiter
from slowapi.util import get_remote_address

limiter = Limiter(key_func=get_remote_address)

# ✅ GOOD: Rate limit authentication endpoints
@app.post("/auth/login")
@limiter.limit("5/minute")  # Max 5 attempts per minute
async def login(request: Request, credentials: LoginRequest):
    user = await authenticate(credentials)
    if not user:
        raise HTTPException(status_code=401, detail="Invalid credentials")
    return {"token": create_access_token({"sub": user.id})}
```

**Checklist**:
- [ ] Passwords hashed with bcrypt/argon2
- [ ] JWT tokens have expiration (`exp` claim)
- [ ] Strong secrets (64+ random bytes)
- [ ] Rate limiting on authentication endpoints
- [ ] Session management secure (HTTPOnly cookies)
- [ ] Multi-factor authentication for sensitive operations

### 3. Sensitive Data Exposure

**Risk**: Exposing sensitive data to unauthorized parties.

#### Secrets Management

```python
# ❌ BAD: Secrets in code
DATABASE_URL = "mongodb://admin:password123@localhost"
API_KEY = "sk_live_abc123"

# ✅ GOOD: Environment variables
import os
from pydantic_settings import BaseSettings

class Settings(BaseSettings):
    database_url: str
    api_key: str

    class Config:
        env_file = ".env"

settings = Settings()
```

#### HTTPS Enforcement

```python
from fastapi.middleware.httpsredirect import HTTPSRedirectMiddleware

# ✅ GOOD: Force HTTPS in production
if not settings.debug:
    app.add_middleware(HTTPSRedirectMiddleware)
```

#### Data Encryption at Rest

```python
from cryptography.fernet import Fernet

# ✅ GOOD: Encrypt sensitive fields
def encrypt_field(value: str, key: bytes) -> str:
    f = Fernet(key)
    return f.encrypt(value.encode()).decode()

def decrypt_field(encrypted: str, key: bytes) -> str:
    f = Fernet(key)
    return f.decrypt(encrypted.encode()).decode()

# Encrypt SSN before storing
user.ssn_encrypted = encrypt_field(ssn, ENCRYPTION_KEY)
```

#### Logging Security

```python
# ❌ BAD: Logging sensitive data
logger.info(f"User login: {email}, password: {password}")

# ✅ GOOD: Redact sensitive information
logger.info(f"User login: {email}")  # No password

# ✅ Mask sensitive data in logs
def mask_email(email: str) -> str:
    local, domain = email.split('@')
    return f"{local[0]}***@{domain}"

logger.info(f"Password reset for: {mask_email(email)}")
```

**Checklist**:
- [ ] No secrets/credentials in code
- [ ] HTTPS enforced (TLS 1.2+)
- [ ] Sensitive data encrypted at rest
- [ ] No sensitive data in logs (passwords, tokens, PII)
- [ ] Proper key management (AWS KMS, HashiCorp Vault)

### 4. XSS (Cross-Site Scripting)

**Risk**: Attackers inject malicious scripts that execute in users' browsers.

#### Input Sanitization (Backend)

```python
import bleach

# ❌ BAD: Trust user HTML
def save_post(content: str):
    db.posts.insert({"content": content})  # Stores <script>alert(1)</script>

# ✅ GOOD: Sanitize HTML input
ALLOWED_TAGS = ['p', 'br', 'strong', 'em', 'a']
ALLOWED_ATTRS = {'a': ['href', 'title']}

def save_post(content: str):
    clean_content = bleach.clean(
        content,
        tags=ALLOWED_TAGS,
        attributes=ALLOWED_ATTRS,
        strip=True
    )
    db.posts.insert({"content": clean_content})
```

#### Output Escaping (Frontend)

```vue
<!-- ❌ BAD: Unescaped HTML (Vue) -->
<div v-html="userContent"></div>

<!-- ✅ GOOD: Escaped by default -->
<div>{{ userContent }}</div>

<!-- If HTML needed, sanitize first -->
<div v-html="sanitizedContent"></div>
```

```typescript
// ✅ React escapes by default
return <div>{userContent}</div>

// ❌ Dangerous: Only use with sanitized content
return <div dangerouslySetInnerHTML={{__html: userContent}} />
```

#### Content Security Policy

```python
from fastapi.middleware.cors import CORSMiddleware

# ✅ GOOD: Set CSP headers
@app.middleware("http")
async def add_security_headers(request, call_next):
    response = await call_next(request)
    response.headers["Content-Security-Policy"] = (
        "default-src 'self'; "
        "script-src 'self'; "
        "style-src 'self' 'unsafe-inline'; "
        "img-src 'self' data: https:;"
    )
    return response
```

**Checklist**:
- [ ] User input sanitized (use libraries like bleach)
- [ ] Output escaped in templates (default in modern frameworks)
- [ ] Content Security Policy configured
- [ ] Avoid `v-html`, `dangerouslySetInnerHTML` unless necessary

### 5. Broken Access Control

**Risk**: Users access resources they shouldn't have permission to.

#### Authorization Checks

```python
# ❌ BAD: No authorization check
@app.get("/posts/{post_id}")
async def get_post(post_id: int):
    post = await db.posts.find_one({"_id": post_id})
    return post  # Any user can read any post!

# ✅ GOOD: Verify ownership
@app.get("/posts/{post_id}")
async def get_post(
    post_id: int,
    current_user: User = Depends(get_current_user)
):
    post = await db.posts.find_one({"_id": post_id})
    if not post:
        raise HTTPException(status_code=404, detail="Post not found")

    # Check authorization
    if post.user_id != current_user.id and not current_user.is_admin:
        raise HTTPException(status_code=403, detail="Access denied")

    return post
```

#### Insecure Direct Object References (IDOR)

```python
# ❌ BAD: Predictable IDs, no ownership check
@app.delete("/users/{user_id}")
async def delete_user(user_id: int):
    await db.users.delete_one({"_id": user_id})

# ✅ GOOD: Verify current user or admin
@app.delete("/users/{user_id}")
async def delete_user(
    user_id: int,
    current_user: User = Depends(get_current_user)
):
    # Only allow deleting own account or admin
    if user_id != current_user.id and not current_user.is_admin:
        raise HTTPException(status_code=403, detail="Access denied")

    await db.users.delete_one({"_id": user_id})
```

#### Role-Based Access Control

```python
from enum import Enum

class UserRole(str, Enum):
    USER = "user"
    ADMIN = "admin"
    MODERATOR = "moderator"

def require_role(required_role: UserRole):
    def role_checker(current_user: User = Depends(get_current_user)):
        if current_user.role != required_role:
            raise HTTPException(status_code=403, detail="Insufficient permissions")
        return current_user
    return role_checker

# ✅ GOOD: Role-based endpoint protection
@app.delete("/posts/{post_id}")
async def delete_post(
    post_id: int,
    admin: User = Depends(require_role(UserRole.ADMIN))
):
    await db.posts.delete_one({"_id": post_id})
```

**Checklist**:
- [ ] Authorization checks on all endpoints
- [ ] Ownership verification for user resources
- [ ] Role-based access control implemented
- [ ] No direct object references without validation
- [ ] Principle of least privilege applied

## Security Headers

```python
@app.middleware("http")
async def add_security_headers(request, call_next):
    response = await call_next(request)

    # Prevent clickjacking
    response.headers["X-Frame-Options"] = "DENY"

    # Prevent MIME type sniffing
    response.headers["X-Content-Type-Options"] = "nosniff"

    # Enable XSS protection
    response.headers["X-XSS-Protection"] = "1; mode=block"

    # Content Security Policy
    response.headers["Content-Security-Policy"] = "default-src 'self'"

    # HSTS (force HTTPS)
    response.headers["Strict-Transport-Security"] = "max-age=31536000; includeSubDomains"

    return response
```

## Security Review Checklist

### Injection (A03:2021)
- [ ] All SQL queries use parameterized statements
- [ ] NoSQL queries validate input types with Pydantic
- [ ] No use of `eval()`, `exec()` with user input
- [ ] Shell commands use subprocess with array arguments

### Broken Authentication (A07:2021)
- [ ] Passwords hashed with bcrypt/argon2 (min cost 12)
- [ ] JWT tokens include expiration (`exp`)
- [ ] Secrets are 256+ bits, stored in environment
- [ ] Rate limiting on auth endpoints (5 attempts/min)
- [ ] Failed login doesn't reveal if user exists

### Sensitive Data Exposure (A02:2021)
- [ ] No secrets in code or version control
- [ ] HTTPS enforced in production
- [ ] Sensitive data encrypted at rest
- [ ] No PII/passwords in logs
- [ ] Secure cookie attributes (HTTPOnly, Secure, SameSite)

### XSS (A03:2021)
- [ ] User input sanitized (backend) with bleach
- [ ] Output escaped in templates (framework default)
- [ ] Content Security Policy configured
- [ ] Minimal use of `v-html` / `dangerouslySetInnerHTML`

### Broken Access Control (A01:2021)
- [ ] Authorization checks on all protected endpoints
- [ ] Ownership verified for user resources
- [ ] Role-based access control implemented
- [ ] IDOR vulnerabilities addressed

### Security Headers
- [ ] X-Frame-Options: DENY
- [ ] X-Content-Type-Options: nosniff
- [ ] Content-Security-Policy configured
- [ ] Strict-Transport-Security enabled

## Severity Levels

When reporting security issues:

- **Critical**: Immediate security risk, production access possible
  - Example: SQL injection on public endpoint

- **High**: Significant vulnerability, requires privilege
  - Example: IDOR allowing access to other users' data

- **Medium**: Moderate risk, limited impact
  - Example: Missing rate limiting on non-critical endpoint

- **Low**: Minor issue, minimal exploitability
  - Example: Verbose error messages

## Tools

- **Static Analysis**: Bandit (Python), ESLint security plugins
- **Dependency Scanning**: pip-audit, npm audit, Snyk
- **Dynamic Testing**: OWASP ZAP, Burp Suite
- **Secret Scanning**: Gitleaks, TruffleHog