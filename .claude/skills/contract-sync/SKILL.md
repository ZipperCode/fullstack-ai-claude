---
name: contract-sync
description: Export OpenAPI spec from FastAPI and validate frontend/mobile DTO consistency. Use PROACTIVELY after API changes and before PR merge.
allowed-tools: Read, Bash, Grep, Glob
---

# API Contract Sync & Validation Skill

## Purpose

Ensure API contracts are **enforced at the code level**, not just documented.

### The Problem This Solves

❌ Backend changes API, frontend breaks silently
❌ TypeScript types drift from actual API responses
❌ Mobile team uses outdated data models
❌ Integration bugs caught only at runtime

✅ Single source of truth: OpenAPI Spec
✅ Automated validation before commits
✅ Early detection of breaking changes
✅ Type safety across the stack

---

## Workflow

### Phase 1: Export OpenAPI Spec from FastAPI

**When to run:**
- After modifying any FastAPI route/endpoint
- After changing Pydantic schemas
- Before creating a PR with backend changes
- In CI/CD pipeline

**How to export:**

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

# Method 2: Using FastAPI CLI (if available)
fastapi openapi --app src.main:app --out ../../openapi.json

# Method 3: HTTP endpoint (if dev server running)
curl http://localhost:8000/openapi.json > ../../openapi.json
```

**Expected Output:**

```json
{
  "openapi": "3.1.0",
  "info": { "title": "API", "version": "1.0.0" },
  "paths": {
    "/api/v1/auth/login": {
      "post": {
        "requestBody": {
          "content": {
            "application/json": {
              "schema": { "$ref": "#/components/schemas/LoginRequest" }
            }
          }
        },
        "responses": {
          "200": {
            "content": {
              "application/json": {
                "schema": { "$ref": "#/components/schemas/LoginResponse" }
              }
            }
          }
        }
      }
    }
  },
  "components": {
    "schemas": {
      "LoginRequest": {
        "type": "object",
        "properties": {
          "email": { "type": "string", "format": "email" },
          "password": { "type": "string", "minLength": 8 }
        },
        "required": ["email", "password"]
      }
    }
  }
}
```

---

### Phase 2: Validate Frontend TypeScript DTO

**Check consistency between:**
- `openapi.json` schemas
- `packages/frontend/src/types/` TypeScript interfaces

**Validation Methods:**

#### Option 1: Manual Comparison

```bash
# View OpenAPI schema
cat openapi.json | jq '.components.schemas.LoginRequest'

# View TypeScript interface
cat packages/frontend/src/types/auth.ts | grep -A 10 "interface LoginRequest"
```

#### Option 2: Automated Type Generation (Recommended)

```bash
cd packages/frontend

# Install tool (once)
npm install -D openapi-typescript

# Generate TypeScript types from OpenAPI
npx openapi-typescript ../../openapi.json -o src/types/generated/api.ts

# Now frontend can import:
# import type { LoginRequest, LoginResponse } from '@/types/generated/api'
```

#### Option 3: Schema Diff

```bash
# Install diff tool
npm install -g @openapitools/openapi-diff

# Compare old vs new OpenAPI specs (detect breaking changes)
openapi-diff openapi-old.json openapi.json --fail-on-breaking
```

**Common Issues to Detect:**

1. **Missing Fields**
   ```typescript
   // OpenAPI Schema
   {
     "email": { "type": "string" },
     "password": { "type": "string" },
     "rememberMe": { "type": "boolean" }  // ← Backend added this
   }

   // Frontend Type (OUTDATED!)
   interface LoginRequest {
     email: string;
     password: string;
     // ❌ Missing: rememberMe
   }
   ```

2. **Type Mismatch**
   ```typescript
   // OpenAPI: "userId": { "type": "integer" }
   // Frontend: userId: string  // ❌ Should be number
   ```

3. **Required vs Optional Mismatch**
   ```typescript
   // OpenAPI: "required": ["email", "password"]
   // Frontend: email?: string  // ❌ Should be required (email: string)
   ```

**Expected Validation Output:**

```
🔍 Validating Frontend Contracts...

❌ LoginRequest - MISMATCH
  - Missing field: rememberMe (boolean, optional)
  - Type mismatch: userId (expected: number, got: string)

❌ UserProfile - MISMATCH
  - Extra field: deprecated_field (not in OpenAPI)
  - Required mismatch: bio (OpenAPI: optional, Frontend: required)

✅ LoginResponse - OK
✅ UserListResponse - OK

Summary: 2 schemas have issues, 2 are valid

🔧 Recommended fix:
npx openapi-typescript ../../openapi.json -o src/types/generated/api.ts
```

---

### Phase 3: Validate Android Kotlin Data Classes

**Check consistency between:**
- `openapi.json` schemas
- `packages/mobile/app/src/main/java/.../data/model/` Kotlin data classes

**Common Issues:**

1. **Field Naming Convention**
   ```kotlin
   // OpenAPI (snake_case)
   {
     "user_id": { "type": "integer" },
     "created_at": { "type": "string", "format": "date-time" }
   }

   // Kotlin (camelCase) - NEEDS SERIALIZATION ANNOTATION
   data class User(
       @SerializedName("user_id")  // ✅ Correct
       val userId: Int,

       val createdAt: String  // ❌ Missing @SerializedName("created_at")
   )
   ```

2. **Type Mapping**
   ```kotlin
   // OpenAPI Type → Kotlin Type
   // "integer" → Int
   // "string" → String
   // "boolean" → Boolean
   // "number" → Double/Float
   // "array" → List<T>
   // "object" → Custom data class
   ```

3. **Nullable Mismatch**
   ```kotlin
   // OpenAPI: "nullable": false
   // Kotlin: val email: String?  // ❌ Should be non-nullable
   ```

**Validation Commands:**

```bash
# Extract Kotlin data classes
grep -r "data class" packages/mobile/app/src/main/java/*/data/model/

# Check @SerializedName usage
grep -B 1 "@SerializedName" packages/mobile/app/src/main/java/*/data/model/

# View specific data class
cat packages/mobile/.../User.kt
```

**Automated Code Generation (Recommended):**

```kotlin
// In packages/mobile/build.gradle
plugins {
    id "org.openapi.generator" version "7.0.0"
}

openApiGenerate {
    inputSpec = "$rootDir/../../openapi.json"
    generatorName = "kotlin"
    outputDir = "$buildDir/generated"
    configOptions = [
        dateLibrary: "java8",
        serializationLibrary: "gson"
    ]
}
```

---

## Breaking Change Detection

### ⚠️ Breaking Changes (Require coordination)

```diff
- "email": { "type": "string" }
+ "email": { "type": "string", "format": "email", "minLength": 5 }
  ↑ More restrictive validation

- "userId": { "type": "string" }
+ "userId": { "type": "integer" }
  ↑ Type change

- "required": ["email", "password"]
+ "required": ["email", "password", "username"]
  ↑ New required field
```

**When breaking changes detected:**

```bash
# Use openapi-diff tool
openapi-diff openapi-v1.json openapi-v2.json

# Output example:
⚠️  Breaking Changes Detected:

1. POST /api/v1/users
   - New required field: username (string)
   - Impact: All clients must send username

2. GET /api/v1/users/{id}
   - Response field type changed: userId (string → integer)
   - Impact: Frontend/mobile must update type definitions
```

**Breaking Change Strategy C (Agreed):**

When breaking change detected:
1. Suggest API versioning (v1 → v2)
2. Generate migration guide
3. Keep old version deprecated for grace period

Example:
```
Breaking change detected in POST /api/v1/auth/login

Recommendation:
- Create new endpoint: POST /api/v2/auth/login (with new schema)
- Deprecate: POST /api/v1/auth/login (keep for 3 months)
- Migration guide:
  * Add "username" field (required)
  * Change "userId" from string to number
```

### ✅ Non-Breaking Changes

```diff
+ "bio": { "type": "string" }
  ↑ New optional field

- "deprecated_field": { "type": "string" }
  ↑ Removed deprecated field (after grace period)

- "description": "Old description"
+ "description": "New description"
  ↑ Documentation update
```

---

## Integration with Git Hooks

### Pre-commit Hook Example

```bash
# .husky/pre-commit
#!/bin/bash

echo "🔍 Validating API Contracts..."

# Check if backend files changed
if git diff --cached --name-only | grep -q "packages/backend/src/api\|packages/backend/src/schemas"; then
    echo "⚠️  Backend API files changed, validating contracts..."

    # Export latest OpenAPI spec
    cd packages/backend
    python -c "
    import sys
    sys.path.insert(0, 'src')
    from main import app
    import json
    with open('../../openapi.json', 'w') as f:
        json.dump(app.openapi(), f, indent=2)
    " || exit 1

    # Check if openapi.json changed
    if git diff openapi.json | grep -q "^+"; then
        echo "✅ OpenAPI spec updated"

        # Validate frontend types (if tool installed)
        if command -v openapi-typescript &> /dev/null; then
            cd packages/frontend
            npx openapi-typescript ../../openapi.json --output src/types/generated/api.ts

            # Stage generated types
            git add src/types/generated/api.ts
        fi

        # Stage openapi.json
        git add ../../openapi.json

        echo "⚠️  Remember to notify frontend/mobile teams if breaking changes exist!"
    fi
fi
```

---

## CI/CD Integration

### GitHub Actions Example

```yaml
# .github/workflows/contract-validation.yml
name: API Contract Validation

on:
  pull_request:
    paths:
      - 'packages/backend/src/api/**'
      - 'packages/backend/src/schemas/**'

jobs:
  validate-contracts:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
        with:
          fetch-depth: 2  # Need to compare with base

      - name: Export Current OpenAPI Spec
        run: |
          cd packages/backend
          python -c "from src.main import app; import json; json.dump(app.openapi(), open('openapi-new.json', 'w'))"

      - name: Get Base OpenAPI Spec
        run: |
          git show origin/main:openapi.json > openapi-old.json

      - name: Check for Breaking Changes
        run: |
          npm install -g @openapitools/openapi-diff
          openapi-diff openapi-old.json openapi-new.json --fail-on-breaking || {
            echo "❌ Breaking changes detected!"
            echo "Please:"
            echo "1. Consider API versioning (v1 → v2)"
            echo "2. Add migration guide"
            echo "3. Notify frontend/mobile teams"
            exit 1
          }

      - name: Validate Frontend Types
        run: |
          cd packages/frontend
          npx openapi-typescript ../../openapi-new.json --output src/types/generated/api-test.ts
          diff src/types/generated/api.ts src/types/generated/api-test.ts || {
            echo "⚠️  Frontend types need update!"
            echo "Run: npx openapi-typescript ../../openapi.json -o src/types/generated/api.ts"
          }
```

---

## Troubleshooting

### Issue: "Module 'main' not found"

```bash
# Solution: Ensure Python path is correct
cd packages/backend
PYTHONPATH=src python -c "from main import app; ..."
```

### Issue: "openapi-typescript command not found"

```bash
# Solution: Install tool
cd packages/frontend
npm install -D openapi-typescript
```

### Issue: "Generated types differ from committed types"

This is **expected** when backend changed API but frontend hasn't updated yet.

**Action:**
1. Review the changes in generated types
2. Update frontend code to use new types
3. Commit updated types

### Issue: "False positive breaking change detection"

Some changes may be flagged as breaking but are actually safe (e.g., adding optional fields).

**Action:**
1. Review the diff carefully
2. If safe, document why it's not breaking
3. Override the check if necessary

---

## Best Practices

### 1. Version Control openapi.json

```bash
# Always commit openapi.json
git add openapi.json
git commit -m "chore: update OpenAPI spec for login API changes"
```

### 2. Verify Before Merging PR

```bash
# Checklist before merge:
# [ ] OpenAPI spec exported
# [ ] No unexpected breaking changes
# [ ] Frontend types updated (if needed)
# [ ] Mobile data classes updated (if needed)
# [ ] Migration guide created (if breaking changes)
# [ ] Affected teams notified
```

### 3. Regular Contract Audits

```bash
# Weekly: Check for contract drift
npm run validate:contracts

# If drift detected, investigate and fix immediately
```

### 4. Document Breaking Changes

```markdown
# CHANGELOG.md

## [2.0.0] - 2024-01-15

### Breaking Changes

- **POST /api/v1/auth/login**: New required field `username`
  - Migration: Add username field to login form
  - Old endpoint deprecated, will be removed in 3 months
```

---

## Tool Installation Guide

See [tools-setup.md](tools-setup.md) for detailed installation instructions.

### Quick Setup

```bash
# Frontend tools
cd packages/frontend
npm install -D openapi-typescript

# OpenAPI diff tool (global)
npm install -g @openapitools/openapi-diff

# Mobile code generation (optional)
# Add to packages/mobile/build.gradle
```

---

## Summary Checklist

Every time you modify backend API:

- [ ] Export OpenAPI spec: `python -c "from src.main import app; ..."`
- [ ] Check for breaking changes: `openapi-diff old.json new.json`
- [ ] Validate frontend types: `npx openapi-typescript ...`
- [ ] Validate Android data classes: Manual review or codegen
- [ ] Notify affected teams via Task tool
- [ ] Create migration guide (if breaking changes)
- [ ] Commit openapi.json
- [ ] Update CHANGELOG.md (if breaking changes)

**This skill is the foundation of reliable frontend-backend integration!**
