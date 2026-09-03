---
name: infra-docker
description: Docker standards - multi-stage builds for Python, Go, Node/TypeScript and Java, .dockerignore, Docker Compose for local dev, non-root images, and a container security checklist. Use when writing or reviewing a Dockerfile, docker-compose.yml, .dockerignore, or containerizing an app for CI/CD.
---

# Skill: Docker

## Principles
- Minimal images: only what production needs
- Multi-stage builds: separate build from runtime
- Non-root user in production
- One process per container
- Immutable: no runtime changes, all config via env vars

---

## Multi-stage builds by language

### Python
```dockerfile
# Stage 1: dependencies
FROM python:3.12-slim AS builder
WORKDIR /app
COPY pyproject.toml uv.lock ./
RUN pip install uv && uv sync --frozen --no-dev

# Stage 2: runtime
FROM python:3.12-slim
WORKDIR /app
COPY --from=builder /app/.venv .venv
COPY src/ src/
RUN useradd -r -s /bin/false appuser
USER appuser
CMD [".venv/bin/python", "-m", "app"]
```

### Go
```dockerfile
# Stage 1: build
FROM golang:1.22-alpine AS builder
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -o /app/server ./cmd/api

# Stage 2: minimal runtime
FROM scratch
COPY --from=builder /app/server /server
COPY --from=builder /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/
USER 65534:65534
ENTRYPOINT ["/server"]
```

### Node / TypeScript
```dockerfile
# Stage 1: build
FROM node:22-alpine AS builder
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci --include=dev
COPY . .
RUN npm run build

# Stage 2: runtime
FROM node:22-alpine
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci --omit=dev && addgroup -S appgroup && adduser -S appuser -G appgroup
COPY --from=builder /app/dist ./dist
USER appuser
CMD ["node", "dist/index.js"]
```

### Java
```dockerfile
# Stage 1: build
FROM eclipse-temurin:21-jdk AS builder
WORKDIR /app
COPY gradle/ gradle/
COPY gradlew build.gradle.kts settings.gradle.kts ./
RUN ./gradlew dependencies --no-daemon
COPY src/ src/
RUN ./gradlew bootJar --no-daemon

# Stage 2: runtime (JRE, not JDK)
FROM eclipse-temurin:21-jre
WORKDIR /app
COPY --from=builder /app/build/libs/*.jar app.jar
RUN useradd -r -s /bin/false appuser
USER appuser
ENTRYPOINT ["java", "-jar", "app.jar"]
```

---

## .dockerignore — always present

```
.git
.github
.claude
docs/
tests/
*.md
.env
.env.*
node_modules/
__pycache__/
*.pyc
.pytest_cache/
.mypy_cache/
dist/
build/
*.log
```

---

## Docker Compose — local development

```yaml
# docker-compose.yml
services:
  app:
    build: .
    ports:
      - "8080:8080"
    environment:
      DATABASE_URL: postgresql://postgres:dev@db:5432/app
      REDIS_URL: redis://cache:6379
    depends_on:
      db:
        condition: service_healthy
      cache:
        condition: service_started
    volumes:
      - ./src:/app/src  # hot reload in dev

  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_PASSWORD: dev
      POSTGRES_DB: app
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      timeout: 5s
      retries: 5

  cache:
    image: redis:7-alpine
    ports:
      - "6379:6379"

volumes:
  postgres_data:
```

---

## Security checklist

- [ ] Non-root user (`USER appuser`)
- [ ] No secrets in the Dockerfile or in the image (use `--secret` or runtime env vars)
- [ ] Base image pinned to a specific tag, not `latest`
- [ ] `.dockerignore` present and complete
- [ ] No build tooling in the final image
- [ ] Vulnerability scanning in CI (Trivy / Snyk)

```yaml
# CI: scan the image with Trivy
- name: Scan image
  uses: aquasecurity/trivy-action@v0.36.0
  with:
    image-ref: app:${{ github.sha }}
    severity: CRITICAL,HIGH
    exit-code: '1'
```

---

## Ballpark image sizes

| Stack | Base | Multi-stage |
|-------|------|-------------|
| Go | ~300MB (golang) | ~10MB (scratch) |
| Python | ~200MB (python:slim) | ~120MB |
| Node | ~180MB (node:alpine) | ~100MB |
| Java | ~400MB (jdk) | ~200MB (jre) |

---

## Common Docker decisions

Apply the decision protocol (this project's CLAUDE.md if it defines one, otherwise dev-harness's docs/DECISION_PROTOCOL.md) when facing:
- **Base image:** distroless vs alpine vs slim vs scratch
- **Orchestration:** Docker Compose vs ECS vs Kubernetes
- **Registry:** ECR vs GHCR vs Docker Hub
- **Runtime secrets:** env vars vs Docker secrets vs AWS Secrets Manager
- **Dev volumes:** bind mount vs named volume
