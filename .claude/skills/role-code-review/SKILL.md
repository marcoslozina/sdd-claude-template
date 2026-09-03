---
name: role-code-review
description: Reviewing code with real technical criteria — per-layer checklists (domain, use cases, infrastructure, API), OWASP security and privacy/PII checks, performance and test checklists, design-smell naming, and a severity-tagged feedback format. Use when reviewing a PR or diff, giving code feedback, or gating a merge on security, privacy, or design issues.
---

# Skill: Code Review

## Role
Review code with real technical criteria. No generic positive comments.
For every problem found: description, concrete impact, fix with code.

---

## Checklist by layer

### Domain
- [ ] Entities with no framework or infrastructure imports
- [ ] Business logic in the domain, not in application services or adapters
- [ ] Immutable value objects with validation
- [ ] Semantic domain exceptions (`UserNotFoundError`, not `Exception`)
- [ ] Names that communicate business intent

### Application (Use Cases)
- [ ] The use case orchestrates, it does not implement technical details
- [ ] Depends on interfaces (ports), not on concrete implementations
- [ ] One use case = one responsibility
- [ ] No presentation logic (formatting, serialization)

### Infrastructure
- [ ] The adapter implements the domain's port
- [ ] No business logic in adapters
- [ ] Optimized queries (no N+1)
- [ ] Explicit handling of infra errors (timeouts, connections)

### API / Entry Points
- [ ] Input validation at the edge (not in the domain)
- [ ] Semantic HTTP status codes (201 vs 200, 422 vs 400)
- [ ] No business logic in controllers/routes
- [ ] Centralized error handling

---

## Security checklist (OWASP)

- [ ] **Injection**: inputs sanitized before queries/commands
- [ ] **Authentication**: tokens validated, correct expiration
- [ ] **Authorization**: permission check before executing
- [ ] **Secrets**: no secret in code, use env vars
- [ ] **Sensitive data**: never log passwords, tokens, PII
- [ ] **Dependencies**: versions with no known vulnerabilities
- [ ] **Rate limiting**: public endpoints with a request limit

## Privacy checklist (mandatory if there is user data)

- [ ] **Secrets in code**: scan with regex before approving
  - Patterns: `api_key\s*=\s*["']`, `sk-`, `pk_live_`, `postgres://user:pass@`
- [ ] **PII in logs**: emails, names, full IPs, ID documents → never in full
- [ ] **Data minimization**: the API only returns the fields the client needs
- [ ] **Sensitive fields**: passwords, tokens, hashes → never in responses
- [ ] **Encryption in transit**: all communication over HTTPS/TLS
- [ ] **Encryption at rest**: critical PII encrypted in the DB (national ID, cards, medical data)
- [ ] **Retention**: data with a defined expiration policy

### Privacy warning signs — block the PR

| Sign | Severity | Action |
|-------|-----------|--------|
| Hardcoded secret | 🔴 Critical | Block + revoke immediately |
| Password/token in a log | 🔴 Critical | Block |
| `password_hash` in a response | 🔴 Critical | Block |
| Full email in a log | 🟡 Important | Ask for a fix before merge |
| Unencrypted PII in the DB | 🟡 Important | Ask for a fix before merge |
| More data than necessary in a response | 🔵 Suggestion | Comment in the review |

---

## Performance checklist

- [ ] No N+1 queries (eager loading where appropriate)
- [ ] DB indexes on frequently searched fields
- [ ] No unnecessary allocations inside loops
- [ ] Blocking operations on separate threads (where applicable)
- [ ] Cache where recomputing is expensive
- [ ] Pagination on endpoints that return lists

---

## Tests checklist

- [ ] Tests that verify behavior, not implementation
- [ ] Descriptive names: `should_X_when_Y`
- [ ] No conditional logic in tests
- [ ] Each test verifies a single thing
- [ ] Fakes/stubs instead of mocks whenever possible
- [ ] Integration tests against real infra (not H2/SQLite if prod uses Postgres)
- [ ] Coverage of edge cases and error paths, not just the happy path

---

## Bad design signs (detect and name them)

| Sign | Pattern name | What to say |
|-------|-------------------|-----------|
| Class with 500+ lines | God Object | "This class has too many responsibilities. Should we split it by [X] and [Y]?" |
| Method with 5+ parameters | Long Parameter List | "Too many parameters. Should we group them into an object?" |
| Comment that explains WHAT the code does | Non-expressive code | "The code should document itself. Should we rename things so it's obvious?" |
| Switch/if-else over types | Missing polymorphism | "This can be solved with polymorphism. Should we refactor it?" |
| Duplicated logic | DRY violation | "This logic already exists in [place]. Should we extract it?" |
| Test that can never fail | Tautological test | "This test doesn't verify anything real. Should we rewrite it?" |

---

## Feedback format

```
🔴 CRITICAL — [file:line]
Problem: [one-line description]
Impact: [what can happen if it isn't fixed]
Fix:
  [corrected code]

🟡 IMPORTANT — [file:line]
Problem: ...
Impact: ...
Fix: ...

🔵 SUGGESTION — [file:line]
Context: ...
Proposed improvement: ...
```

Only point out real problems. No "good job", no filler.
