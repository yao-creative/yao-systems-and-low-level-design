# OOP Decision Matrix (Compressed)

Use OOP concepts to decide where state lives, who may change it, and how variation enters the system without spreading conditionals everywhere.

ROI scoring: `5 = learn early, appears everywhere`, `3 = useful but situational`, `1 = know the idea, low interview frequency`.

| Situation | Symptom | OOP Concept | Use It For | Avoid When | Compression | ROI to Learn |
| --- | --- | --- | --- | --- | --- | --- |
| Data and behavior belong together | Service layer knows every field | Encapsulation | Keep invariants next to state | Object is just a DTO crossing boundaries | State + guardrails | 5 |
| Multiple concrete variants share a contract | Callers branch on type | Abstraction | Program against capability, not implementation | There is only one stable implementation | Stable boundary | 5 |
| Variants differ mainly by behavior | Deep inheritance tree forming | Polymorphism | Swap behavior behind one interface | Differences are only data, not behavior | One call, many implementations | 5 |
| One object is changing for many reasons | God class | Single Responsibility | Group logic that changes together | Splitting creates meaningless wrappers | Cohesive ownership | 5 |
| High-level policy depends on low-level details | Business logic imports concrete infra | Dependency Inversion | Point dependencies inward to contracts | The concrete dependency is local and trivial | Stable dependency direction | 5 |
| Related objects collaborate around one state | Rules scattered across helpers | Composition | Build behavior from collaborating objects | Objects have no meaningful boundaries | Object graph over monolith | 5 |
| Construction requires valid combinations | Half-built objects appear | Constructor discipline / Builder | Enforce valid creation | Object is trivial to construct | Valid-by-construction | 4 |
| New variants appear often | Repeated edits to old code | Open/Closed | Add new behavior additively | Change is mostly in one fixed flow | Extension seam | 4 |
| Logic should happen automatically on state change | Callers forget required side effects | Tell, don't ask | Put behavior on the owner object | Workflow spans many bounded contexts | Push intent to owner | 4 |
| External users need a smaller surface | Too many methods exposed | Interface segregation | Expose role-specific APIs | It fragments one coherent API | Narrow contract | 3 |
| Shared base class is forcing fake methods | Subclasses violate expectations | Liskov Substitution | Preserve substitutability | Hierarchy exists only for code reuse | Behavioral compatibility | 3 |
| Variants reuse common state/behavior | Duplicate fields and methods | Inheritance | Share template behavior and identity | Relationship is not truly "is-a" | Reuse by hierarchy | 2 |

---

# Common OOP Tradeoffs

| Decision | Prefer This When | Watch For |
| --- | --- | --- |
| Inheritance vs composition | Inheritance only when the subtype really preserves base semantics | Fragile base classes, deep trees |
| Interface vs concrete class | Interface when multiple implementations or a stable boundary matter | Premature abstraction |
| Rich model vs anemic model | Rich model when invariants and workflows live with the entity | Bloated entities doing orchestration |
| Mutable vs immutable object | Immutable for value objects and shared read-heavy state | Excess copy cost in hot mutable workflows |
| Public setters vs intent methods | Intent methods when state transitions have rules | Verbose APIs if object is truly just data |

## Highest-ROI OOP Topics First

1. Encapsulation
2. Abstraction
3. Polymorphism
4. Composition
5. Single Responsibility
6. Dependency Inversion

---

# Neutral Examples

## Encapsulation

Instead of letting callers set `bankAccount.balance` directly, expose `deposit(amount)` and `withdraw(amount)` so the object can reject negative amounts or overdrafts.

## Abstraction

A report exporter can depend on an `Exporter` contract with `export(data)` while concrete implementations handle PDF or CSV formatting.

## Inheritance

A `Document` base type might hold `id`, `owner`, and `createdAt`, while `SpreadsheetDocument` and `TextDocument` extend it with specific behavior.

## Polymorphism

A notification service can call `channel.send(message)` without branching on whether the channel is email, SMS, or push.

## Composition

A `CheckoutService` can be composed from `PricingPolicy`, `TaxCalculator`, and `PaymentProcessor` instead of inheriting one giant superclass.

## Dependency Inversion

A billing workflow should depend on a `PaymentGateway` interface, not directly on one vendor SDK, so tests and vendor swaps stay localized.

## Rich Domain Model

A `Reservation` object can own `confirm()`, `cancel()`, and `expire()` so invalid transitions are rejected close to the state it protects.
