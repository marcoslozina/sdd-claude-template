---
name: role-cicd
description: CI/CD pipeline design and delivery practices — GitHub Actions job structure and fail-fast ordering, Blue/Green vs Canary vs Rolling deploys, secrets handling, Docker multi-stage build hygiene, merge checklists and pipeline alerting. Use when writing or reviewing workflow YAML, Dockerfiles, deploy strategies, rollback plans, or release gating.
---

# Skill: CI/CD

## Principles
- The pipeline is code. It's versioned, reviewed, and tested.
- Fail fast: the quickest checks run first.
- No deploy to production without going through staging.
- Every deploy must be reversible (rollback < 5 min).

---

## Pipeline structure (GitHub Actions)

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  # 1. Fast checks first (< 2 min)
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Lint
        run: make lint

  # 2. Unit tests (< 5 min)
  unit-tests:
    runs-on: ubuntu-latest
    needs: lint
    steps:
      - uses: actions/checkout@v4
      - name: Unit tests
        run: make test-unit

  # 3. Integration tests (with services)
  integration-tests:
    runs-on: ubuntu-latest
    needs: unit-tests
    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_PASSWORD: test
    steps:
      - uses: actions/checkout@v4
      - name: Integration tests
        run: make test-integration

  # 4. Build and push the image
  build:
    runs-on: ubuntu-latest
    needs: integration-tests
    if: github.ref == 'refs/heads/main'
    steps:
      - uses: actions/checkout@v4
      - name: Build Docker image
        run: docker build -t app:${{ github.sha }} .
      - name: Push to registry
        run: docker push app:${{ github.sha }}
```

---

## Deploy strategies

### Blue/Green
```
Traffic → Load Balancer → Blue (active)
                       → Green (new, no traffic)

1. Deploy to Green
2. Run smoke tests on Green
3. Shift traffic Blue → Green
4. Keep Blue around as rollback
```
**When:** stateless apps, rollback is critical, zero-downtime is mandatory.

### Canary
```
Traffic → 95% → Stable
        →  5% → Canary (new)

Monitor metrics → if OK → increase % → 100%
                → if KO → 0% canary → rollback
```
**When:** you want to validate in production with controlled risk.

### Rolling
```
Pod 1 → update → healthy
Pod 2 → update → healthy
Pod 3 → update → healthy
```
**When:** Kubernetes, gradual updates, you don't need full zero-downtime.

---

## Secrets — rules

```yaml
# ✅ Use GitHub Secrets
env:
  DATABASE_URL: ${{ secrets.DATABASE_URL }}
  API_KEY: ${{ secrets.API_KEY }}

# ❌ NEVER
env:
  DATABASE_URL: "postgres://user:password@host/db"
```

- Secrets are rotated periodically
- Least privilege: each job uses only the secrets it needs
- Never log secrets (`echo $SECRET` → pipeline error)

---

## Docker — best practices

```dockerfile
# Multi-stage build for a minimal image
FROM python:3.12-slim AS builder
WORKDIR /app
COPY pyproject.toml .
RUN pip install uv && uv sync --frozen

FROM python:3.12-slim
WORKDIR /app
COPY --from=builder /app/.venv .venv
COPY src/ src/
USER nonroot
CMD [".venv/bin/python", "-m", "app"]
```

- Final image without build tools
- Non-root user
- `.dockerignore` to exclude `.git`, `tests/`, `docs/`
- Tag with the commit SHA, not `latest` in production

---

## Checklist before merging to main

- [ ] Lint passes
- [ ] Unit tests pass
- [ ] Integration tests pass
- [ ] Coverage did not drop
- [ ] Security scan (Dependabot / Trivy) with no criticals
- [ ] Code review approved
- [ ] Changelog / release notes updated

---

## Minimum pipeline observability

```yaml
- name: Notify on failure
  if: failure()
  uses: slackapi/slack-github-action@v1
  with:
    payload: |
      {
        "text": "❌ Pipeline failed on ${{ github.ref }} — ${{ github.run_url }}"
      }
```

---

## Common architecture decisions in CI/CD

Apply the decision protocol (this project's CLAUDE.md if it defines one, otherwise dev-harness's docs/DECISION_PROTOCOL.md) when facing:
- **CI platform:** GitHub Actions vs GitLab CI vs CircleCI vs Jenkins
- **Deploy strategy:** Blue/Green vs Canary vs Rolling
- **Container registry:** ECR vs GCR vs Docker Hub vs GitHub Packages
- **IaC:** Terraform vs CDK vs Pulumi vs CloudFormation
- **Secrets management:** GitHub Secrets vs AWS Secrets Manager vs Vault
