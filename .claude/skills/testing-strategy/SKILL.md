---
name: testing-strategy
description: Comprehensive testing strategy covering unit tests, integration tests, and E2E tests following the test pyramid model. Use when implementing new features requiring test coverage, setting up testing infrastructure, debugging failing tests, or improving code quality. Apply for pytest/Jest test implementation, API endpoint testing, UI component testing, and test automation. Essential for maintaining reliable, bug-free applications with confidence in deployments.
---

# Testing Strategy

Implement comprehensive testing coverage following the test pyramid model for reliable, maintainable applications.

## Test Pyramid

```
    ┌─────────────┐
    │   E2E (5%) │  ← Browser automation (Playwright)
    ├─────────────┤
    │ Integration │  ← API tests, DB tests
    │    (15%)    │
    ├─────────────┤
    │  Unit Tests │  ← Function/component tests
    │    (80%)    │     (Jest, pytest)
    └─────────────┘
```

**Philosophy**:
- **Many unit tests**: Fast, isolated, test business logic
- **Some integration tests**: Test component interactions
- **Few E2E tests**: Test critical user flows only

## Unit Tests (Backend - Python)

### Basic Test Structure

```python
# tests/test_user_service.py
import pytest
from app.services.user_service import UserService
from app.models import User

@pytest.mark.asyncio
async def test_create_user():
    """Test user creation with valid data."""
    service = UserService()
    user_data = {
        "email": "test@example.com",
        "password": "secure123",
        "name": "Test User"
    }

    result = await service.create_user(user_data)

    assert result.email == "test@example.com"
    assert result.name == "Test User"
    assert result.id is not None

@pytest.mark.asyncio
async def test_create_user_duplicate_email():
    """Test user creation fails with duplicate email."""
    service = UserService()
    user_data = {
        "email": "existing@example.com",
        "password": "secure123",
        "name": "Test User"
    }

    # Create first user
    await service.create_user(user_data)

    # Attempt duplicate should raise error
    with pytest.raises(ValueError, match="Email already exists"):
        await service.create_user(user_data)
```

### Fixtures for Setup/Teardown

```python
# conftest.py
import pytest
from motor.motor_asyncio import AsyncIOMotorClient

@pytest.fixture
async def db():
    """Provide clean test database."""
    client = AsyncIOMotorClient("mongodb://localhost:27017")
    db = client.test_database

    yield db

    # Cleanup after test
    await db.users.delete_many({})
    client.close()

@pytest.fixture
async def sample_user(db):
    """Create a sample user for testing."""
    user = {
        "email": "sample@example.com",
        "name": "Sample User",
        "created_at": datetime.utcnow()
    }
    result = await db.users.insert_one(user)
    user["_id"] = result.inserted_id
    return user

# Usage in tests
@pytest.mark.asyncio
async def test_get_user(db, sample_user):
    """Test fetching user by ID."""
    service = UserService(db)
    user = await service.get_user(sample_user["_id"])
    assert user.email == "sample@example.com"
```

### Mocking External Dependencies

```python
from unittest.mock import AsyncMock, patch

@pytest.mark.asyncio
@patch('app.services.email_service.send_email')
async def test_user_registration_sends_email(mock_send_email):
    """Test that user registration triggers welcome email."""
    mock_send_email.return_value = AsyncMock(return_value=True)

    service = UserService()
    await service.register_user({
        "email": "new@example.com",
        "password": "secure123"
    })

    # Verify email was sent
    mock_send_email.assert_called_once_with(
        to="new@example.com",
        subject="Welcome!",
        template="welcome"
    )
```

## Integration Tests (API)

### Testing FastAPI Endpoints

```python
# tests/test_api_users.py
import pytest
from httpx import AsyncClient
from app.main import app

@pytest.fixture
async def client():
    """Provide async HTTP client."""
    async with AsyncClient(app=app, base_url="http://test") as ac:
        yield ac

@pytest.mark.asyncio
async def test_create_user_endpoint(client):
    """Test POST /api/v1/users endpoint."""
    response = await client.post("/api/v1/users", json={
        "email": "newuser@example.com",
        "password": "securePass123",
        "name": "New User"
    })

    assert response.status_code == 201
    data = response.json()
    assert data["email"] == "newuser@example.com"
    assert "id" in data
    assert "password" not in data  # Password should not be returned

@pytest.mark.asyncio
async def test_login_endpoint(client):
    """Test POST /api/v1/auth/login endpoint."""
    # First create user
    await client.post("/api/v1/users", json={
        "email": "user@example.com",
        "password": "password123",
        "name": "Test User"
    })

    # Then login
    response = await client.post("/api/v1/auth/login", json={
        "email": "user@example.com",
        "password": "password123"
    })

    assert response.status_code == 200
    data = response.json()
    assert "access_token" in data
    assert data["token_type"] == "bearer"

@pytest.mark.asyncio
async def test_get_user_unauthorized(client):
    """Test GET /api/v1/users/me without authentication."""
    response = await client.get("/api/v1/users/me")

    assert response.status_code == 401
    assert "detail" in response.json()
```

### Testing with Authentication

```python
@pytest.fixture
async def authenticated_client(client):
    """Provide client with authentication token."""
    # Register user
    await client.post("/api/v1/users", json={
        "email": "auth@example.com",
        "password": "password123",
        "name": "Auth User"
    })

    # Login to get token
    response = await client.post("/api/v1/auth/login", json={
        "email": "auth@example.com",
        "password": "password123"
    })
    token = response.json()["access_token"]

    # Add token to client headers
    client.headers["Authorization"] = f"Bearer {token}"
    return client

@pytest.mark.asyncio
async def test_get_current_user(authenticated_client):
    """Test fetching current authenticated user."""
    response = await authenticated_client.get("/api/v1/users/me")

    assert response.status_code == 200
    data = response.json()
    assert data["email"] == "auth@example.com"
```

## Unit Tests (Frontend - Vue/React)

### Vue Component Test (Vitest)

```typescript
// UserProfile.spec.ts
import { describe, it, expect } from 'vitest'
import { mount } from '@vue/test-utils'
import UserProfile from './UserProfile.vue'

describe('UserProfile', () => {
  it('renders user information', () => {
    const wrapper = mount(UserProfile, {
      props: {
        user: {
          name: 'John Doe',
          email: 'john@example.com',
          avatar: 'https://example.com/avatar.jpg'
        }
      }
    })

    expect(wrapper.text()).toContain('John Doe')
    expect(wrapper.text()).toContain('john@example.com')
  })

  it('emits update event when edit button clicked', async () => {
    const wrapper = mount(UserProfile, {
      props: {
        user: { name: 'John', email: 'john@example.com' },
        editable: true
      }
    })

    await wrapper.find('[data-test="edit-button"]').trigger('click')

    expect(wrapper.emitted()).toHaveProperty('edit')
    expect(wrapper.emitted('edit')).toHaveLength(1)
  })

  it('disables edit button when not editable', () => {
    const wrapper = mount(UserProfile, {
      props: {
        user: { name: 'John', email: 'john@example.com' },
        editable: false
      }
    })

    const editButton = wrapper.find('[data-test="edit-button"]')
    expect(editButton.exists()).toBe(false)
  })
})
```

### React Component Test (Jest + React Testing Library)

```typescript
// UserProfile.test.tsx
import { render, screen, fireEvent } from '@testing-library/react'
import UserProfile from './UserProfile'

describe('UserProfile', () => {
  it('renders user information', () => {
    render(
      <UserProfile
        user={{
          name: 'John Doe',
          email: 'john@example.com',
          avatar: 'https://example.com/avatar.jpg'
        }}
      />
    )

    expect(screen.getByText('John Doe')).toBeInTheDocument()
    expect(screen.getByText('john@example.com')).toBeInTheDocument()
  })

  it('calls onUpdate when edit button clicked', () => {
    const onUpdate = jest.fn()

    render(
      <UserProfile
        user={{ name: 'John', email: 'john@example.com' }}
        editable={true}
        onUpdate={onUpdate}
      />
    )

    fireEvent.click(screen.getByRole('button', { name: /edit/i }))

    expect(onUpdate).toHaveBeenCalledTimes(1)
  })
})
```

### Testing Composables/Hooks

```typescript
// useUser.test.ts
import { renderHook, waitFor } from '@testing-library/react'
import { useUser } from './useUser'

describe('useUser', () => {
  it('fetches user data', async () => {
    const { result } = renderHook(() => useUser(123))

    expect(result.current.loading).toBe(true)

    await waitFor(() => {
      expect(result.current.loading).toBe(false)
    })

    expect(result.current.user).toEqual({
      id: 123,
      name: 'John Doe'
    })
  })

  it('handles fetch error', async () => {
    const { result } = renderHook(() => useUser(999))

    await waitFor(() => {
      expect(result.current.error).toBeTruthy()
    })

    expect(result.current.user).toBeNull()
  })
})
```

## E2E Tests (Playwright)

### Login Flow Test

```typescript
// tests/e2e/auth.spec.ts
import { test, expect } from '@playwright/test'

test('user can login successfully', async ({ page }) => {
  // Navigate to login page
  await page.goto('/login')

  // Fill login form
  await page.fill('input[type="email"]', 'test@example.com')
  await page.fill('input[type="password"]', 'password123')

  // Submit form
  await page.click('button[type="submit"]')

  // Wait for navigation to dashboard
  await page.waitForURL('/dashboard')

  // Verify user is logged in
  await expect(page.locator('text=Welcome back')).toBeVisible()
})

test('login fails with invalid credentials', async ({ page }) => {
  await page.goto('/login')

  await page.fill('input[type="email"]', 'wrong@example.com')
  await page.fill('input[type="password"]', 'wrongpassword')

  await page.click('button[type="submit"]')

  // Should show error message
  await expect(page.locator('text=Invalid credentials')).toBeVisible()

  // Should stay on login page
  await expect(page).toHaveURL('/login')
})
```

### User Registration Flow

```typescript
test('user can register and access dashboard', async ({ page }) => {
  await page.goto('/register')

  // Fill registration form
  await page.fill('input[name="name"]', 'New User')
  await page.fill('input[name="email"]', 'newuser@example.com')
  await page.fill('input[name="password"]', 'SecurePass123')
  await page.fill('input[name="confirmPassword"]', 'SecurePass123')

  await page.click('button[type="submit"]')

  // Should redirect to dashboard after registration
  await page.waitForURL('/dashboard')
  await expect(page.locator('text=Welcome, New User')).toBeVisible()
})
```

### API Mocking in E2E

```typescript
test('handles slow API gracefully', async ({ page }) => {
  // Mock API with delay
  await page.route('**/api/v1/users/me', async route => {
    await new Promise(resolve => setTimeout(resolve, 2000))
    await route.fulfill({
      status: 200,
      body: JSON.stringify({
        id: 1,
        name: 'Test User',
        email: 'test@example.com'
      })
    })
  })

  await page.goto('/profile')

  // Should show loading state
  await expect(page.locator('text=Loading...')).toBeVisible()

  // Should eventually show data
  await expect(page.locator('text=Test User')).toBeVisible({ timeout: 5000 })
})
```

## Coverage Targets

```bash
# Backend (pytest-cov)
pytest --cov=app --cov-report=html --cov-report=term

# Frontend (Vitest)
vitest run --coverage

# E2E (Playwright)
npx playwright test
```

**Target Coverage**:
- **Overall**: 85%+
- **Critical paths** (auth, payment, data loss): 95%+
- **E2E**: Cover critical user journeys

## Testing Best Practices

### 1. Arrange-Act-Assert Pattern

```python
def test_user_creation():
    # Arrange: Set up test data
    user_data = {"email": "test@example.com", "name": "Test"}

    # Act: Perform the action
    result = create_user(user_data)

    # Assert: Verify the outcome
    assert result.email == "test@example.com"
```

### 2. Test One Thing Per Test

```python
# ❌ BAD: Testing multiple things
def test_user_operations():
    user = create_user({})
    assert user.email is not None
    updated = update_user(user.id, {"name": "New"})
    assert updated.name == "New"
    deleted = delete_user(user.id)
    assert deleted is True

# ✅ GOOD: Separate tests
def test_create_user():
    user = create_user({})
    assert user.email is not None

def test_update_user():
    user = create_user({})
    updated = update_user(user.id, {"name": "New"})
    assert updated.name == "New"
```

### 3. Use Descriptive Test Names

```python
# ❌ BAD: Vague name
def test_user():
    pass

# ✅ GOOD: Descriptive name
def test_create_user_with_valid_email_succeeds():
    pass

def test_create_user_with_duplicate_email_raises_error():
    pass
```

### 4. Test Error Cases

```python
def test_get_user_returns_none_when_not_found():
    result = get_user(user_id=99999)
    assert result is None

def test_create_user_raises_error_for_invalid_email():
    with pytest.raises(ValidationError):
        create_user({"email": "invalid-email"})
```

## Testing Checklist

- [ ] Unit tests cover business logic functions
- [ ] Integration tests cover API endpoints
- [ ] E2E tests cover critical user flows
- [ ] Tests use descriptive names
- [ ] Each test focuses on one behavior
- [ ] Error cases tested (invalid input, not found, unauthorized)
- [ ] Fixtures used for test setup/teardown
- [ ] External dependencies mocked
- [ ] Overall coverage > 85%
- [ ] Critical path coverage > 95%
- [ ] Tests run in CI/CD pipeline