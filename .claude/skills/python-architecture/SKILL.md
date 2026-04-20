# Skill: Python Architecture

## Cuándo aplica
Siempre que se diseñe o implemente código Python en este proyecto.

## Estructura de proyecto (Clean/Hexagonal)

```
src/
  domain/
    entities/        # dataclasses o Pydantic models (puros, sin deps externas)
    ports/           # interfaces (ABC) que definen contratos
    value_objects/   # tipos inmutables con validación
  application/
    use_cases/       # orquestan dominio, no conocen infraestructura
    services/        # servicios de aplicación reutilizables
  infrastructure/
    adapters/        # implementaciones concretas de los ports
    db/              # SQLAlchemy, repos concretos
    http/            # clientes HTTP, APIs externas
  api/
    routes/          # FastAPI routers
    schemas/         # Pydantic I/O schemas (distintos del dominio)
```

## Reglas de dependencia

- `domain` no importa NADA externo al dominio
- `application` importa `domain`, nunca `infrastructure`
- `infrastructure` implementa ports de `domain`
- `api` importa `application` y `infrastructure` para DI

## Patrones clave

### Port (interfaz)
```python
from abc import ABC, abstractmethod

class UserRepository(ABC):
    @abstractmethod
    async def find_by_id(self, user_id: UserId) -> User | None: ...
```

### Use Case
```python
class CreateUserUseCase:
    def __init__(self, repo: UserRepository, events: EventBus) -> None:
        self._repo = repo
        self._events = events

    async def execute(self, cmd: CreateUserCommand) -> UserId:
        user = User.create(cmd.name, cmd.email)
        await self._repo.save(user)
        await self._events.publish(UserCreated(user.id))
        return user.id
```

### Dependency Injection (sin framework)
```python
# En main.py o app factory
repo = PostgresUserRepository(session)
use_case = CreateUserUseCase(repo, event_bus)
```

## Convenciones Python

- Type hints en todo: params, returns, variables de clase
- `from __future__ import annotations` en cada archivo
- Dataclasses para value objects simples, Pydantic para validación
- `raise ValueError` en dominio, excepciones de dominio custom para casos de negocio
- No `Optional[X]` → usar `X | None`
- No `Union[X, Y]` → usar `X | Y`

## Testing

```python
# Unit: mockear ports, testear use cases
# Integration: testear adapters contra infra real (Docker)
# Naming: test_<what>_when_<condition>_then_<expected>

def test_create_user_when_email_exists_then_raises_duplicate_error():
    ...
```

## ADR — cuándo crear uno

Crear `docs/adr/ADR-XXX.md` ante:
- Elección de framework/librería
- Decisión de estructura de DB
- Pattern de comunicación entre capas
- Cualquier tradeoff no obvio
