---
name: docker-patterns
description: Dockerfile best practices, multi-stage builds, docker-compose patterns, and container optimization. Use when writing Dockerfiles, optimizing builds, or setting up container orchestration.
allowed-tools: Read, Grep, Glob
---

# Docker Patterns

Quick reference for Dockerfile and docker-compose best practices.

## Dockerfile Best Practices

### Multi-Stage Build (Python)

```dockerfile
# Build stage
FROM python:3.11-slim AS builder
COPY --from=ghcr.io/astral-sh/uv:latest /uv /bin/uv
WORKDIR /app

COPY pyproject.toml uv.lock ./
RUN --mount=type=cache,target=/root/.cache/uv \
    uv sync --frozen --no-dev

# Runtime stage
FROM python:3.11-slim AS runtime
WORKDIR /app

# Create non-root user
RUN useradd --create-home appuser
USER appuser

# Copy only what's needed
COPY --from=builder /app/.venv /app/.venv
COPY src/ ./src/

ENV PATH="/app/.venv/bin:$PATH"
CMD ["python", "-m", "app"]
```

### Layer Optimization

```dockerfile
# BAD - cache busted on every code change
COPY . .
RUN pip install -r requirements.txt

# GOOD - dependencies cached separately
COPY requirements.txt .
RUN pip install -r requirements.txt
COPY . .
```

### BuildKit Cache Mounts

```dockerfile
# Cache pip downloads
RUN --mount=type=cache,target=/root/.cache/pip \
    pip install -r requirements.txt

# Cache apt downloads
RUN --mount=type=cache,target=/var/cache/apt \
    apt-get update && apt-get install -y curl
```

## Base Image Selection

| Need | Image | Size |
|------|-------|------|
| Smallest | `python:3.11-alpine` | ~50MB |
| Compatible | `python:3.11-slim` | ~150MB |
| Full tools | `python:3.11` | ~900MB |
| Most secure | `gcr.io/distroless/python3` | ~50MB |

## Security Best Practices

```dockerfile
# Run as non-root
RUN useradd --create-home appuser
USER appuser

# Read-only filesystem (in compose)
# read_only: true

# Drop capabilities (in compose)
# cap_drop: [ALL]

# No new privileges
# security_opt: [no-new-privileges:true]
```

## Docker Compose Patterns

### Development Setup

```yaml
version: "3.8"

services:
  app:
    build:
      context: .
      dockerfile: Dockerfile
    volumes:
      - .:/app                    # Code hot-reload
      - /app/.venv                # Preserve venv
    environment:
      - DEBUG=true
    ports:
      - "8000:8000"
    depends_on:
      db:
        condition: service_healthy

  db:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: app
      POSTGRES_USER: app
      POSTGRES_PASSWORD: secret
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U app"]
      interval: 5s
      timeout: 5s
      retries: 5

volumes:
  postgres_data:
```

### Production Setup

```yaml
version: "3.8"

services:
  app:
    image: myapp:${VERSION:-latest}
    deploy:
      replicas: 2
      resources:
        limits:
          cpus: "1"
          memory: 512M
    restart: unless-stopped
    read_only: true
    security_opt:
      - no-new-privileges:true
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s
```

### Health Checks

```yaml
healthcheck:
  test: ["CMD", "curl", "-f", "http://localhost:8000/health"]
  interval: 30s
  timeout: 10s
  retries: 3
  start_period: 40s
```

```dockerfile
# In Dockerfile
HEALTHCHECK --interval=30s --timeout=10s --retries=3 \
  CMD curl -f http://localhost:8000/health || exit 1
```

### Service Dependencies

```yaml
services:
  app:
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_started
      migrations:
        condition: service_completed_successfully
```

## Common Patterns

### .dockerignore

```
.git
.gitignore
__pycache__
*.pyc
.env
.venv
venv
node_modules
*.md
tests/
.pytest_cache
.coverage
```

### Graceful Shutdown

```dockerfile
# Use exec form for proper signal handling
CMD ["python", "-m", "app"]

# NOT shell form (signals don't propagate)
# CMD python -m app
```

```python
# In application
import signal
import sys

def shutdown_handler(signum, frame):
    print("Shutting down gracefully...")
    sys.exit(0)

signal.signal(signal.SIGTERM, shutdown_handler)
```

### Environment Variables

```yaml
# From file
env_file:
  - .env

# Inline
environment:
  - DATABASE_URL=postgresql://...
  - DEBUG=${DEBUG:-false}
```

### Volumes

```yaml
volumes:
  # Named volume (persistent)
  - postgres_data:/var/lib/postgresql/data

  # Bind mount (development)
  - ./src:/app/src

  # Anonymous volume (preserve container dir)
  - /app/node_modules
```

### Networking

```yaml
services:
  app:
    networks:
      - frontend
      - backend

  db:
    networks:
      - backend

networks:
  frontend:
  backend:
    internal: true  # No external access
```

## Build Optimization

### Parallel Builds

```bash
# Build multiple services in parallel
docker compose build --parallel

# Use BuildKit
DOCKER_BUILDKIT=1 docker build .
```

### Reduce Context Size

```bash
# Check context size
du -sh . --exclude=.git

# Use .dockerignore aggressively
```

### Layer Caching Tips

1. Put rarely-changing instructions first
2. Combine RUN commands with `&&`
3. Use `--mount=type=cache` for package managers
4. Copy dependency files before source code

## Debugging

```bash
# View build stages
docker build --target builder -t myapp:builder .

# Inspect image layers
docker history myapp:latest

# Check container logs
docker logs -f container_name

# Shell into running container
docker exec -it container_name /bin/sh

# Shell into failed build
docker run -it --entrypoint /bin/sh myapp:builder
```
