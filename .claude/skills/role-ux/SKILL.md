---
name: role-ux
description: User experience and product flows - discovery questions and jobs-to-be-done, information architecture, happy path plus empty/error/loading states, Nielsen usability heuristics, micro-interactions, onboarding, and deliverables per phase. Use when mapping a user flow, defining screen structure or navigation, designing empty/error states or onboarding, or auditing usability.
---

# Skill: UX Design

## Base principle

Design starts with the user's problem, not with the screen. Before proposing any flow, understand: who uses it, when, with what goal, and what alternative do they have today?

---

## Discovery — mandatory questions before designing

```
Who is the user? (role, context, technical level)
What is the job-to-be-done? (not "I want to do X" but "so that I can Y")
How often does this task happen?
What happens if they can't complete it?
What do they do today to solve this?
```

Don't start wireframes until you have clear answers.

---

## Information architecture

### Content hierarchy
```
1. What can the user do here? (primary action)
2. What do they need to know to decide? (supporting information)
3. What can they explore next? (secondary)
```

### One screen, one primary action
Every screen has ONE clear primary action. If there are two equally important actions, you have an architecture problem, not a design problem.

---

## User flows

### Map the happy path first
```
Entry → Action 1 → Action 2 → ... → Success state
```

### Then the error states and edge cases
- What happens if the user has no data yet? (empty state)
- What happens if the action fails? (error state)
- What happens if it takes longer than expected? (loading state)
- What happens if the user leaves halfway through? (interruption)

### Signs of a broken flow
- The user needs to go "back" to complete a task
- There are more than 3 screens for a simple task
- The user asks "so what do I do now?"

---

## Usability principles (Nielsen)

| Principle | What it means in practice |
|-----------|------------------------------|
| Visibility of system status | The system always shows what's happening (loading, success, error) |
| Match with the real world | Use the user's vocabulary, not the system's |
| Control and freedom | There is always an "undo" or "cancel" |
| Consistency | The same element always does the same thing |
| Error prevention | Better to prevent than to show error messages |
| Recognition > Recall | The user shouldn't have to memorize things |
| Flexibility | Shortcuts for advanced users without complicating it for beginners |
| Minimalist aesthetic | Nothing that doesn't contribute directly to the goal |
| Error recovery | Clear messages + a concrete solution |
| Help and documentation | If it needs an explanation, the design failed first |

---

## Empty states — one of the most overlooked

```
❌ Generic empty state: "No data"
✅ Useful empty state:
   - Contextual illustration (not decorative)
   - Explain WHY it's empty
   - A CTA so it stops being empty
   - An example of how it would look with data
```

---

## Micro-interactions

Transitions and immediate feedback reduce cognitive load:

- **Action feedback**: the button reacts to the click (visual + timing)
- **Progress**: if it takes >1s, show progress; if it takes >3s, give an estimate
- **Success confirmation**: the user knows it worked without reading text
- **In-context errors**: the error appears where it happened, not in a generic alert

---

## Onboarding

```
❌ An 8-step tour with tooltips on everything
✅ Effective onboarding:
   - Show value before asking for effort
   - One action at a time
   - The user learns by doing, not by reading
   - Skip always available
   - The user can revisit the onboarding later
```

---

## Deliverables per phase

| Phase | Deliverable | Suggested tool |
|------|-----------|---------------------|
| Discovery | User journey map, JTBD | Miro, FigJam |
| Architecture | Sitemap, screen flows | Whimsical, FigJam |
| Exploration | Low-fidelity wireframes | Figma (unstyled) |
| Validation | Clickable prototype | Figma prototype |
| Specification | Annotations + states | Figma + Dev Mode |
