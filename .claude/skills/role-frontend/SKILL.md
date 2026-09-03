---
name: role-frontend
description: React 18 + strict TypeScript frontend standards: folder structure, container/presentational split, state selection (useState, Zustand, TanStack Query), typed props, memoization, and a11y rules. Use when writing or reviewing .tsx/.jsx components, hooks, stores, or choosing a frontend state/routing/styling library.
---

# Skill: Frontend

## Base stack
React 18+ with strict TypeScript. No `any`. No `// @ts-ignore`.

---

## Project structure

```
src/
  components/
    ui/              # atoms: Button, Input, Badge (no business logic)
    features/        # per-feature organisms: UserCard, CheckoutForm
    layouts/         # page shells: DashboardLayout, AuthLayout
  pages/             # route entry points (minimal logic)
  hooks/             # reusable custom hooks
  stores/            # global state (Zustand / Redux Toolkit)
  services/          # API calls (no UI logic)
  types/             # shared types and interfaces
  utils/             # pure functions with no side effects
```

---

## Component principles

### Container / Presentational
```tsx
// ✅ Presentational: receives data, doesn't fetch it
const UserCard = ({ name, email, onEdit }: UserCardProps) => (
  <div>
    <h2>{name}</h2>
    <p>{email}</p>
    <button onClick={onEdit}>Edit</button>
  </div>
)

// ✅ Container: fetches data, delegates rendering
const UserCardContainer = ({ userId }: { userId: string }) => {
  const { user, isLoading } = useUser(userId)
  if (isLoading) return <Skeleton />
  return <UserCard {...user} onEdit={() => navigate(`/users/${userId}/edit`)} />
}
```

### Single responsibility rule
- A component does one thing
- If the name contains "And" → split it (`UserFormAndValidation` → two components)
- Max props: ~5-7. More than that → extract an object or split the component

---

## State — when to use what

| State type | Solution |
|---------------|---------|
| Local UI (toggle, form) | `useState` |
| Derived state | `useMemo` / compute during render |
| Side effects | `useEffect` (with explicit dependencies) |
| Global UI state | Zustand |
| Server state | TanStack Query / SWR |
| Complex forms | React Hook Form |

**Rule:** state lives at the lowest level that needs it.

---

## TypeScript in components

```tsx
// ✅ Explicitly typed props
interface ButtonProps {
  label: string
  variant: 'primary' | 'secondary' | 'danger'
  isLoading?: boolean
  onClick: () => void
}

// ✅ Generics in reusable hooks
function useLocalStorage<T>(key: string, initialValue: T): [T, (value: T) => void]

// ✅ Discriminated unions for states
type AsyncState<T> =
  | { status: 'idle' }
  | { status: 'loading' }
  | { status: 'success'; data: T }
  | { status: 'error'; error: Error }
```

---

## Performance

```tsx
// Memoize expensive components
const ExpensiveList = memo(({ items }: { items: Item[] }) => ...)

// Memoize expensive computations
const sorted = useMemo(() => items.sort(compareFn), [items])

// Stabilize callbacks
const handleClick = useCallback(() => doSomething(id), [id])

// Route-level code splitting
const Dashboard = lazy(() => import('./pages/Dashboard'))
```

**When NOT to memoize:** simple components, cheap renders. Memoization has a cost.

---

## Accessibility (a11y) — non-negotiable

- Buttons with `aria-label` when they have no visible text
- Images with descriptive `alt` (or `alt=""` if decorative)
- Forms with a `<label>` associated to every `<input>`
- Visible focus (no `outline: none` without a replacement)
- Color never as the only differentiator of information

---

## Common Frontend architecture decisions

Apply the decision protocol (this project's CLAUDE.md if it defines one, otherwise dev-harness's docs/DECISION_PROTOCOL.md) when facing:
- **Global state:** Zustand vs Redux Toolkit vs Context API
- **Data fetching:** TanStack Query vs SWR vs manual fetch
- **Routing:** React Router vs TanStack Router
- **Styling:** Tailwind vs CSS Modules vs styled-components
- **Forms:** React Hook Form vs Formik vs manual controlled
- **Testing:** React Testing Library vs Playwright vs Cypress
