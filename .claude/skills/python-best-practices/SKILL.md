---
name: python-best-practices
description: Python coding standards following PEP 8, type hints, and async/await patterns for FastAPI and modern Python development. Use when writing Python code, reviewing Python implementations, setting up FastAPI endpoints, or ensuring code quality and maintainability. Apply for consistent formatting, proper type annotations, async operation optimization, error handling patterns, and Pydantic model definitions. Essential for professional, maintainable Python codebases.
---

# Python Best Practices

Write clean, maintainable, performant Python code following modern standards and conventions.

## PEP 8 Style Guide

### Naming Conventions

```python
# ✅ GOOD: Proper naming
class UserService:  # PascalCase for classes
    pass

def get_user_by_id(user_id: int):  # snake_case for functions
    pass

MAX_RETRIES = 3  # UPPER_CASE for constants
user_count = 0   # snake_case for variables

# ❌ BAD: Inconsistent naming
class user_service:  # Should be PascalCase
    pass

def getUserById(userId):  # Should be snake_case
    pass
```

### Formatting

```python
# Indentation: 4 spaces (not tabs)
# Max line length: 88 characters (Black formatter standard)

# ✅ GOOD: Proper formatting
def calculate_total(
    items: list[dict],
    tax_rate: float = 0.1,
    discount: float = 0
) -> float:
    """Calculate total with tax and discount."""
    subtotal = sum(item["price"] for item in items)
    return subtotal * (1 + tax_rate) * (1 - discount)

# Use trailing commas in multi-line structures
ALLOWED_STATUSES = [
    "pending",
    "active",
    "completed",  # Trailing comma for easier diffs
]
```

### Imports

```python
# Order: stdlib, third-party, local
# Group by type, sort alphabetically

# ✅ GOOD: Organized imports
import asyncio
import json
from datetime import datetime
from typing import Optional

from fastapi import FastAPI, HTTPException
from pydantic import BaseModel

from app.models import User
from app.services import UserService

# ❌ BAD: Unorganized imports
from app.models import User
import json
from fastapi import FastAPI
import asyncio
```

## Type Hints

### Function Annotations

```python
from typing import Optional, Union

# ✅ GOOD: Comprehensive type hints
def get_user(user_id: int) -> Optional[User]:
    """Fetch user by ID, return None if not found."""
    return db.get(user_id)

async def create_user(data: UserCreate) -> User:
    """Create new user from input data."""
    user = User(**data.dict())
    await db.insert(user)
    return user

# Modern Python 3.10+ union syntax
def find_user(identifier: int | str) -> User | None:
    """Find user by ID (int) or email (str)."""
    if isinstance(identifier, int):
        return db.get_by_id(identifier)
    return db.get_by_email(identifier)

# ❌ BAD: No type hints
def get_user(user_id):
    return db.get(user_id)
```

### Collection Type Hints

```python
# Python 3.9+: Use built-in types
from typing import Optional

# ✅ GOOD: Modern type hints
def process_users(users: list[User]) -> dict[str, int]:
    return {user.email: user.id for user in users}

def get_config() -> dict[str, str | int | bool]:
    return {"host": "localhost", "port": 8000, "debug": True}

# ❌ BAD: Old style (pre-3.9)
from typing import List, Dict

def process_users(users: List[User]) -> Dict[str, int]:
    return {user.email: user.id for user in users}
```

### Pydantic Models

```python
from pydantic import BaseModel, EmailStr, Field, validator

class UserBase(BaseModel):
    """Base user model with common fields."""
    email: EmailStr
    name: str = Field(..., min_length=1, max_length=100)
    age: int | None = Field(None, ge=0, le=150)

class UserCreate(UserBase):
    """User creation request with password."""
    password: str = Field(..., min_length=8)

    @validator('password')
    def password_strength(cls, v):
        if not any(c.isupper() for c in v):
            raise ValueError('Password must contain uppercase letter')
        if not any(c.isdigit() for c in v):
            raise ValueError('Password must contain digit')
        return v

class User(UserBase):
    """User model with database fields."""
    id: int
    created_at: datetime
    is_active: bool = True

    class Config:
        orm_mode = True  # Enable ORM compatibility
```

## Async/Await Patterns

### Parallel Execution

```python
import asyncio

# ❌ BAD: Sequential execution (slow)
async def get_dashboard():
    user = await fetch_user()          # 50ms
    posts = await fetch_posts()        # 100ms
    comments = await fetch_comments()  # 80ms
    return {"user": user, "posts": posts, "comments": comments}
# Total: 230ms

# ✅ GOOD: Parallel execution (fast)
async def get_dashboard():
    user, posts, comments = await asyncio.gather(
        fetch_user(),
        fetch_posts(),
        fetch_comments()
    )
    return {"user": user, "posts": posts, "comments": comments}
# Total: 100ms (slowest operation)

# ✅ Handle errors gracefully
async def get_dashboard_safe():
    results = await asyncio.gather(
        fetch_user(),
        fetch_posts(),
        fetch_comments(),
        return_exceptions=True  # Don't fail on first exception
    )

    user, posts, comments = results
    if isinstance(posts, Exception):
        posts = []  # Fallback value
    if isinstance(comments, Exception):
        comments = []

    return {"user": user, "posts": posts, "comments": comments}
```

### Async Comprehensions

```python
# ✅ GOOD: Async comprehensions
async def get_all_users() -> list[dict]:
    users = await db.fetch_all("SELECT * FROM users")
    # Fetch posts for each user in parallel
    return [
        {
            **user,
            "posts": await fetch_user_posts(user["id"])
        }
        async for user in users  # Async comprehension
    ]

# For parallel processing of multiple items
async def enrich_users(user_ids: list[int]) -> list[dict]:
    tasks = [fetch_and_enrich(uid) for uid in user_ids]
    return await asyncio.gather(*tasks)
```

### Context Managers

```python
from contextlib import asynccontextmanager

# ✅ GOOD: Async context manager
@asynccontextmanager
async def get_db_connection():
    conn = await db.connect()
    try:
        yield conn
    finally:
        await conn.close()

# Usage
async def get_user(user_id: int):
    async with get_db_connection() as conn:
        return await conn.fetch_one("SELECT * FROM users WHERE id = ?", user_id)
```

## Error Handling

### FastAPI Exception Handling

```python
from fastapi import HTTPException, status

# ✅ GOOD: Specific exceptions with details
async def get_user(user_id: int) -> User:
    user = await db.users.find_one({"_id": user_id})

    if not user:
        raise HTTPException(
            status_code=status.HTTP_404_NOT_FOUND,
            detail=f"User {user_id} not found"
        )

    if not user.is_active:
        raise HTTPException(
            status_code=status.HTTP_403_FORBIDDEN,
            detail="User account is inactive"
        )

    return user

# ✅ Custom exception handling
class UserNotFoundError(Exception):
    pass

@app.exception_handler(UserNotFoundError)
async def user_not_found_handler(request, exc):
    return JSONResponse(
        status_code=404,
        content={"error": "User not found", "detail": str(exc)}
    )
```

### Logging

```python
import logging

logger = logging.getLogger(__name__)

# ✅ GOOD: Proper logging levels
async def process_payment(amount: float):
    logger.info(f"Processing payment: ${amount}")

    try:
        result = await payment_api.charge(amount)
        logger.info(f"Payment successful: {result.transaction_id}")
        return result
    except PaymentError as e:
        logger.error(f"Payment failed: {e}", exc_info=True)
        raise HTTPException(status_code=402, detail="Payment failed")
    except Exception as e:
        logger.critical(f"Unexpected error in payment: {e}", exc_info=True)
        raise
```

## FastAPI Best Practices

### Dependency Injection

```python
from fastapi import Depends

# ✅ GOOD: Dependency injection
async def get_db():
    db = Database()
    try:
        yield db
    finally:
        await db.close()

async def get_current_user(
    token: str = Depends(oauth2_scheme),
    db = Depends(get_db)
) -> User:
    user = await db.get_user_by_token(token)
    if not user:
        raise HTTPException(status_code=401, detail="Invalid token")
    return user

@app.get("/profile")
async def get_profile(current_user: User = Depends(get_current_user)):
    return current_user
```

### Request/Response Models

```python
# ✅ GOOD: Separate input/output models
class UserCreate(BaseModel):
    """Input: User registration"""
    email: EmailStr
    password: str
    name: str

class UserResponse(BaseModel):
    """Output: User data (no password)"""
    id: int
    email: EmailStr
    name: str
    created_at: datetime

@app.post("/users", response_model=UserResponse, status_code=201)
async def create_user(user: UserCreate):
    # Password hashing logic
    hashed_password = hash_password(user.password)

    # Create user
    new_user = await db.create_user(
        email=user.email,
        password=hashed_password,
        name=user.name
    )

    return new_user  # Pydantic automatically excludes password
```

### Background Tasks

```python
from fastapi import BackgroundTasks

# ✅ GOOD: Use background tasks for non-blocking operations
def send_welcome_email(email: str):
    # Send email (slow operation)
    email_service.send(email, "Welcome!")

@app.post("/register")
async def register_user(
    user: UserCreate,
    background_tasks: BackgroundTasks
):
    # Create user immediately
    new_user = await db.create_user(user)

    # Send email in background (don't wait)
    background_tasks.add_task(send_welcome_email, new_user.email)

    return new_user
```

## Code Quality Tools

### Black (Formatter)

```bash
# Install
pip install black

# Format code
black app/

# Check without modifying
black --check app/

# pyproject.toml configuration
[tool.black]
line-length = 88
target-version = ['py311']
```

### Ruff (Linter)

```bash
# Install
pip install ruff

# Lint code
ruff check app/

# Auto-fix issues
ruff check --fix app/

# pyproject.toml configuration
[tool.ruff]
select = ["E", "F", "I", "N", "UP"]
ignore = ["E501"]  # Line too long (handled by Black)
line-length = 88
```

### MyPy (Type Checker)

```bash
# Install
pip install mypy

# Check types
mypy app/

# pyproject.toml configuration
[tool.mypy]
python_version = "3.11"
warn_return_any = true
warn_unused_configs = true
disallow_untyped_defs = true
```

## Best Practices Checklist

- [ ] Follow PEP 8 naming conventions (snake_case, PascalCase)
- [ ] Use type hints for all function signatures
- [ ] Prefer `async/await` over sync code in FastAPI
- [ ] Use `asyncio.gather()` for parallel async operations
- [ ] Implement proper error handling with specific exceptions
- [ ] Use Pydantic models for request/response validation
- [ ] Leverage FastAPI dependency injection
- [ ] Configure logging appropriately (INFO for normal, ERROR for issues)
- [ ] Run Black formatter for consistent code style
- [ ] Use Ruff for linting and catching common issues
- [ ] Run MyPy for type checking
- [ ] Keep functions focused and small (< 50 lines)
- [ ] Write docstrings for public APIs
- [ ] Use trailing commas in multi-line collections