---
name: lang-python
description: Python 3.12+ standards - uv for dependencies, mandatory type hints, hexagonal layer structure with ABC ports and use cases, pytest naming, ruff/mypy, and Pydantic vs dataclass guidance. Use when writing or reviewing Python code, .py files, pyproject.toml, FastAPI routes, or pytest suites.
---

# Skill: Python

## Project setup

```bash
uv init <name>
uv add <dep>          # never pip directly
uv add --dev pytest ruff mypy
```

## Mandatory conventions

- Python 3.12+
- `from __future__ import annotations` in every file
- Type hints everywhere: params, returns, class variables
- `X | None` instead of `Optional[X]`
- `X | Y` instead of `Union[X, Y]`

## Layer structure

```
src/
  domain/
    entities/        # pure dataclasses, no external deps
    ports/           # ABCs with contracts (interfaces)
    value_objects/   # immutable types with validation
  application/
    use_cases/       # orchestrate the domain
    services/        # reusable services
  infrastructure/
    adapters/        # concrete port implementations
    db/              # SQLAlchemy, repos
    http/            # external clients
  api/
    routes/          # FastAPI routers
    schemas/         # Pydantic I/O (separate from the domain)
tests/
  unit/
  integration/
```

## Key patterns

### Port (domain interface)
```python
from abc import ABC, abstractmethod
from domain.entities.user import User, UserId

class UserRepository(ABC):
    @abstractmethod
    async def find_by_id(self, user_id: UserId) -> User | None: ...

    @abstractmethod
    async def save(self, user: User) -> None: ...
```

### Domain entity
```python
from __future__ import annotations
from dataclasses import dataclass, field
from uuid import UUID, uuid4

@dataclass
class User:
    id: UserId
    name: str
    email: str

    @classmethod
    def create(cls, name: str, email: str) -> User:
        return cls(id=UserId(uuid4()), name=name, email=email)

@dataclass(frozen=True)
class UserId:
    value: UUID
```

### Use Case
```python
from __future__ import annotations
from domain.ports.user_repository import UserRepository
from domain.entities.user import User, UserId

class CreateUserUseCase:
    def __init__(self, repo: UserRepository) -> None:
        self._repo = repo

    async def execute(self, name: str, email: str) -> UserId:
        user = User.create(name, email)
        await self._repo.save(user)
        return user.id
```

### Manual DI (no framework)
```python
# main.py or app factory
repo = PostgresUserRepository(session)
use_case = CreateUserUseCase(repo)
```

## Testing

```bash
pytest tests/           # everything
pytest tests/unit/      # unit tests only
pytest -v -k "test_create_user"
```

Naming: `test_<what>_when_<condition>_then_<result>`

```python
def test_create_user_when_email_exists_then_raises_duplicate():
    repo = FakeUserRepository(existing_email="a@b.com")
    use_case = CreateUserUseCase(repo)
    with pytest.raises(DuplicateEmailError):
        await use_case.execute("Juan", "a@b.com")
```

- Unit: mock ports with fakes/stubs, test use cases
- Integration: adapters against real infra (Docker / testcontainers)

## Linting and types

```bash
ruff check src/         # linting
ruff format src/        # formatting
mypy src/               # types
```

## When to use Pydantic vs dataclass

| Case | Use |
|------|------|
| Pure domain entity | `@dataclass` |
| Value object with validation | `pydantic.BaseModel` (frozen) |
| API schema (I/O) | `pydantic.BaseModel` |
| App config | `pydantic-settings` |

## Common architecture decisions in Python

For these choices, apply the decision protocol from CLAUDE.md:
- **ORM:** SQLAlchemy vs SQLModel vs raw queries
- **HTTP:** FastAPI vs Flask vs aiohttp
- **Async:** native asyncio vs blocking sync
- **Tests:** pytest-mock vs hand-written fakes
- **Validation:** Pydantic v2 vs attrs vs plain dataclasses
