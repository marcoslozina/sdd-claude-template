---
name: role-backend
description: Language-agnostic REST and GraphQL API design — URL conventions, semantic HTTP status codes, consistent error payloads, cursor vs offset pagination, DB schema and migration rules, N+1 removal, error translation across layers, and request logging. Use when designing or reviewing API endpoints, DB schemas, migrations, auth, rate limiting, or backend error handling.
---

# Skill: Backend

## Applies to
REST and GraphQL APIs, agnostic of language and framework.

---

## REST API design

### URL conventions
```
GET    /users              → list (with pagination)
GET    /users/:id          → get one
POST   /users              → create
PUT    /users/:id          → full replace
PATCH  /users/:id          → partial update
DELETE /users/:id          → delete

# Nested resources (2 levels maximum)
GET    /users/:id/orders
POST   /users/:id/orders

# Non-CRUD actions → verbs as sub-resources
POST   /users/:id/activate
POST   /orders/:id/cancel
```

### Semantic HTTP status codes
| Situation | Code |
|-----------|--------|
| Successful creation | 201 Created |
| Operation with no response body | 204 No Content |
| Resource not found | 404 Not Found |
| Invalid input | 422 Unprocessable Entity |
| No permission | 403 Forbidden |
| Not authenticated | 401 Unauthorized |
| State conflict | 409 Conflict |
| Server error | 500 Internal Server Error |

### Consistent error format
```json
{
  "error": {
    "code": "USER_NOT_FOUND",
    "message": "No user found with id abc-123",
    "details": []
  }
}
```

---

## Pagination

```json
// Cursor-based (preferred for large or changing datasets)
GET /users?cursor=eyJpZCI6MTAwfQ&limit=20

{
  "data": [...],
  "pagination": {
    "next_cursor": "eyJpZCI6MTIwfQ",
    "has_more": true
  }
}

// Offset (simple, for static admin tables)
GET /users?page=2&per_page=20

{
  "data": [...],
  "pagination": {
    "page": 2,
    "per_page": 20,
    "total": 340
  }
}
```

**Cursor-based** for feeds, large lists, data that keeps updating.
**Offset** for admin panels, reports, static datasets.

---

## Database

### Schema rules
- Primary keys: UUIDs (no autoincrement exposed in the API)
- Timestamps: `created_at`, `updated_at` on every table
- Soft delete: a `deleted_at` column instead of a physical DELETE when auditing is required
- Indexes on: foreign keys, frequently searched fields, sorting fields

### Migrations
- Always versioned migrations (Alembic, Flyway, Liquibase)
- Each migration: one atomic change
- No business logic in migrations
- Backwards-compatible migrations whenever possible

### N+1 — detect and eliminate
```
❌ N+1:
  for user in users:         # 1 query
      print(user.orders)     # N queries

✅ Eager loading:
  users = db.query(User).options(joinedload(User.orders)).all()
```

---

## Error handling

```
                  ┌─────────────────────────────┐
Invalid request   │ Validate at entry point (422)│
                  └─────────────┬───────────────┘
                                │ clean input
                  ┌─────────────▼───────────────┐
Business error    │ Domain exception             │ → 409 / 404 / 422
                  └─────────────┬───────────────┘
                                │
                  ┌─────────────▼───────────────┐
Infra error       │ Adapter catches and translates│ → 500 + log
                  └─────────────────────────────┘
```

Never let infrastructure exceptions (SQL, HTTP, timeout) reach the client untranslated.

---

## Minimum observability

```python
# Every request must log:
{
  "request_id": "uuid",          # traceability
  "method": "POST",
  "path": "/users",
  "status": 201,
  "duration_ms": 45,
  "user_id": "abc"               # business context
}

# Never log:
# - passwords, tokens, API keys
# - full personal data (PII)
# - full payloads in production
```

---

## Common Backend decisions

Apply the decision protocol (this project's CLAUDE.md if it defines one, otherwise dev-harness's docs/DECISION_PROTOCOL.md) when facing:
- **API style:** REST vs GraphQL vs gRPC vs tRPC
- **Auth:** JWT vs sessions vs OAuth2
- **Pagination:** cursor vs offset
- **Queue:** sync vs async with a queue
- **Cache:** where and what to cache, invalidation
- **Rate limiting:** per user vs per IP vs per endpoint
