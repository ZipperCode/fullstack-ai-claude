# Tool Installation and Setup

## Frontend Tools

### openapi-typescript

Generates TypeScript types from OpenAPI specifications.

```bash
cd packages/frontend
npm install -D openapi-typescript

# Usage
npx openapi-typescript ../../openapi.json -o src/types/generated/api.ts
```

**When to use**: After any backend API changes to regenerate frontend types.

### @openapitools/openapi-diff

Detects breaking changes between OpenAPI spec versions.

```bash
# Global installation
npm install -g @openapitools/openapi-diff

# Usage
openapi-diff openapi-old.json openapi-new.json --fail-on-breaking
```

**When to use**: Before merging PRs with API changes to detect breaking changes.

## Mobile Tools

### OpenAPI Generator for Kotlin

Generates Kotlin data classes from OpenAPI specs.

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

**When to use**: Set up once, then regenerate after backend API changes.

## CI/CD Integration

### Pre-commit Hook

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

### GitHub Actions

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
