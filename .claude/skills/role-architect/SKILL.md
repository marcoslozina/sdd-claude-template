---
name: role-architect
description: System design before implementation — the dependency rule, layer separation, choosing between Layered/Clean/Hexagonal/microservices, the decision protocol, ADR authoring, architectural smells, and Ports & Adapters / CQRS / event-driven patterns. Use when designing a new system, picking a pattern or framework, spotting layer coupling, or writing an ADR.
---

# Skill: Architect

## Role
Design systems before implementing them. There is never code without a documented architecture decision.

## When to activate this skill
- The user describes a new system
- You have to choose between patterns or frameworks
- Coupling between layers is detected
- The current solution is not going to scale

---

## Fundamental principles (agnostic)

### Dependency rule
```
Entry Points → Application → Domain ← Infrastructure
```
The arrows point inward. The domain knows nothing external. Ever.

### Separation of concerns
| Layer | Responsibility | What it does NOT do |
|------|-----------------|----------------|
| Domain | Business logic, rules | Know about the DB, HTTP, frameworks |
| Application | Orchestrate use cases | Implement technical details |
| Infrastructure | Technical adapters | Contain business logic |
| Entry Points | Receive requests | Business logic |

### Dependency inversion
```
Domain defines the interface (Port)
Infrastructure implements the interface (Adapter)
Application uses the interface, not the implementation
```

---

## When to apply which architecture

| Complexity | Pattern | When |
|-------------|--------|--------|
| Simple script / CLI | Flat + functions | 1-2 responsibilities |
| Small app | MVC or Layered | Basic CRUD, small team |
| Medium app | Clean Architecture | Real business logic |
| Complex app | Hexagonal / Ports & Adapters | Multiple integrations |
| Distributed | Microservices + events | Scale of teams / domains |

**Rule:** start simple. Migrate when the pain is real, not anticipated.

---

## Architecture decision protocol

For every significant decision:

```
🏗️ DECISION: [title]

Context: [the force driving the decision]

Option A — [name]
  ✓ [advantage]
  ✗ [drawback]

Option B — [name]
  ✓ [advantage]
  ✗ [drawback]

Recommendation: [option] because [technical reason]

Do you confirm?
```

After confirming → create an ADR.

---

## ADR (Architecture Decision Record)

Folder: `docs/adr/ADR-XXX-kebab-title.md`

```markdown
# ADR-001: [Title]

**Status:** Accepted | Proposed | Deprecated
**Date:** YYYY-MM-DD

## Context
[What force or problem drives this decision]

## Decision
[What we decided to do]

## Consequences
### Positive
- ...
### Negative / Tradeoffs
- ...

## Rejected alternatives
- [Alternative A]: rejected because ...
```

---

## Architectural warning signs

| Sign | Problem | Solution |
|-------|---------|---------|
| Business logic in a controller/route | Layer violation | Move it to a use case |
| Use case importing the ORM directly | Dependency violation | Extract a port + adapter |
| Entity that knows the framework | Contaminated domain | Separate the entity from the ORM model |
| Service that does everything | God Object | Split by responsibility |
| Tests that mock the DB in use cases | Test coupled to infra | Use a fake repository |

---

## Key patterns

### Port & Adapter (Hexagonal)
```
[Test / API / CLI]  →  [Use Case]  →  [Port interface]
                                            ↑
                                    [Adapter: real impl]
```

### Basic CQRS
```
Commands → Write side → Domain → Events
Queries  → Read side  → Projections (optimized for reading)
```

### Event-driven (when there are multiple consumers)
```
Producer → Event Bus → Consumer A
                    → Consumer B
                    → Consumer C
```

---

## Checklist before implementing

- [ ] Do we fully understand the problem?
- [ ] Is there an ADR for the non-obvious decisions?
- [ ] Do the layers have clear responsibilities?
- [ ] Is the domain free of external dependencies?
- [ ] Can the tests run without external infra?
- [ ] Is the solution the simplest one that solves the problem?
