---
name: role-orchestrator
description: Agent-team orchestration: delegate-first rules, sync task vs async delegate, sub-agent prompt structure, Engram context injection and topic keys, SDD phase parallelization, and task dependency analysis. Use when coordinating sub-agents, running SDD phases, or deciding what to parallelize and what to keep sequential.
---

# Skill: Orchestrator — Agent Team Architecture

## Role
The orchestrator does NOT do the work. It coordinates, delegates, and synthesizes.
All real work (reading code, writing code, analysis) goes to sub-agents.

---

## Fundamental principle

```
Orchestrator = minimal context + coordination
Sub-agents   = isolated context + real work

Benefits:
  ⏱  Time:     parallel phases cut wall clock by up to 50%
  🪙  Tokens:   each sub-agent processes only what it needs
  🧠  Memory:   Engram persists across agents and sessions
  🔒  Isolated: a failing agent doesn't pollute the main context
```

---

## Orchestrator rules

| Rule | Description |
|-------|-------------|
| No inline work | Reading/writing code → always a sub-agent |
| Delegate-first | Prefer `delegate` (async) over `task` (sync) |
| Parallel by default | If two phases don't block each other → launch them together |
| Context injection | The orchestrator searches Engram and passes context to the sub-agent |
| Write to Engram | Sub-agents save discoveries before finishing |

**Anti-patterns — never do these:**
- Reading code files "to understand" → delegate
- Writing code directly → delegate
- Doing a "quick" inline analysis → delegate
- Passing full artifact content between agents → pass the Engram ID

---

## Delegation flow

### Delegate (async — default)
```
Orchestrator: "I'm going to launch spec and design in parallel"
    → delegate: sdd-spec  (runs in the background)
    → delegate: sdd-design (runs in the background)
    → [wait for both]
    → synthesize results
    → show to the user
```

### Task (sync — only if you need the result before continuing)
```
Orchestrator: "I need the exploration before proposing"
    → task: sdd-explore
    → [wait for the result]
    → use the result to build the proposal
```

---

## Context protocol for sub-agents

### What the orchestrator does BEFORE launching a sub-agent

```
1. Search for relevant context in Engram:
   mem_search(query: "topic keywords", project: "project-name")

2. If there's a result → mem_get_observation(id) for the full content

3. Include in the sub-agent prompt:
   - The exact path of the skill to load
   - Relevant prior artifacts (Engram topic keys, not the content)
   - An explicit instruction to save discoveries to Engram
```

### Sub-agent prompt structure

```markdown
SKILL: Read `.claude/skills/{name}/SKILL.md` before starting.

CONTEXT (from previous sessions):
  - Approved proposal: topic key `sdd/{change}/proposal` in Engram
  - Chosen stack: Python + FastAPI + PostgreSQL

TASK:
  [specific description of what it has to do]

EXPECTED OUTPUT:
  [what format, what artifact to produce]

MEMORY:
  If you make important discoveries or decisions, or find bugs,
  save them to Engram with mem_save before finishing.
  project: "{project-name}"
```

---

## SDD phases — what runs in parallel

```
sdd-explore         → sync (you need the result before proposing)
       ↓
sdd-propose         → sync (you need the proposal for spec and design)
       ↓
sdd-spec  ──────────┐
                    ├── PARALLEL (independent of each other)
sdd-design ─────────┘
       ↓
sdd-tasks           → sync (needs spec + design)
       ↓
sdd-apply task-1 ───┐
sdd-apply task-2 ───┤
sdd-apply task-3 ───┼── PARALLEL (if the tasks are independent)
sdd-apply task-4 ───┘
       ↓
sdd-verify          → sync (verifies everything together)
```

### When NOT to parallelize apply

Tasks that block each other do NOT go in parallel:
```
❌ Wrong parallelization:
   task-1: create the users table
   task-2: add a foreign key that depends on users
   → task-2 fails if task-1 hasn't finished

✅ Sequential:
   task-1 → task-2

✅ Correct parallelization:
   task-A: implement UserRepository
   task-B: implement ProductRepository
   → independent, they go together
```

---

## Engram — topic keys per project

```
sdd-init/{project}               → initial project context
sdd/{change}/explore             → exploration artifact
sdd/{change}/proposal            → chosen proposal
sdd/{change}/spec                → specification
sdd/{change}/design              → technical design
sdd/{change}/tasks               → task list
sdd/{change}/apply-progress      → implementation progress
sdd/{change}/verify-report       → verification report
```

Retrieving an artifact:
```
1. mem_search(query: "sdd/{change}/spec")  → get the ID
2. mem_get_observation(id: {id})           → full content
```

---

## Estimated savings from parallelization

| Phase | Without parallelism | With parallelism |
|------|----------------|----------------|
| explore → propose | 2 min | 2 min (sync) |
| spec + design | 4 min | 2 min (parallel) |
| tasks | 1 min | 1 min (sync) |
| apply (4 tasks) | 8 min | 2-3 min (parallel) |
| verify | 1 min | 1 min (sync) |
| **Total** | **16 min** | **8-9 min** |

The real savings depend on complexity. On large projects the difference is bigger.

---

## Full orchestration example

```
User: "I need to implement JWT authentication"

Orchestrator:
  1. mem_search("jwt auth {project}") → nothing prior

  2. [task] sdd-explore "JWT authentication in this project"
     → result: system context, stack, existing patterns

  3. Presents a proposal (2-3 options) → user picks option B

  4. [delegate] sdd-spec  "JWT auth per proposal B"
     [delegate] sdd-design "JWT auth per proposal B"
     → both in parallel, both write to Engram

  5. Waits for both → presents a summary → user confirms

  6. [task] sdd-tasks "JWT auth"
     → list of 5 tasks

  7. User confirms → analyze dependencies:
     task-1 (User entity) → independent
     task-2 (JWT utils)   → independent
     task-3 (middleware)  → depends on task-1 and task-2
     task-4 (routes)      → depends on task-3
     task-5 (tests)       → depends on all of them

  8. [delegate] sdd-apply task-1
     [delegate] sdd-apply task-2
     → parallel

  9. Wait → [task] sdd-apply task-3 → [task] sdd-apply task-4

  10. [task] sdd-verify → final report → present to the user
```

---

## Agent architecture decisions

Apply the decision protocol from CLAUDE.md when facing:
- **Sync vs async:** do I need the result before continuing?
- **Task granularity:** very small tasks = agent overhead; very large ones = not parallelizable
- **What to save in Engram:** decisions, bugs, discoveries — not ephemeral state
- **How many agents in parallel:** more than 4-5 at once can be counterproductive
