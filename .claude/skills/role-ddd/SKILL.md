---
name: role-ddd
description: Domain-Driven Design, tactical and strategic — ubiquitous language, Event Storming workshops, entities, value objects, aggregates, domain events and services, repositories as ports, bounded contexts and context maps, plus DDD anti-patterns. Use when modeling a business domain, naming entities, deciding aggregate boundaries, splitting a system into contexts or services, or fixing an anemic domain model.
---

# Skill: Domain-Driven Design (DDD)

## Role
Build software whose model reflects the real business, not the database or the framework.
The code must speak the language of the business — if a developer and a domain expert
read the code and don't understand the same thing, the model is wrong.

## When to activate this skill
- The system has non-trivial business logic
- Multiple teams are working on the same system
- The business terms and the code differ ("order" in the business = "transaction" in the code)
- The system grows and changes in one area break unrelated areas
- Exploration or Architecture Proposal phase in SDD

---

## Ubiquitous Language

The business vocabulary and the code must be identical. No translations.

```python
# ❌ Technical language disconnected from the business
class Transaction:
    def execute(self): ...          # what does "execute" mean?
    def update_status(self): ...    # which status? why does it change?

# ✅ Domain language
class Order:
    def place(self): ...            # the business "places" an order
    def confirm(self): ...          # the business "confirms" when there's stock
    def cancel(self, reason): ...   # the business "cancels" with a reason
    def fulfill(self): ...          # the business "fulfills" when it's ready
```

**How to build the ubiquitous language:**
1. Meet with the domain expert (not just the PM — the person who knows the business)
2. Listen to the terms they use naturally
3. Document the glossary — one definition per term, no ambiguity
4. If two people use the same term for different things → there are two concepts, name them differently

---

## Event Storming — discovering the domain

Event Storming is a 2-4 hour workshop to map a complete domain with everyone involved.

### Card types (standard colors)

```
🟠 DOMAIN EVENT     → "Order confirmed", "Payment processed", "Stock depleted"
🔵 COMMAND          → "Confirm order", "Process payment", "Reserve stock"
🟡 AGGREGATE        → "Order", "Payment", "Inventory"
🟣 POLICY           → "When payment fails → notify the user"
🔴 HOTSPOT          → Doubts, conflicts, risk zones
🟢 EXTERNAL SYSTEM  → "Payment gateway", "Email service"
```

### Event Storming protocol

**Phase 1 — Creative chaos (20 min)**
Everyone writes Domain Events on orange stickies. No order, no discussion. Events only.
Mandatory format: **past participle** — "Order CONFIRMED", not "Confirm order".

**Phase 2 — Order the timeline (20 min)**
Stick the events in chronological order on a wall/miro board.
Identify duplicates and conflicts.

**Phase 3 — Commands and Aggregates (30 min)**
For each event, which command caused it? Which entity processes it?

**Phase 4 — Policies (20 min)**
What automatic reactions are there? "When X happens → do Y"

**Phase 5 — Hotspots (10 min)**
Mark in red everything that generates discussion, doubt, or risk.

**Result:** a visual map of the domain that everyone understands — business and tech.

---

## DDD Building Blocks

### Entity
Has its own identity that persists over time. Two entities with the same data are different if they have different IDs.

```python
@dataclass
class Order:
    id: OrderId          # identity
    customer_id: CustomerId
    items: list[OrderItem]
    status: OrderStatus

    def confirm(self) -> None:
        if self.status != OrderStatus.PENDING:
            raise OrderNotConfirmableError(self.id)
        self.status = OrderStatus.CONFIRMED
        self._events.append(OrderConfirmed(order_id=self.id))
```

### Value Object
No identity — defined by its attributes. Immutable. Two Value Objects with the same data ARE the same object.

```python
@dataclass(frozen=True)  # immutable
class Money:
    amount: Decimal
    currency: str

    def __post_init__(self):
        if self.amount < 0:
            raise ValueError("Money cannot be negative")

    def add(self, other: "Money") -> "Money":
        if self.currency != other.currency:
            raise CurrencyMismatchError()
        return Money(self.amount + other.amount, self.currency)

# ✅ Money(100, "USD") == Money(100, "USD") → True
# ✅ It has no ID — it is the value, not an entity
```

### Aggregate
A cluster of entities and value objects treated as a unit. It has a root (Aggregate Root) that is the only entry point.

```python
class Order:  # Aggregate Root
    def add_item(self, product_id: ProductId, quantity: int) -> None:
        # The business logic lives here, not in the controller
        if self.status != OrderStatus.DRAFT:
            raise OrderNotEditableError()
        item = OrderItem(product_id=product_id, quantity=quantity)
        self.items.append(item)

    def total(self) -> Money:
        return sum(item.subtotal() for item in self.items)
```

**Aggregate rules:**
- Only modify state through the Aggregate Root
- Do not reference another aggregate's internal entities — only by ID
- Keep the aggregate small — if it grows a lot, it's probably two aggregates

### Domain Event
Something significant that happened in the domain. In the past tense. Immutable.

```python
@dataclass(frozen=True)
class OrderConfirmed:
    order_id: OrderId
    customer_id: CustomerId
    total: Money
    confirmed_at: datetime
```

### Domain Service
Business logic that doesn't belong to any specific entity.

```python
class PricingService:
    def calculate_discount(self, order: Order, customer: Customer) -> Money:
        # Logic that involves Order AND Customer — it doesn't belong to either alone
        if customer.is_premium() and order.total() > Money(1000, "USD"):
            return order.total() * Decimal("0.1")
        return Money(0, "USD")
```

### Repository (Port)
An abstraction to persist and retrieve aggregates. The domain defines the interface.

```python
class OrderRepository(Protocol):
    def find_by_id(self, order_id: OrderId) -> Order | None: ...
    def save(self, order: Order) -> None: ...
    def find_pending_by_customer(self, customer_id: CustomerId) -> list[Order]: ...
```

---

## Bounded Contexts

A large system has multiple models, each valid within its own context.
The same concept can mean different things in different contexts.

```
"Product" in Catalog:    name, description, images, SEO
"Product" in Inventory:  SKU, stock, warehouse location
"Product" in Billing:    price, taxes, tax code

They are the SAME product in the business, but DIFFERENT MODELS in the software.
Don't force a single model — create one per context.
```

### Context Map — how the contexts relate

```
[Catalog]  ──── Published Language ────▶ [Inventory]
[Orders]   ──── Anti-Corruption Layer ──▶ [External payment]
[Orders]   ◀─── Conformist ─────────────  [Logistics]
```

| Relationship | When | What it implies |
|----------|--------|-------------|
| **Shared Kernel** | Small team, tightly coupled contexts | Shared model — coordination is mandatory |
| **Customer/Supplier** | One context depends on the other | The supplier adapts to the customer's needs |
| **Anti-Corruption Layer** | Integration with an external or legacy system | Translate the external model into the internal one — never let it contaminate |
| **Published Language** | Public API consumed by many | Stable, versioned contract |

---

## Integration with Clean Architecture

```
Domain Layer:
  entities/          → Order, Customer, Product (Aggregates + Entities)
  value_objects/     → Money, OrderId, Email
  events/            → OrderConfirmed, PaymentFailed
  services/          → PricingService (Domain Service)
  repositories/      → OrderRepository (Port/Interface)

Application Layer:
  use_cases/         → PlaceOrderUseCase, ConfirmOrderUseCase

Infrastructure Layer:
  repositories/      → PostgresOrderRepository (Adapter)
  events/            → KafkaEventPublisher
```

---

## DDD anti-patterns

| Anti-pattern | Symptom | Fix |
|-------------|---------|-----|
| **Anemic Domain Model** | Entities with only getters/setters, logic in services | Move the logic into the aggregate |
| **Fat Service** | One service holding all the business logic | Distribute it across aggregates and domain services |
| **Shared Database** | Two bounded contexts read the same table | Separate schemas or eventually databases |
| **God Aggregate** | One aggregate with 20 entities inside | Split it — it's probably 3 aggregates |
| **Primitive Obsession** | `String orderId`, `int price` instead of Value Objects | Create semantic types |

---

## DDD checklist

- [ ] Does the code use the same terms as the business?
- [ ] Is there a documented domain glossary?
- [ ] Does the business logic live in the aggregate, not in the controller or service?
- [ ] Are Value Objects immutable and validated on construction?
- [ ] Are aggregates modified only through the root?
- [ ] Are repositories interfaces in the domain and implementations in infra?
- [ ] Are Domain Events in the past tense and immutable?
- [ ] Are the bounded contexts identified and their boundaries clear?
- [ ] Do external integrations go through an Anti-Corruption Layer?
