---
name: lang-typescript
description: TypeScript standards for frontend and backend - strict tsconfig, no any, discriminated unions and branded types, Clean Architecture layout for Node, Zod validation, typed React components and hooks, Vitest/Jest tests. Use when writing or reviewing .ts/.tsx code, tsconfig.json, React components, or Node/Bun services.
---

# Skill: TypeScript

## Applies to
Frontend (React) and Backend (Node.js / Bun). The conventions are the same.

## Base configuration

```json
// tsconfig.json
{
  "compilerOptions": {
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "exactOptionalPropertyTypes": true,
    "target": "ES2022",
    "moduleResolution": "bundler"
  }
}
```

No `any`. No `// @ts-ignore`. No blind `as`.

---

## Type conventions

```typescript
// ✅ Explicit types on public functions
function findUser(id: UserId): Promise<User | null>

// ✅ Discriminated unions for states
type Result<T, E = Error> =
  | { ok: true; value: T }
  | { ok: false; error: E }

// ✅ Branded types for IDs
type UserId = string & { readonly _brand: 'UserId' }
function UserId(raw: string): UserId { return raw as UserId }

// ✅ Readonly for domain objects
type User = Readonly<{
  id: UserId
  name: string
  email: string
}>

// ❌ Never
const user: any = getUser()
function process(data: object): any
```

---

## Node.js Backend (Clean Architecture)

```
src/
  domain/
    entities/        # business types and interfaces
    ports/           # repository and service interfaces
  application/
    use-cases/       # application logic
  infrastructure/
    adapters/        # concrete implementations
    db/              # Prisma / Drizzle
    http/            # external clients
  api/
    routes/          # Fastify / Express routers
    schemas/         # Zod for input validation
tests/
```

### Port (interface)
```typescript
// domain/ports/user-repository.ts
export interface UserRepository {
  findById(id: UserId): Promise<User | null>
  save(user: User): Promise<void>
}
```

### Use Case
```typescript
// application/use-cases/create-user.ts
export class CreateUserUseCase {
  constructor(private readonly repo: UserRepository) {}

  async execute(input: CreateUserInput): Promise<UserId> {
    const user = User.create(input.name, input.email)
    await this.repo.save(user)
    return user.id
  }
}
```

### Validation with Zod
```typescript
import { z } from 'zod'

const CreateUserSchema = z.object({
  name: z.string().min(1).max(100),
  email: z.string().email(),
})

type CreateUserInput = z.infer<typeof CreateUserSchema>
```

---

## React Frontend

### Typed component
```tsx
interface ButtonProps {
  label: string
  variant: 'primary' | 'secondary' | 'danger'
  isLoading?: boolean
  onClick: () => void
}

export const Button = ({ label, variant, isLoading = false, onClick }: ButtonProps) => (
  <button
    className={variants[variant]}
    disabled={isLoading}
    onClick={onClick}
  >
    {isLoading ? <Spinner /> : label}
  </button>
)
```

### Typed custom hook
```typescript
function useAsync<T>(fn: () => Promise<T>): AsyncState<T> {
  const [state, setState] = useState<AsyncState<T>>({ status: 'idle' })

  const run = useCallback(async () => {
    setState({ status: 'loading' })
    try {
      const value = await fn()
      setState({ status: 'success', value })
    } catch (error) {
      setState({ status: 'error', error: error as Error })
    }
  }, [fn])

  return { ...state, run }
}
```

---

## Testing

```typescript
// Vitest (frontend) / Jest (backend)
// Naming: describe what it does, not how
describe('CreateUserUseCase', () => {
  it('returns user id when input is valid', async () => {
    const repo = new FakeUserRepository()
    const useCase = new CreateUserUseCase(repo)
    const id = await useCase.execute({ name: 'Ana', email: 'ana@test.com' })
    expect(repo.findById(id)).resolves.not.toBeNull()
  })

  it('throws DuplicateEmailError when email already exists', async () => {
    const repo = new FakeUserRepository({ existingEmail: 'ana@test.com' })
    const useCase = new CreateUserUseCase(repo)
    await expect(
      useCase.execute({ name: 'Ana', email: 'ana@test.com' })
    ).rejects.toThrow(DuplicateEmailError)
  })
})
```

---

## Common architecture decisions in TypeScript

Apply the decision protocol (this project's CLAUDE.md if it defines one, otherwise dev-harness's docs/DECISION_PROTOCOL.md) when facing:
- **Runtime:** Node.js vs Bun vs Deno
- **HTTP framework:** Fastify vs Express vs Hono
- **ORM/Query:** Prisma vs Drizzle vs Kysely vs raw SQL
- **Validation:** Zod vs Valibot vs TypeBox
- **Bundler:** Vite vs esbuild vs tsup
- **Test:** Vitest vs Jest
