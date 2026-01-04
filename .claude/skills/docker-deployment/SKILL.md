---
name: docker-deployment
description: Docker best practices (multi-stage builds, security).
allowed-tools: Read, Write, Bash
---

# Docker Deployment

## Multi-Stage Build
```dockerfile
# Stage 1: Builder
FROM python:3.11-slim as builder
WORKDIR /app
COPY requirements.txt .
RUN pip install --user -r requirements.txt

# Stage 2: Runtime
FROM python:3.11-slim
RUN useradd -m -u 1000 appuser
WORKDIR /app
COPY --from=builder /root/.local /home/appuser/.local
COPY --chown=appuser:appuser src/ ./src/
USER appuser
CMD ["uvicorn", "src.main:app"]
```

## Docker Compose
```yaml
version: '3.8'
services:
  backend:
    build: ./packages/backend
    environment:
      DATABASE_URL: ${DATABASE_URL}
    depends_on:
      - mongodb
  mongodb:
    image: mongo:7.0
    volumes:
      - mongodb_data:/data/db
volumes:
  mongodb_data:
```

## Best Practices
- Use multi-stage builds (smaller images)
- Run as non-root user
- Use .dockerignore
- Pin specific versions
- Health checks
