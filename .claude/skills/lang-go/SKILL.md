---
name: lang-go
description: Go coding standards - project layout (cmd/internal/pkg), hexagonal ports and use cases, error wrapping with %w and sentinel errors, context propagation, stdlib HTTP handlers, fake-based tests, and concurrency rules. Use when writing or reviewing Go code, .go files, go.mod dependencies, or running go test.
---

# Skill: Go

## Project setup

```bash
go mod init github.com/{user}/{project}
go get {dependency}
```

Go 1.22+. No unnecessary frameworks — the stdlib goes a long way.

---

## Mandatory conventions

- Errors are always handled: no `_` to ignore an `error`
- No `panic` in production code — only in `init()` or impossible conditions
- Small interfaces: 1-3 methods (io.Reader, io.Writer as the model)
- Packages named for what they provide, not for what they contain (`user`, not `models`)
- `context.Context` as the first parameter in every function that does I/O

---

## Project structure

```
cmd/
  api/
    main.go            # entry point — wiring only
internal/
  domain/
    user.go            # entity + business logic
    repository.go      # interface (port)
  application/
    create_user.go     # use case
  infrastructure/
    postgres/
      user_repo.go     # port implementation
    http/
      user_handler.go  # HTTP handler
pkg/                   # exportable, reusable code
config/
  config.go
```

`internal/` makes packages non-importable from outside the module. Use it for everything except what you explicitly want to be a public library.

---

## Key patterns

### Interface (Port)
```go
// internal/domain/repository.go
type UserRepository interface {
    FindByID(ctx context.Context, id UserID) (*User, error)
    Save(ctx context.Context, user *User) error
}
```

### Domain entity
```go
// internal/domain/user.go
type UserID string

type User struct {
    ID    UserID
    Name  string
    Email string
}

func NewUser(name, email string) (*User, error) {
    if name == "" {
        return nil, errors.New("name is required")
    }
    return &User{
        ID:    UserID(uuid.New().String()),
        Name:  name,
        Email: email,
    }, nil
}
```

### Use Case
```go
// internal/application/create_user.go
type CreateUserUseCase struct {
    repo domain.UserRepository
}

func NewCreateUserUseCase(repo domain.UserRepository) *CreateUserUseCase {
    return &CreateUserUseCase{repo: repo}
}

func (uc *CreateUserUseCase) Execute(ctx context.Context, name, email string) (domain.UserID, error) {
    user, err := domain.NewUser(name, email)
    if err != nil {
        return "", fmt.Errorf("creating user: %w", err)
    }
    if err := uc.repo.Save(ctx, user); err != nil {
        return "", fmt.Errorf("saving user: %w", err)
    }
    return user.ID, nil
}
```

### Error handling — conventions
```go
// ✅ Wrap with context
if err := repo.Save(ctx, user); err != nil {
    return fmt.Errorf("CreateUser: saving to db: %w", err)
}

// ✅ Sentinel domain errors
var ErrUserNotFound = errors.New("user not found")
var ErrDuplicateEmail = errors.New("email already exists")

// ✅ Check the error type
if errors.Is(err, ErrUserNotFound) {
    // handle 404
}
```

---

## HTTP with the stdlib (no framework)

```go
// internal/infrastructure/http/user_handler.go
func (h *UserHandler) CreateUser(w http.ResponseWriter, r *http.Request) {
    var input struct {
        Name  string `json:"name"`
        Email string `json:"email"`
    }
    if err := json.NewDecoder(r.Body).Decode(&input); err != nil {
        http.Error(w, "invalid input", http.StatusUnprocessableEntity)
        return
    }

    id, err := h.useCase.Execute(r.Context(), input.Name, input.Email)
    if err != nil {
        if errors.Is(err, domain.ErrDuplicateEmail) {
            http.Error(w, "email already exists", http.StatusConflict)
            return
        }
        http.Error(w, "internal error", http.StatusInternalServerError)
        return
    }

    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(http.StatusCreated)
    json.NewEncoder(w).Encode(map[string]string{"id": string(id)})
}
```

For more complex APIs: **Chi** (lightweight router, stdlib-compatible) or **Gin**.

---

## Testing

```bash
go test ./...                     # everything
go test ./internal/application/   # per package
go test -run TestCreateUser -v    # by name
go test -race ./...               # detect race conditions
```

```go
// Fake repository
type fakeUserRepo struct {
    users  map[domain.UserID]*domain.User
    emails map[string]bool
}

func (r *fakeUserRepo) Save(ctx context.Context, user *domain.User) error {
    if r.emails[user.Email] {
        return domain.ErrDuplicateEmail
    }
    r.users[user.ID] = user
    r.emails[user.Email] = true
    return nil
}

func TestCreateUser_WhenEmailExists_ReturnsError(t *testing.T) {
    repo := &fakeUserRepo{
        emails: map[string]bool{"ana@test.com": true},
    }
    uc := application.NewCreateUserUseCase(repo)

    _, err := uc.Execute(context.Background(), "Ana", "ana@test.com")

    if !errors.Is(err, domain.ErrDuplicateEmail) {
        t.Errorf("expected ErrDuplicateEmail, got %v", err)
    }
}
```

---

## Concurrency — rules

```go
// ✅ Share data through channels, not shared memory
// ✅ If you use a mutex, document what it protects
// ✅ Always pass a context for cancellation
// ✅ go test -race to catch races in CI

// ❌ Never
go func() {
    sharedMap[key] = value  // race condition without a mutex
}()
```

---

## Common Go decisions

Apply the decision protocol (this project's CLAUDE.md if it defines one, otherwise dev-harness's docs/DECISION_PROTOCOL.md) when facing:
- **HTTP framework:** stdlib + Chi vs Gin vs Echo vs Fiber
- **ORM:** GORM vs sqlc vs pgx raw
- **DI:** manual (preferred) vs Wire
- **Config:** plain env vars vs Viper vs godotenv
- **Testing:** testify vs stdlib testing
