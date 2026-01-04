---
name: codebase-analysis
description: Understand project structure, locate key files, and analyze dependencies. Use BEFORE starting any development task to build a code map.
allowed-tools: Read, Grep, Glob, Bash
---

# Codebase Analysis Skill

## Purpose
Quickly understand project structure and locate key files before implementation.

## Analysis Steps

### 1. Project Structure Overview
```bash
tree -L 3 -I 'node_modules|__pycache__|.git|dist|build'
ls -la packages/*/
```

### 2. Locate Entry Points
```bash
# Frontend
find packages/frontend -name "main.ts" -o -name "index.ts"

# Backend
find packages/backend -name "main.py" -o -name "app.py"

# Mobile
find packages/mobile -name "MainActivity.kt"
```

### 3. Find Key Files
```bash
# Routes/API
find . -name "*route*" -o -name "*api*" | grep -v node_modules

# Models/Schemas
find . -name "*model*" -o -name "*schema*" | grep -v node_modules

# Components
find . -name "*.vue" -o -name "*Screen.kt"
```

### 4. Understand Dependencies
```bash
# Frontend
cat packages/frontend/package.json | jq '.dependencies'

# Backend
cat packages/backend/requirements.txt

# Check workspace structure
cat package.json | grep -A 10 "workspaces"
```

## Checklist
- [ ] Identified project structure (monorepo layout)
- [ ] Located entry points
- [ ] Found relevant modules for current task
- [ ] Understood dependencies between packages
- [ ] Checked existing patterns to follow

## Remember
Always use this skill BEFORE starting implementation to avoid duplicating existing code or breaking patterns.
