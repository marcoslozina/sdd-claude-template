---
name: role-requirements
description: Turning vague ideas into testable requirements - Given/When/Then acceptance criteria, Example Mapping, Definition of Ready/Done, minimal PRDs, and non-functional requirements. Use when scoping a new feature, requirements are ambiguous, acceptance criteria are missing, or during the SDD explore/spec/verify phases.
---

# Skill: Requirements Engineering

## Role
Turn vague ideas into testable requirements before writing a single line of code.
A badly written requirement is more expensive than no requirement at all — it produces code that passes the tests
but does not solve the real problem.

## When to activate this skill
- The user describes a new feature
- Requirements are ambiguous, contradictory, or assume implicit knowledge
- There are no clear acceptance criteria
- The team argues about whether something is "done" or not
- Exploration or Spec phase in the SDD flow

---

## Golden rule

```
A requirement is good if two independent people read it
and reach the same conclusion about what to build.
If not — rewrite it.
```

---

## Anatomy of a good requirement

```
❌ "The user can filter products"

✅ GIVEN an authenticated user with at least 1 product in the catalog
   WHEN they apply the filter category = "electronics" and max price = $500
   THEN they see only the products that meet both conditions
   AND the header count reflects the filtered quantity
   AND if there are no results, they see the empty state with a CTA to clear filters
```

**Checklist for a testable requirement:**
- [ ] Has a clear subject (who?)
- [ ] Has a specific action (what exactly does it do?)
- [ ] Has an entry condition (from which state?)
- [ ] Has an observable, verifiable outcome
- [ ] Includes the error case or edge case

---

## Example Mapping — discovering requirements with the team

Example Mapping is a 30-45 minute technique to explore a feature before estimating it.
It uses 4 types of cards:

```
🟡 STORY       → "The user can pay by card"
🔵 RULE        → "Only Visa and Mastercard are accepted"
🟢 EXAMPLE     → "Juan pays $500 with Visa → receives confirmation"
🟢 EXAMPLE     → "Juan tries to pay with Amex → sees a specific error"
🔴 QUESTION    → "What happens if the bank declines for insufficient funds?"
```

**Protocol:**
1. Write the story on a yellow card
2. For each business rule → a blue card
3. For each rule → at least 1 happy example + 1 error example (green cards)
4. Doubts that come up → red cards (to be resolved BEFORE estimating)

**Sign the feature is ready to estimate:** few red cards.
**Sign it is NOT ready:** many red cards → more questions than answers.

---

## Definition of Ready (DoR)

A requirement does not enter the sprint until it meets:

- [ ] Acceptance criteria written in Given/When/Then format
- [ ] Edge cases identified (what happens if it fails? if it's empty? if there are permissions involved?)
- [ ] External dependencies identified (APIs, services, teams)
- [ ] UI design agreed (if applicable)
- [ ] No blocking open questions
- [ ] Estimation possible without big assumptions

---

## Definition of Done (DoD)

A requirement is "done" when:

- [ ] Acceptance criteria verified manually
- [ ] Automated tests covering the happy path and edge cases
- [ ] Code review approved
- [ ] No regressions detected in existing functionality
- [ ] Documentation updated if a public API changes
- [ ] Deployed to the staging environment

---

## Minimum viable PRD

For features requiring more than 3 days of work, document before implementing:

```markdown
## Feature: [name]

### Problem it solves
[1 paragraph — what pain the user has today]

### Affected users
[Who benefits and who could be negatively affected]

### Proposed solution
[What we build — not the how, the what]

### Success criteria
- Metric 1: [what we measure and what the target is]
- Metric 2: ...

### Out of scope (explicit)
- [What this version does NOT include]

### Functional requirements
1. GIVEN ... WHEN ... THEN ...
2. ...

### Non-functional requirements
- Performance: [e.g.: response < 200ms at p95]
- Security: [e.g.: admin role users only]
- Availability: [e.g.: no downtime on deploy]

### Risks
- [Known technical or business risk]
```

---

## Non-functional requirements — the ones always forgotten

| Category | Questions to ask |
|-----------|-----------------|
| **Performance** | How many concurrent users? Acceptable latency at p95? Is there an SLO? |
| **Security** | Who can access it? Which data is sensitive? Is auditing required? |
| **Availability** | Can there be downtime on deploy? What happens if an external service fails? |
| **Scalability** | Could volume grow 10x in 6 months? Are there predictable peaks? |
| **Observability** | Which metrics do we need to know it works in production? |
| **Internationalization** | Multiple languages? Time zones? Date/currency formats? |

---

## Signs of problematic requirements

| Sign | Problem | Action |
|-------|---------|--------|
| "The system should be fast" | Not testable | Define a concrete metric: "< 200ms p95" |
| "The user can manage X" | Ambiguous | Break it down: create, edit, delete, list — that's 4 requirements |
| "The way we've always done it" | Implicit assumption | Document the behavior explicitly |
| "Obviously it also does Y" | Implicit scope creep | Write it down or exclude it — never assume |
| "Same as the previous system" | Hidden debt | Map the previous system before assuming parity |

---

## Integration with SDD

In the SDD flow, this skill activates in:

- **Phase 1 (Exploration):** identify unknowns and open questions
- **Phase 3 (Spec):** write acceptance criteria in Given/When/Then
- **Phase 6 (Verification):** validate that the implementation meets the criteria

Before starting Phase 3, run Example Mapping with the user to clear out the red cards.
