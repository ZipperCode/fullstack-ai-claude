---
name: docker-deployment
description: Docker containerization best practices for production deployments. Use when creating Dockerfiles, setting up Docker Compose configurations, or optimizing container builds. Apply for multi-stage builds, security hardening (non-root users, minimal base images), volume management, networking setup, and deployment orchestration. Essential for ensuring secure, efficient, and production-ready containerized applications.
---

# Docker Deployment Best Practices

Create secure, efficient, production-ready Docker containers following industry best practices.

## Multi-Stage Builds

Multi-stage builds dramatically reduce final image size by separating build dependencies from runtime requirements.

### Pattern: Python Application

```dockerfile
# Stage 1: Builder - Install dependencies
FROM python:3.11-slim as builder

WORKDIR /app

# Install build dependencies
COPY requirements.txt .
RUN pip install --user --no-cache-dir -r requirements.txt

# Stage 2: Runtime - Minimal final image
FROM python:3.11-slim

# Create non-root user
RUN useradd -m -u 1000 appuser

WORKDIR /app

# Copy only installed packages from builder
COPY --from=builder /root/.local /home/appuser/.local

# Copy application code
COPY --chown=appuser:appuser src/ ./src/

# Switch to non-root user
USER appuser

# Ensure packages are in PATH
ENV PATH=/home/appuser/.local/bin:$PATH

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD python -c "import requests; requests.get('http://localhost:8000/health')"

EXPOSE 8000

CMD ["uvicorn", "src.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

**Benefits**:
- Build dependencies (compilers, dev tools) excluded from final image
- Smaller image size = faster deployment
- Reduced attack surface

### Pattern: Node.js Application

```dockerfile
# Stage 1: Dependencies
FROM node:20-alpine as deps
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production

# Stage 2: Builder
FROM node:20-alpine as builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

# Stage 3: Runtime
FROM node:20-alpine
RUN addgroup -g 1000 appgroup && adduser -u 1000 -G appgroup -s /bin/sh -D appuser
WORKDIR /app
COPY --from=deps --chown=appuser:appgroup /app/node_modules ./node_modules
COPY --from=builder --chown=appuser:appgroup /app/dist ./dist
USER appuser
EXPOSE 3000
CMD ["node", "dist/index.js"]
```

## Security Best Practices

### 1. Run as Non-Root User

**Problem**: Running as root increases security risk.

**Solution**: Create and switch to non-root user.

```dockerfile
# Alpine Linux
RUN addgroup -g 1000 appgroup && \
    adduser -u 1000 -G appgroup -s /bin/sh -D appuser

# Debian/Ubuntu
RUN useradd -m -u 1000 appuser

# Switch to non-root user
USER appuser
```

### 2. Use Minimal Base Images

**Prefer**: `alpine`, `slim`, or `distroless` variants.

```dockerfile
# ❌ Bloated (1.2 GB)
FROM python:3.11

# ✅ Slim (150 MB)
FROM python:3.11-slim

# ✅ Alpine (50 MB)
FROM python:3.11-alpine
```

### 3. Don't Expose Secrets

```dockerfile
# ❌ BAD: Secrets in environment
ENV DATABASE_PASSWORD=secret123

# ✅ GOOD: Use Docker secrets or external config
# Pass via docker run -e or docker-compose.yml
```

### 4. Pin Specific Versions

```dockerfile
# ❌ Unpredictable
FROM python:3

# ✅ Reproducible
FROM python:3.11.7-slim
```

## .dockerignore File

Exclude unnecessary files to speed up builds and reduce image size.

```.dockerignore
# Version control
.git
.gitignore

# Dependencies
node_modules
__pycache__
*.pyc
.venv

# IDE
.vscode
.idea
*.swp

# Tests
tests
*.test.js
*.spec.ts

# Documentation
README.md
docs/

# Environment
.env
.env.local
*.log
```

## Docker Compose Configuration

Orchestrate multi-container applications with proper networking and volume management.

### Production-Ready docker-compose.yml

```yaml
version: '3.8'

services:
  backend:
    build:
      context: ./packages/backend
      dockerfile: Dockerfile
    container_name: app-backend
    restart: unless-stopped
    environment:
      DATABASE_URL: ${DATABASE_URL}
      REDIS_URL: redis://redis:6379
      NODE_ENV: production
    depends_on:
      mongodb:
        condition: service_healthy
      redis:
        condition: service_started
    networks:
      - app-network
    ports:
      - "8000:8000"
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/health"]
      interval: 30s
      timeout: 3s
      retries: 3

  frontend:
    build:
      context: ./packages/frontend
      dockerfile: Dockerfile
    container_name: app-frontend
    restart: unless-stopped
    depends_on:
      - backend
    networks:
      - app-network
    ports:
      - "3000:3000"

  mongodb:
    image: mongo:7.0
    container_name: app-mongodb
    restart: unless-stopped
    environment:
      MONGO_INITDB_ROOT_USERNAME: ${MONGO_USERNAME}
      MONGO_INITDB_ROOT_PASSWORD: ${MONGO_PASSWORD}
    volumes:
      - mongodb_data:/data/db
      - ./mongo-init.js:/docker-entrypoint-initdb.d/mongo-init.js:ro
    networks:
      - app-network
    ports:
      - "27017:27017"
    healthcheck:
      test: ["CMD", "mongosh", "--eval", "db.adminCommand('ping')"]
      interval: 10s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    container_name: app-redis
    restart: unless-stopped
    volumes:
      - redis_data:/data
    networks:
      - app-network
    command: redis-server --appendonly yes

volumes:
  mongodb_data:
    driver: local
  redis_data:
    driver: local

networks:
  app-network:
    driver: bridge
```

### Key Configuration Elements

**restart policies**:
- `no`: Never restart (default)
- `always`: Always restart
- `unless-stopped`: Restart unless explicitly stopped
- `on-failure`: Restart only on failure

**depends_on with health checks**:
```yaml
depends_on:
  mongodb:
    condition: service_healthy  # Wait for health check
```

**Named volumes** for data persistence:
```yaml
volumes:
  mongodb_data:  # Persists across container restarts
```

## Health Checks

Essential for production deployments and orchestration tools.

### In Dockerfile

```dockerfile
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD curl -f http://localhost:8000/health || exit 1
```

### In docker-compose.yml

```yaml
healthcheck:
  test: ["CMD", "curl", "-f", "http://localhost:8000/health"]
  interval: 30s
  timeout: 3s
  start_period: 10s
  retries: 3
```

## Build Optimization Tips

### 1. Layer Caching

Order commands from least to most frequently changed:

```dockerfile
# ✅ GOOD: Dependencies change less often than source code
COPY requirements.txt .
RUN pip install -r requirements.txt
COPY src/ ./src/

# ❌ BAD: Invalidates cache on every source change
COPY . .
RUN pip install -r requirements.txt
```

### 2. Combine RUN Commands

```dockerfile
# ❌ Multiple layers
RUN apt-get update
RUN apt-get install -y curl
RUN apt-get clean

# ✅ Single layer
RUN apt-get update && \
    apt-get install -y curl && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*
```

### 3. Use Build Arguments

```dockerfile
ARG NODE_ENV=production
ENV NODE_ENV=$NODE_ENV

# Build with: docker build --build-arg NODE_ENV=development .
```

## Common Commands

```bash
# Build image
docker build -t myapp:latest .

# Build with custom Dockerfile
docker build -f Dockerfile.prod -t myapp:prod .

# Run container
docker run -d -p 8000:8000 --name myapp-container myapp:latest

# View logs
docker logs -f myapp-container

# Execute command in running container
docker exec -it myapp-container /bin/sh

# Stop and remove container
docker stop myapp-container && docker rm myapp-container

# Docker Compose operations
docker-compose up -d          # Start all services
docker-compose down           # Stop and remove containers
docker-compose logs -f backend  # Follow logs
docker-compose exec backend sh  # Execute command
```

## Deployment Checklist

Before deploying to production:

- [ ] Using multi-stage builds
- [ ] Running as non-root user
- [ ] Using minimal base images (slim/alpine)
- [ ] .dockerignore configured
- [ ] No secrets in Dockerfile or image
- [ ] Versions pinned (base images, dependencies)
- [ ] Health checks implemented
- [ ] Restart policy configured
- [ ] Volumes for persistent data
- [ ] Resource limits set (CPU, memory)
- [ ] Logging configured
- [ ] Monitoring/metrics exposed
