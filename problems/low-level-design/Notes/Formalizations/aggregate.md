In **Domain-Driven Design (DDD)**, an **Aggregate** is the formalism for **defining a consistency boundary around domain state and the operations that preserve its invariants**.

A very useful mental translation:

> **Object = unit of data encapsulation**
> **Aggregate = unit of consistency and transactional correctness**
> **Bounded Context = unit of language and model ownership**

---

# 1. Formal definition

An aggregate is a tuple:

[
A = (E, V, I, T)
]

where:

* **E (Entities)**: objects with identity and lifecycle
* **V (Value Objects)**: immutable descriptive objects without identity
* **I (Invariants)**: rules that must always hold true
* **T (Transitions)**: operations that transform state while preserving invariants

The aggregate exposes only:

[
T: State \rightarrow State'
]

such that:

[
I(State) \implies I(State')
]

Meaning: if the aggregate is valid before an operation, all public operations guarantee it remains valid afterward.

This is the heart of DDD aggregates.

---

# 2. Aggregate Root

One entity is chosen as the **Aggregate Root**.

It acts as the gatekeeper.

```
Outside world
      |
      v
+----------------+
| Order          |  <- Aggregate Root
|----------------|
| addItem()      |
| removeItem()   |
| submit()       |
+----------------+
       |
       |
+--------------+
| OrderItem    |
+--------------+
       |
+--------------+
| Address      |
+--------------+
```

External code can only hold references to the root.

Good:

```python
order.add_item(product, qty)
```

Bad:

```python
order.items[0].quantity = -10
```

because that bypasses invariant checks.

---

# 3. How to spot an Aggregate boundary

The key question is:

> "What state must change atomically to preserve a business rule?"

Anything that requires a single transaction belongs together.

---

## Example: E-commerce Order

Business rules:

```
Order total = sum(items)
Order cannot have duplicate products
Submitted orders cannot be modified
```

These invariants involve:

```
Order
 |
 +-- OrderItems
 |
 +-- ShippingAddress
```

Therefore:

```
Aggregate: Order
```

### Root

```
Order(id)
```

### Entities

```
OrderItem(id)
```

because it has lifecycle (can be modified or removed).

### Value Objects

```
Money
Address
```

because identity does not matter.

---

Code:

```python
class Order:
    def __init__(self):
        self.items = []
        self.status = "DRAFT"

    def add_item(self, product_id, qty):
        if self.status == "SUBMITTED":
            raise Exception("Cannot modify")

        if any(x.product_id == product_id for x in self.items):
            raise Exception("Duplicate product")

        self.items.append(OrderItem(product_id, qty))

    def submit(self):
        if not self.items:
            raise Exception("Empty order")

        self.status = "SUBMITTED"
```

The `Order` guarantees the invariants.

---

# 4. What should NOT be inside the aggregate?

A common mistake is making aggregates too big.

Consider:

```
Customer
 |
 + Orders
      |
      + Items
```

Should `Customer` own `Orders`?

Usually **no**.

Why?

Because there is no invariant:

```
Customer + all Orders
```

that must be atomically consistent.

Instead:

```
Customer Aggregate

Customer(id)
 name
 email


Order Aggregate

Order(id)
 customer_id
 items
 status
```

The relationship is by ID:

```python
order.customer_id
```

not by object reference.

---

# 5. The Aggregate boundary algorithm

For every domain object ask:

## Step 1 — Find invariants

Examples:

```
Account balance cannot go negative
Reservation seats cannot exceed capacity
Order total equals item totals
```

---

## Step 2 — Determine the minimum state required

Example:

```
Seat count <= Capacity
```

requires:

```
Event
Seats
```

So:

```
Event Aggregate
    |
    + Seats
```

---

## Step 3 — Ask if it must be transactional

If a rule must hold **immediately after every command**:

```
use same aggregate
```

If eventual consistency is acceptable:

```
separate aggregates
```

---

# 6. Aggregate versus Database Tables

A very common confusion:

```
Aggregate != Table
```

Example:

Database:

```
orders
order_items
addresses
```

DDD:

```
Order Aggregate
```

Three tables can be one aggregate.

Conversely:

```
customers
orders
```

might be two aggregates.

---

# 7. Aggregate vs Class Design

| Concept         | Protects            | Boundary         |
| --------------- | ------------------- | ---------------- |
| Class           | Encapsulation       | Object           |
| Module          | Code organization   | Namespace        |
| Aggregate       | Business invariants | Transaction      |
| Bounded Context | Model meaning       | System ownership |
| Microservice    | Deployment/runtime  | Process/network  |

---

# 8. Control theory interpretation

You often like mapping software to formal systems, and aggregates map surprisingly well.

An aggregate is a **controlled state machine**:

[
x_{t+1} = f(x_t, u_t)
]

where:

* (x) = aggregate state
* (u) = commands (`addItem`, `submit`)
* (f) = domain logic
* invariants define the admissible state space

```
Commands
    |
    v
+--------------+
| Aggregate    |
| State Machine|
+--------------+
    |
    v
Valid states only
```

The aggregate root is the **controller interface** that restricts possible transitions.

---

# 9. Compression Cheat Sheet

| Ask                                               | If YES           | If NO               |
| ------------------------------------------------- | ---------------- | ------------------- |
| Does this data participate in the same invariant? | Same aggregate   | Separate            |
| Must changes be atomic?                           | Same transaction | Separate aggregates |
| Does external code need direct mutation?          | Probably wrong   | Use root methods    |
| Does identity matter?                             | Entity           | Value Object        |
| Is the object a gateway for consistency?          | Aggregate Root   | Internal object     |

---

The deepest way to think about aggregates is:

> **An aggregate is the smallest state machine that can enforce a set of business invariants under a transaction.**

This viewpoint connects DDD directly to automata, state transition systems, and control theory, which is why it is a powerful formalism for LLD.
