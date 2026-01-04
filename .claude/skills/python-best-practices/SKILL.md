---
name: python-best-practices
description: Python coding standards (PEP 8, type hints, async/await).
allowed-tools: Read, Edit
---

# Python Best Practices

## PEP 8 Style Guide
- 4 spaces indentation
- Max line length: 88 (Black formatter)
- Use snake_case for functions/variables
- Use PascalCase for classes

## Type Hints
```python
def get_user(user_id: int) -> User | None:
    return await db.users.find_one({"_id": user_id})

async def create_user(data: UserCreate) -> User:
    user = User(**data.dict())
    await db.users.insert_one(user.dict())
    return user
```

## Async/Await
```python
# Good: Parallel execution
async def get_dashboard():
    user_task = fetch_user()
    posts_task = fetch_posts()
    user, posts = await asyncio.gather(user_task, posts_task)
    return {"user": user, "posts": posts}

# Bad: Sequential execution
async def get_dashboard():
    user = await fetch_user()
    posts = await fetch_posts()
    return {"user": user, "posts": posts}
```

## Error Handling
```python
try:
    result = await operation()
except SpecificError as e:
    logger.error(f"Error: {e}")
    raise HTTPException(status_code=400, detail="User-friendly message")
```
