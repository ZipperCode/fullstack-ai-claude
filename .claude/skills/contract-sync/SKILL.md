---
name: contract-sync
description: Export OpenAPI spec from FastAPI and validate frontend/mobile DTO consistency to prevent integration bugs. Use PROACTIVELY after API changes (modifying FastAPI routes, Pydantic schemas, or response models) and before PR merge. Also use when detecting type mismatches between backend responses and frontend TypeScript interfaces, or when mobile team reports data structure issues. Critical for maintaining single source of truth and catching breaking changes early.
---

# API Contract Sync & Validation

Ensure API contracts are **enforced at code level**, preventing integration bugs between backend, frontend, and mobile.

## The Problem This Solves

- ❌ Backend changes API, frontend breaks silently
- ❌ TypeScript types drift from actual API responses
- ❌ Mobile team uses outdated data models
- ❌ Integration bugs caught only at runtime

## The Solution

- ✅ Single source of truth: OpenAPI Spec
- ✅ Automated validation before commits
- ✅ Early detection of breaking changes
- ✅ Type safety across the stack

## Core Workflow

### 1. Export OpenAPI Spec from FastAPI

**When**: After modifying any FastAPI route, endpoint, or Pydantic schema.

**How**:

```bash
# Method 1: Direct Python export (Recommended)
cd packages/backend
python -c "
import sys
sys.path.insert(0, 'src')
from main import app
import json

spec = app.openapi()
with open('../../openapi.json', 'w') as f:
    json.dump(spec, f, indent=2, ensure_ascii=False)

print('✅ OpenAPI spec exported to openapi.json')
print(f'   - {len(spec[\"paths\"])} endpoints')
print(f'   - {len(spec.get(\"components\", {}).get(\"schemas\", {}))} schemas')
"

# Method 2: HTTP endpoint (if dev server running)
curl http://localhost:8000/openapi.json > ../../openapi.json
```

The exported `openapi.json` contains:
- All endpoint paths and HTTP methods
- Request/response schemas
- Data type definitions
- Validation rules

**Always commit** `openapi.json` to version control as the single source of truth.

### 2. Validate Frontend TypeScript Types

**When**: After exporting OpenAPI spec, before merging PR.

**Goal**: Ensure frontend TypeScript interfaces match backend schemas.

#### Recommended: Auto-Generate Types

```bash
cd packages/frontend

# Install tool (once)
npm install -D openapi-typescript

# Generate TypeScript types from OpenAPI
npx openapi-typescript ../../openapi.json -o src/types/generated/api.ts
```

**Usage in frontend**:
```typescript
import type { LoginRequest, LoginResponse } from '@/types/generated/api'

// Types are automatically synchronized with backend
const loginData: LoginRequest = {
  email: "user@example.com",
  password: "secure123"
}
```

#### Common Issues to Detect

1. **Missing Fields**: Backend added field, frontend type doesn't have it
2. **Type Mismatch**: Backend returns `number`, frontend expects `string`
3. **Required vs Optional**: Backend requires field, frontend marks it optional

**Validation output example**:
```
🔍 Validating Frontend Contracts...

❌ LoginRequest - MISMATCH
  - Missing field: rememberMe (boolean, optional)
  - Type mismatch: userId (expected: number, got: string)

✅ LoginResponse - OK
✅ UserListResponse - OK

Summary: 1 schema has issues, 2 are valid

🔧 Fix: npx openapi-typescript ../../openapi.json -o src/types/generated/api.ts
```

### 3. Validate Android Kotlin Data Classes

**When**: After exporting OpenAPI spec, before mobile release.

**Goal**: Ensure Android data classes match backend schemas.

#### Common Issues

1. **Field Naming**: Backend uses `snake_case`, Kotlin uses `camelCase`
   ```kotlin
   data class User(
       @SerializedName("user_id")  // ✅ Correct
       val userId: Int,

       @SerializedName("created_at")  // ✅ Required for snake_case
       val createdAt: String
   )
   ```

2. **Type Mapping**:
   - OpenAPI `integer` → Kotlin `Int`
   - OpenAPI `string` → Kotlin `String`
   - OpenAPI `boolean` → Kotlin `Boolean`
   - OpenAPI `number` → Kotlin `Double`
   - OpenAPI `array` → Kotlin `List<T>`

3. **Nullability**: Ensure Kotlin nullable types match OpenAPI nullable fields

#### Recommended: Auto-Generate Data Classes

See [references/tools-setup.md](references/tools-setup.md) for OpenAPI Generator setup.

## Breaking Change Detection

### What Constitutes a Breaking Change

⚠️ **Breaking** (requires coordination):
- Type change: `string` → `integer`
- New required field added
- Field removed
- More restrictive validation (shorter max length, stricter format)

✅ **Non-breaking** (safe to deploy):
- New optional field added
- Less restrictive validation
- Documentation updates
- Deprecated field removed (after grace period)

### Detecting Breaking Changes

```bash
# Install diff tool (once)
npm install -g @openapitools/openapi-diff

# Compare old vs new specs
openapi-diff openapi-old.json openapi-new.json --fail-on-breaking
```

**Output example**:
```
⚠️  Breaking Changes Detected:

1. POST /api/v1/users
   - New required field: username (string)
   - Impact: All clients must send username

2. GET /api/v1/users/{id}
   - Response field type changed: userId (string → integer)
   - Impact: Frontend/mobile must update type definitions
```

### Handling Breaking Changes

**Strategy**: API versioning with deprecation period

1. Create new endpoint version: `/api/v2/auth/login`
2. Keep old version: `/api/v1/auth/login` (deprecated)
3. Document migration in CHANGELOG
4. Remove old version after grace period (e.g., 3 months)

**Example**:
```
Breaking change detected in POST /api/v1/auth/login

Recommendation:
- Create: POST /api/v2/auth/login (new schema)
- Deprecate: POST /api/v1/auth/login (keep for 3 months)
- Migration guide:
  * Add "username" field (required)
  * Change "userId" from string to number
  * Update error response format
```

## Best Practices

### 1. Always Commit openapi.json

```bash
git add openapi.json
git commit -m "chore: update OpenAPI spec for auth API changes"
```

### 2. Merge PR Checklist

Before merging any PR with API changes:

- [ ] OpenAPI spec exported and committed
- [ ] No unexpected breaking changes detected
- [ ] Frontend types regenerated (if needed)
- [ ] Mobile data classes regenerated (if needed)
- [ ] Migration guide created (if breaking changes)
- [ ] Affected teams notified

### 3. Document Breaking Changes

```markdown
# CHANGELOG.md

## [2.0.0] - 2024-01-15

### Breaking Changes

- **POST /api/v1/auth/login**: New required field `username`
  - Migration: Add username field to login form
  - Old endpoint deprecated, will be removed 2024-04-15
```

### 4. Regular Contract Audits

```bash
# Weekly: Check for contract drift
npm run validate:contracts

# If drift detected, investigate and fix immediately
```

## Tool Setup

For detailed installation instructions for all tools (openapi-typescript, openapi-diff, Kotlin generator, CI/CD setup), see [references/tools-setup.md](references/tools-setup.md).

Quick setup:
```bash
# Frontend
cd packages/frontend
npm install -D openapi-typescript

# Diff tool (global)
npm install -g @openapitools/openapi-diff
```

## Summary

Every time you modify backend API:

1. Export OpenAPI spec
2. Check for breaking changes
3. Validate/regenerate frontend types
4. Validate/regenerate mobile data classes
5. Create migration guide (if breaking)
6. Notify affected teams
7. Commit openapi.json
8. Update CHANGELOG (if breaking)

**This skill is the foundation of reliable full-stack integration.**
