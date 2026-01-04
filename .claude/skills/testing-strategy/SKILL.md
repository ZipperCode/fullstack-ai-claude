---
name: testing-strategy
description: Test strategy and best practices (unit, integration, E2E).
allowed-tools: Read, Write, Bash
---

# Testing Strategy

## Test Pyramid
```
    ┌─────────┐
    │ E2E (5%)│  ← Playwright
    ├─────────┤
    │ Integration (15%)│  ← API tests
    ├─────────┤
    │ Unit Tests (80%)│  ← Jest/pytest
    └─────────┘
```

## Unit Test (Backend)
```python
@pytest.mark.asyncio
async def test_create_user():
    user_data = {"email": "test@example.com", "password": "pass123"}
    result = await create_user(user_data)
    assert result.email == "test@example.com"
```

## Integration Test (API)
```python
async def test_login_endpoint(client):
    response = await client.post("/api/v1/auth/login", json={
        "email": "test@example.com",
        "password": "password123"
    })
    assert response.status_code == 200
    assert "token" in response.json()
```

## E2E Test (Playwright)
```typescript
test('login flow', async ({ page }) => {
  await page.goto('/login')
  await page.fill('input[type="email"]', 'test@example.com')
  await page.fill('input[type="password"]', 'password')
  await page.click('button[type="submit"]')
  await expect(page).toHaveURL('/dashboard')
})
```

## Coverage Target
- Overall: 85%+
- Critical paths: 95%+
