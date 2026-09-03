---
name: role-solid
description: SOLID principles, clean code, and design patterns with cross-language examples - DRY/KISS/YAGNI, code smells and their refactorings, creational/structural/behavioral patterns, complexity thresholds, and a code review checklist. Use when reviewing or refactoring classes and modules, untangling duplicated logic, or when tests are hard to write.
---

# Skill: SOLID, Clean Code and Design Patterns

## Role
Write code that communicates intent, withstands change, and needs no comments to be understood.
Clean code is not code that works — it is code the next developer can change without fear.

## When to activate this skill
- Code review of any class or module
- A class is found to have more than one reason to change
- There is duplicated logic across modules
- A change in one place breaks another unrelated place
- Tests are hard to write (a sign of bad design)

---

## SOLID — principles with real examples

### S — Single Responsibility
A class has a single reason to change.

```python
# ❌ Violates SRP: processes, validates AND sends
class OrderService:
    def process(self, order):
        if order.total < 0: raise ValueError()  # validation
        self.db.save(order)                       # persistence
        self.email.send(order.user, "confirmed")  # notification

# ✅ Each responsibility in its own place
class OrderValidator: ...
class OrderRepository: ...
class OrderNotifier: ...
class OrderService:
    def process(self, order):
        self.validator.validate(order)
        self.repository.save(order)
        self.notifier.notify(order)
```

### O — Open/Closed
Open for extension, closed for modification.

```typescript
// ❌ Every new discount type modifies the function
function applyDiscount(order: Order, type: string): number {
  if (type === 'percentage') return order.total * 0.9
  if (type === 'fixed') return order.total - 10
  // new type = modify this function
}

// ✅ Extension without modification
interface DiscountStrategy {
  apply(total: number): number
}
class PercentageDiscount implements DiscountStrategy { ... }
class FixedDiscount implements DiscountStrategy { ... }
// new type = new class, nothing else is touched
```

### L — Liskov Substitution
A subtype must be able to replace its supertype without breaking the program.

```java
// ❌ Violates LSP: Square breaks the Rectangle contract
class Rectangle { setWidth(w); setHeight(h); area() }
class Square extends Rectangle {
  setWidth(w) { super.setWidth(w); super.setHeight(w); } // surprise
}

// ✅ Models it correctly without forced inheritance
interface Shape { area(): double }
class Rectangle implements Shape { ... }
class Square implements Shape { ... }
```

### I — Interface Segregation
No client should depend on methods it does not use.

```go
// ❌ Fat interface: FileWriter is forced to implement Read and Seek
type Storage interface {
    Read(key string) ([]byte, error)
    Write(key string, data []byte) error
    Seek(offset int) error
    Delete(key string) error
}

// ✅ Small, focused interfaces
type Writer interface { Write(key string, data []byte) error }
type Reader interface { Read(key string) ([]byte, error) }
type Deleter interface { Delete(key string) error }
```

### D — Dependency Inversion
High-level modules do not depend on low-level modules. Both depend on abstractions.

```typescript
// ❌ Use case coupled to a concrete implementation
class CreateUserUseCase {
  constructor(private repo: PostgresUserRepository) {} // infrastructure inside the domain
}

// ✅ Depends on the abstraction (port)
interface UserRepository { save(user: User): Promise<void> }
class CreateUserUseCase {
  constructor(private repo: UserRepository) {} // the domain defines the contract
}
```

---

## DRY, KISS, YAGNI

| Principle | Rule | When it's violated |
|-----------|-------|--------------|
| **DRY** — Don't Repeat Yourself | Logic has a single authoritative representation | Copy-paste of logic, not of structure |
| **KISS** — Keep It Simple | The simplest solution that works | Over-engineering, premature abstractions |
| **YAGNI** — You Aren't Gonna Need It | Don't implement what you don't need now | "Just in case", "in the future" |

> DRY is not "don't repeat code" — it is "don't repeat KNOWLEDGE". Two functions with similar code but different logic are NOT a DRY violation.

---

## Code smells — signs of bad design

| Smell | Symptom | Refactor |
|-------|---------|----------|
| **God Object** | Class with 500+ lines and 20 methods | Split by responsibility |
| **Feature Envy** | A method that uses more data from another class than its own | Move the method to that class |
| **Data Clump** | A group of variables that always travel together | Extract a Value Object |
| **Primitive Obsession** | `String email`, `String phone`, `int age` with no types | Create semantic types |
| **Long Parameter List** | Function with 5+ parameters | Extract a configuration object |
| **Shotgun Surgery** | One change touches 10 files | Consolidate responsibility |
| **Divergent Change** | One class frequently changes for different reasons | Split into two classes |
| **Dead Code** | Commented-out code, methods never called | Delete without mercy |

---

## Design Patterns — when to use them

### Creational
| Pattern | When | Problem it solves |
|---------|--------|----------------------|
| **Factory Method** | Create objects without specifying the exact class | Decouple creation from use |
| **Builder** | Complex object with many optional parameters | Avoid constructors with 8 args |
| **Singleton** | A single global instance (with caution) | Config, logger, connection pool |

### Structural
| Pattern | When | Problem it solves |
|---------|--------|----------------------|
| **Adapter** | Integrate an incompatible interface | Wrap an external API for the domain |
| **Decorator** | Add behavior without modifying the class | Transparent logging, caching, retry |
| **Facade** | Simplify a complex subsystem | Public API over internal logic |

### Behavioral
| Pattern | When | Problem it solves |
|---------|--------|----------------------|
| **Strategy** | Interchangeable algorithms | Remove type-based if/else |
| **Observer** | Notify changes without coupling | Domain events |
| **Command** | Encapsulate operations as objects | Undo/redo, work queues |
| **Chain of Responsibility** | Processing pipeline | Middlewares, chained validations |

---

## Complexity metrics

| Metric | Healthy threshold | Action if exceeded |
|---------|-----------------|------------------|
| **Cyclomatic complexity** | ≤ 10 per function | Extract functions |
| **Lines per function** | ≤ 20 | Split responsibilities |
| **Lines per class** | ≤ 200 | Review SRP |
| **Parameters per function** | ≤ 3 | Extract an object |
| **Nesting depth** | ≤ 3 levels | Early return / extract |

---

## Code review checklist

- [ ] Does each class have a single reason to change?
- [ ] Are the interfaces small and focused?
- [ ] Do the use cases depend on abstractions, not implementations?
- [ ] Is there duplicated logic that should be extracted?
- [ ] Does each class/function name communicate exactly what it does?
- [ ] Is there commented-out code or unused methods?
- [ ] Could the primitive parameters be Value Objects?
- [ ] Are the tests easy to write? (if not, the design is the problem)
