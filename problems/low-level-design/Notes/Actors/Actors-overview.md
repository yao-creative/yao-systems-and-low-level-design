A clean way to make this precise is to treat Low Level Design as a **control-flow + ownership model over a runtime object graph**, not as “classes and methods”.

So the real question becomes:

> **Who is the agent that initiates control, who holds state invariants, and who is authorized to mutate what?**

That decomposes into two core abstractions:

---

# 1. Actors in LLD = Control Originators of State Transitions

An **actor** is anything that can *initiate a state transition* in the system.

You can classify them causally:

### (A) External Actors (system boundary injectors)

These originate from outside the program:

* User
* API caller
* Timer / scheduler
* Message queue consumer trigger
* OS / hardware event

**Role:** initiate *input events*

They do NOT own business logic.

---

### (B) Orchestration Actors (control-plane objects)

These coordinate multiple domain objects:

* Controllers (in MVC)
* Services (application layer)
* Use-cases / Interactors (Clean Architecture)
* Job workers

**Role:** decide *what sequence of domain operations occurs*

They own **workflow**, not **truth**

---

### (C) Domain Actors (state + invariants owners)

These are the “business truth holders”:

* Entities
* Aggregates (DDD)
* Value objects (immutable)

**Role:** enforce invariants and local consistency

They are the only things that should *authoritatively mutate their own state*.

---

### (D) Infrastructure Actors (effect executors)

* DB repositories
* HTTP clients
* file systems
* caches

**Role:** side effects only
They should NOT decide business logic.

---

# 2. Ownership of Methods = Ownership of State Transitions

A method is not just “inside a class”. In LLD it is:

> a **legal transition function on a state space owned by an actor**

So the key rule becomes:

## Rule 1: “State lives where invariants are enforced”

If a class owns a state invariant, it owns the methods that mutate it.

Example:

* `BankAccount.balance`

  * invariant: balance ≥ 0
  * therefore: `deposit()`, `withdraw()` MUST live here

Not in a service.

---

## Rule 2: “Orchestration owns sequencing, not mutation”

Bad design:

```text
AccountService.withdraw(account, amount)
```

Good design:

```text
account.withdraw(amount)
```

Service becomes:

```text
service.transfer(a, b, x):
    a.withdraw(x)
    b.deposit(x)
```

So:

* Service owns **control flow**
* Entity owns **state transitions**

---

## Rule 3: “Method ownership follows causality, not convenience”

Ask:

> What causes this change?

Then assign method ownership to the closest causal stabilizer:

| Change type           | Owner            |
| --------------------- | ---------------- |
| business invariant    | entity           |
| cross-entity workflow | service/use-case |
| external integration  | infrastructure   |

---

# 3. A Formal Mental Model (Useful for interviews)

You can model LLD as a directed system:

### State space:

* ( S ): set of all valid system states

### Actors:

Each actor is a partial function:

* ( f_i: S \rightarrow S )

But with constraints:

* Entities: local constrained transitions
* Services: composition of transitions
* Infrastructure: stochastic / external transition sources

So:

> LLD design = assignment of transition functions to ownership domains such that invariants are preserved locally.

---

# 4. Ownership Hierarchy (Key Insight)

Think of a funnel:

```
External Actor
     ↓
Application / Service (coordination)
     ↓
Domain Entity (truth + invariants)
     ↓
Infrastructure (effect)
```

Ownership is strictly inverted:

* top layer: *decides*
* middle: *coordinates*
* bottom domain: *validates truth*
* infrastructure: *executes side effects*

---

# 5. Common Failure Modes (important for intuition)

### Failure 1: Anemic domain model

All logic in services

Symptom:

* entities are just data bags

Cause:

* ownership collapsed upward into orchestration layer

---

### Failure 2: God entity

Entities orchestrate global workflows

Symptom:

* entity knows too much about other entities

Cause:

* ownership collapsed downward

---

### Failure 3: Mixed ownership methods

Same method both:

* enforces invariant AND
* calls external APIs

This violates separation of transition types.

---

# 6. Interview Execution Heuristic (compressed decision flow)

When designing:

### Step 1: Identify actors

* Who triggers this?
* Who coordinates?
* Who owns truth?
* Who executes side effects?

---

### Step 2: Assign state ownership

For each state variable:

* where is invariant enforced?

---

### Step 3: Assign method placement

For each operation:

* does it:

  * mutate one entity → entity method
  * coordinate multiple entities → service
  * external effect → infrastructure

---

### Step 4: Validate invariance locality

Ask:

> Can invalid state exist inside the object at any point?

If yes → wrong ownership.

---

# 7. One-line core principle

> Methods belong to the layer that is closest to the **smallest closed system where the invariant can be guaranteed without external knowledge**.

---

If you want, I can map this directly onto classic patterns (MVC, Clean Architecture, DDD aggregates, CQRS) and show exactly how “ownership of methods” shifts across them.
