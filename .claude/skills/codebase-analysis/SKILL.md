---
name: codebase-analysis
description: Understand project structure, locate key files, and analyze dependencies. Use BEFORE starting any development task to build a code map. Apply when: (1) Starting work on a new codebase or unfamiliar project, (2) Before implementing new features to understand existing patterns, (3) When asked to locate specific functionality or files, (4) Before refactoring to understand dependencies and impact, (5) When analyzing project architecture or tech stack, (6) Before writing code to avoid duplicating existing functionality, (7) When onboarding to a project. CRITICAL: Always use this skill FIRST before making code changes to understand context and avoid breaking existing patterns.
---

# Codebase Analysis Skill

## Purpose

Systematically understand project structure, architecture, and patterns BEFORE implementing features. This prevents:
- Duplicating existing functionality
- Breaking established patterns
- Missing critical dependencies
- Introducing inconsistent code style

## Analysis Workflow

### Phase 1: High-Level Overview

**Goal:** Understand project type, structure, and tech stack

#### 1.1 Identify Project Type

Use Glob and Read tools to determine:

```
# Check for monorepo indicators
packages/*/package.json
pnpm-workspace.yaml
lerna.json
nx.json

# Check for single repo
package.json (root level only)
requirements.txt
Cargo.toml
go.mod
```

**Common Patterns:**
- **Monorepo**: Multiple `package.json` files, workspace config, packages/ or apps/ directory
- **Single Repo**: One main config file at root
- **Fullstack**: Separate frontend/backend/mobile directories

#### 1.2 Map Directory Structure

Use Glob to identify key directories:

```
# Frontend/UI
**/src/**
**/components/**
**/pages/**
**/screens/**

# Backend/API
**/routes/**
**/api/**
**/controllers/**
**/handlers/**

# Shared/Domain Logic
**/models/**
**/schemas/**
**/entities/**
**/types/**

# Configuration
**/config/**
**/.env.example
**/tsconfig.json
```

**Document findings:**
```
Project Structure:
- Type: Monorepo (pnpm workspaces)
- Packages:
  - packages/frontend (Vue 3 + TypeScript)
  - packages/backend (FastAPI + Python)
  - packages/mobile (Kotlin + Jetpack Compose)
  - packages/shared (TypeScript types)
```

### Phase 2: Entry Points & Architecture

**Goal:** Find where code execution begins and understand flow

#### 2.1 Locate Entry Points

**Frontend:**
```
# Use Glob to find
**/main.ts
**/main.tsx
**/index.ts
**/App.vue
**/App.tsx
```

**Backend:**
```
# Use Glob to find
**/main.py
**/app.py
**/server.ts
**/index.ts
**/main.go
```

**Mobile:**
```
# Use Glob to find
**/MainActivity.kt
**/MainApplication.kt
**/App.tsx (React Native)
```

#### 2.2 Trace Architecture Flow

Use Read tool on entry points, then follow imports:

**Example Analysis:**
```typescript
// Read: packages/backend/src/main.ts

import { createApp } from './app'
import { setupRoutes } from './routes'
import { connectDatabase } from './database'

// Findings:
// - App factory pattern (app.ts)
// - Routes centralized (routes/index.ts likely)
// - Database module (database.ts)
```

**Follow the trail:**
1. Read entry point file
2. Identify key imports and initialization
3. Read those files to understand structure
4. Map the dependency flow

### Phase 3: Locate Relevant Code

**Goal:** Find existing functionality related to your task

#### 3.1 Search by Functionality

Use Grep with semantic search terms:

**Example: Adding authentication**
```
# Search for existing auth patterns
Grep: "authentication|auth|login|jwt"
Grep: "middleware.*auth"
Grep: "class.*Auth.*Service"

# Check existing routes
Grep: "POST.*login|/api/auth"

# Find existing user management
Grep: "interface User|class User|type User"
```

**Example: Adding a new API endpoint**
```
# Find existing endpoint patterns
Grep: "@Post|@Get|@Put|@Delete" (NestJS/decorators)
Grep: "router\.(get|post|put|delete)" (Express)
Grep: "@app\.(get|post|put|delete)" (FastAPI)

# Find similar endpoints
Grep: "/api/users|/api/products"
```

#### 3.2 Understand File Organization

Use Glob patterns to understand conventions:

```
# Controllers/Handlers
**/*Controller.ts
**/*Handler.ts
**/*Route.ts

# Services/Business Logic
**/*Service.ts
**/*UseCase.ts

# Data Access
**/*Repository.ts
**/*Model.ts
**/*Entity.ts

# API DTOs
**/*Dto.ts
**/*Request.ts
**/*Response.ts
```

**Document pattern:**
```
Backend Structure Pattern:
- routes/*.ts → Route definitions
- controllers/*.ts → Request handlers
- services/*.ts → Business logic
- repositories/*.ts → Data access
- models/*.ts → Database schemas
- dtos/*.ts → Request/Response types
```

### Phase 4: Analyze Dependencies

**Goal:** Understand what libraries and patterns are used

#### 4.1 Tech Stack Analysis

**Frontend (Node.js):**
```
# Read package.json
Read: packages/frontend/package.json

Key dependencies to note:
- Framework: vue@3.x, react@18.x
- State: pinia, zustand, redux
- Router: vue-router, react-router
- HTTP: axios, fetch
- UI: element-plus, ant-design, mui
- Build: vite, webpack
```

**Backend (Python):**
```
# Read requirements.txt or pyproject.toml
Read: packages/backend/requirements.txt

Key dependencies:
- Framework: fastapi, django, flask
- ORM: sqlalchemy, prisma
- Database: psycopg2, motor, pymongo
- Validation: pydantic
- Auth: python-jose, passlib
```

**Backend (Node.js):**
```
# Read package.json
Read: packages/backend/package.json

Key dependencies:
- Framework: express, nestjs, fastify
- ORM: prisma, typeorm, mongoose
- Validation: zod, joi
- Auth: jsonwebtoken, passport
```

#### 4.2 Identify Patterns and Conventions

Use Grep to find common patterns:

**Dependency Injection:**
```
Grep: "@Injectable|@inject|@Inject"
Grep: "constructor.*:.*Service"
```

**Error Handling:**
```
Grep: "class.*Error extends"
Grep: "throw new"
Grep: "try.*catch"
```

**Validation:**
```
Grep: "@IsEmail|@IsString" (class-validator)
Grep: "z\.object|z\.string" (zod)
Grep: "Joi\.object|Joi\.string" (joi)
```

**Testing:**
```
Grep: "describe\\(|it\\(|test\\("
Grep: "@Test|@pytest"
```

### Phase 5: Understand Data Models

**Goal:** Map database schema and data flow

#### 5.1 Find Database Models

Use Glob to locate:

```
# ORM Models
**/models/**/*.ts
**/models/**/*.py
**/entities/**/*.ts

# Database Schemas
**/schemas/**/*.ts
**/migrations/**/*

# API DTOs
**/dtos/**/*.ts
```

#### 5.2 Analyze Model Relationships

Read model files to understand:

```typescript
// Example: Read user model
// File: models/User.ts

@Entity()
class User {
  @PrimaryGeneratedColumn()
  id: number;

  @Column()
  email: string;

  @OneToMany(() => Post, post => post.author)
  posts: Post[];
}

// Findings:
// - User has many Posts
// - Post belongs to User (author relationship)
// - Need to check Post model next
```

**Document relationships:**
```
Data Model Map:
- User
  - id (PK)
  - email, name
  - Has many: Posts, Comments

- Post
  - id (PK)
  - authorId (FK -> User)
  - Belongs to: User (author)
  - Has many: Comments

- Comment
  - id (PK)
  - postId (FK -> Post)
  - authorId (FK -> User)
```

### Phase 6: Code Style & Conventions

**Goal:** Identify coding patterns to maintain consistency

#### 6.1 Naming Conventions

Use Grep to identify patterns:

```
# Check function naming
Grep: "function |const .* = |async function"

# Check class naming
Grep: "class |interface |type "

# Check file naming
Glob: **/*.ts (check if camelCase or kebab-case)
Glob: **/*Service.ts (suffix patterns)
```

**Document conventions:**
```
Code Conventions:
- Files: kebab-case (user-service.ts)
- Classes: PascalCase (UserService)
- Functions: camelCase (getUserById)
- Constants: UPPER_SNAKE_CASE (MAX_RETRIES)
- Interfaces: PascalCase with I prefix (IUserRepository) OR no prefix (UserRepository)
```

#### 6.2 Architecture Patterns

Use Grep to identify:

```
# Layered architecture
Grep: "Controller|Service|Repository"

# MVI (Android)
Grep: "Intent.*State|ViewModel.*Intent"

# MVVM
Grep: "ViewModel|@HiltViewModel"

# Clean Architecture
Grep: "UseCase|Domain|Infrastructure"
```

**Document architecture:**
```
Architecture: Clean Architecture + MVI (Mobile)
- Presentation: ViewModels, Intents, States, Composables
- Domain: UseCases, Repository Interfaces
- Data: Repository Implementations, API, Database
```

### Phase 7: Configuration & Environment

**Goal:** Understand configuration and environment setup

#### 7.1 Environment Variables

Use Glob and Read:

```
# Find env files
Glob: **/.env.example
Glob: **/.env.template

Read: .env.example

# Document required config
Required Environment Variables:
- DATABASE_URL (PostgreSQL connection)
- JWT_SECRET (Authentication)
- API_BASE_URL (Backend API endpoint)
- PORT (Server port)
```

#### 7.2 Build & Development Config

Use Read on config files:

```
# Build configuration
Read: vite.config.ts
Read: tsconfig.json
Read: webpack.config.js

# Development setup
Read: package.json (scripts section)
Read: Makefile
Read: docker-compose.yml
```

## Analysis Documentation Template

After analysis, document findings:

```markdown
# Codebase Analysis: [Task Name]

## Project Overview
- Type: Monorepo (pnpm workspaces)
- Packages: frontend (Vue 3), backend (FastAPI), mobile (Kotlin)
- Architecture: Clean Architecture + MVI (mobile)

## Relevant Files for Task
- API: `packages/backend/src/routes/users.py`
- Service: `packages/backend/src/services/user_service.py`
- Repository: `packages/backend/src/repositories/user_repository.py`
- Model: `packages/backend/src/models/user.py`
- DTO: `packages/backend/src/dtos/user_dto.py`

## Existing Patterns to Follow
- Route Handler → Service → Repository → Database
- Use Pydantic for validation
- Return consistent response format: `{"success": bool, "data": any}`
- Use dependency injection via FastAPI Depends()

## Dependencies
- Database: PostgreSQL (via SQLAlchemy)
- Auth: JWT tokens (python-jose)
- Validation: Pydantic v2

## Code Conventions
- Files: snake_case
- Classes: PascalCase
- Functions: snake_case
- Type hints required
- Docstrings for public functions

## Similar Implementations to Reference
- `POST /api/users` → `routes/users.py:45`
- `UserService.create_user()` → `services/user_service.py:23`
```

## Common Analysis Scenarios

### Scenario 1: Adding a New API Endpoint

**Analysis Steps:**
1. Find existing API routes (Grep for route patterns)
2. Read route files to understand structure
3. Identify controller/handler pattern
4. Find service layer implementation
5. Check data access (repository/ORM)
6. Review validation approach
7. Check error handling patterns

### Scenario 2: Adding a New UI Feature

**Analysis Steps:**
1. Find similar components (Glob for component files)
2. Understand state management (Grep for store/context)
3. Identify API integration patterns
4. Check routing structure
5. Review styling approach (CSS modules, styled-components, etc.)
6. Identify reusable components

### Scenario 3: Refactoring Existing Code

**Analysis Steps:**
1. Map all files that import the target code (Grep for imports)
2. Identify dependencies and dependents
3. Check test coverage
4. Understand current behavior
5. Identify related functionality
6. Check for duplicate code that could be consolidated

### Scenario 4: Bug Investigation

**Analysis Steps:**
1. Locate relevant code (Grep for error messages, function names)
2. Trace execution flow from entry point
3. Identify data sources and transformations
4. Check error handling
5. Review related tests
6. Check recent changes (git blame/log)

## Key Principles

**Always Analyze Before Coding:**
- Prevents duplicating existing code
- Maintains consistency with project patterns
- Avoids breaking dependencies
- Ensures proper integration

**Use Tools, Not Bash:**
- Prefer Glob for file finding (not `find`)
- Prefer Grep for code search (not `grep` command)
- Prefer Read for file reading (not `cat`)
- Use tools for better context and permissions

**Document Findings:**
- Create a mental (or written) map
- Note patterns and conventions
- Identify files to modify/create
- List dependencies to consider

**Focus on Relevance:**
- Don't analyze everything
- Focus on code related to your task
- Follow the imports and usage
- Understand the immediate context

## Quick Analysis Checklist

- [ ] Identified project type (monorepo vs single)
- [ ] Located entry points
- [ ] Found relevant existing code
- [ ] Understood file organization pattern
- [ ] Reviewed tech stack and dependencies
- [ ] Identified code conventions (naming, structure)
- [ ] Mapped data models and relationships
- [ ] Checked error handling patterns
- [ ] Reviewed test approach
- [ ] Documented architecture pattern
- [ ] Listed files to modify/create
- [ ] Ready to implement following established patterns
