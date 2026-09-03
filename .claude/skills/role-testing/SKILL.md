---
name: role-testing
description: Testing strategy focused on behavior over implementation - test pyramid, unit tests for use cases and domain, integration tests with Testcontainers, E2E/API tests, naming conventions, test doubles (fake/stub/mock/spy), and coverage perspective. Use when writing or reviewing tests, choosing fakes vs mocks, setting up test infrastructure, or debating coverage thresholds.
---

# Skill: Testing Strategy

## Base principle
Tests verify BEHAVIOR, not implementation.
A test that passes when the code is broken is not a test — it's noise.

---

## Test pyramid

```
           /\
          /  \
         / E2E \          ← few, slow, expensive (critical flows)
        /--------\
       /Integration\      ← adapters against real infra
      /--------------\
     /   Unit Tests   \   ← the majority, fast, isolated
    /------------------\
```

**Rule:** if you have many E2E tests and few unit tests, the pyramid is inverted. That's fragile and slow.

---

## Unit Tests — use cases and domain

```python
# ✅ Test behavior, not implementation
def test_create_user_assigns_unique_id():
    repo = FakeUserRepository()
    use_case = CreateUserUseCase(repo)

    id1 = use_case.execute(name="Ana", email="ana@test.com")
    id2 = use_case.execute(name="Luis", email="luis@test.com")

    assert id1 != id2

def test_create_user_raises_when_email_already_exists():
    repo = FakeUserRepository(existing_emails={"ana@test.com"})
    use_case = CreateUserUseCase(repo)

    with pytest.raises(DuplicateEmailError):
        use_case.execute(name="Ana2", email="ana@test.com")

# ✅ Fake instead of mock when the contract is simple
class FakeUserRepository(UserRepository):
    def __init__(self, existing_emails: set[str] = None):
        self._users: dict[UserId, User] = {}
        self._existing_emails = existing_emails or set()

    async def save(self, user: User) -> None:
        if user.email in self._existing_emails:
            raise DuplicateEmailError(user.email)
        self._users[user.id] = user

    async def find_by_id(self, user_id: UserId) -> User | None:
        return self._users.get(user_id)
```

**Fakes vs Mocks:**
- **Fake:** a simplified implementation of the contract (preferred)
- **Mock:** verification of calls (only when the observable behavior IS the call)

---

## Integration Tests — adapters

```python
# Test the adapter against real infrastructure
import pytest
from testcontainers.postgres import PostgresContainer

@pytest.fixture(scope="session")
def postgres():
    with PostgresContainer("postgres:16") as pg:
        yield pg

@pytest.fixture
def repo(postgres):
    engine = create_engine(postgres.get_connection_url())
    Base.metadata.create_all(engine)
    with Session(engine) as session:
        yield PostgresUserRepository(session)

def test_save_and_find_user(repo):
    user = User.create("Ana", "ana@test.com")
    repo.save(user)
    found = repo.find_by_id(user.id)
    assert found.email == "ana@test.com"
```

**Never H2/SQLite in place of Postgres in production.** The behavior differs. Use Testcontainers.

---

## E2E / API Tests

```python
# Test the full API contract
import httpx
from fastapi.testclient import TestClient

def test_create_user_returns_201(client: TestClient):
    response = client.post("/users", json={"name": "Ana", "email": "ana@test.com"})
    assert response.status_code == 201
    assert "id" in response.json()

def test_create_user_with_invalid_email_returns_422(client: TestClient):
    response = client.post("/users", json={"name": "Ana", "email": "not-an-email"})
    assert response.status_code == 422

def test_get_nonexistent_user_returns_404(client: TestClient):
    response = client.get("/users/00000000-0000-0000-0000-000000000000")
    assert response.status_code == 404
```

---

## Naming — conventions

```
test_<what>_when_<condition>_then_<result>

test_create_user_when_email_exists_then_raises_duplicate_error
test_find_user_when_id_not_found_then_returns_none
test_post_user_when_input_invalid_then_returns_422
```

Every name should read as documentation. If it isn't clear, the test does too much.

---

## What NOT to test

- Trivial getters/setters with no logic
- The framework (FastAPI already tests that routing works)
- Third-party infrastructure (don't test that Postgres works)
- Internal implementation (a refactor should not break tests)

---

## Coverage — the right perspective

**High coverage does not equal good tests.**

- 100% coverage with trivial tests = false confidence
- 70% coverage with real behavior tests = far more value

**What you SHOULD measure:** how many bugs escape to production.

---

## Test doubles — when to use what

| Type | What it does | When |
|------|----------|--------|
| **Fake** | Simplified implementation | Port with a simple contract |
| **Stub** | Returns a fixed value | Dependency with a single relevant path |
| **Mock** | Verifies it was called | When the effect IS the call (emails, events) |
| **Spy** | Records calls without changing behavior | Debugging complex tests |

---

## Common decisions in Testing

Apply the decision protocol from CLAUDE.md when facing:
- **Fakes vs Mocks:** for each domain port
- **Testcontainers vs embedded DB:** for database adapters
- **Coverage threshold:** what minimum % to enforce in CI
- **Test isolation:** a clean database between tests vs transactions that roll back
