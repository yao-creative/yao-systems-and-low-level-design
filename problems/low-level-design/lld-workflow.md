# LLD Problem Solving Funnel (Compressed)

The entire workflow compresses to:

> **Model constrained state evolution under controlled dependency structure.**

Or operationally:

```text id="2kvrp5"
Requirements
    ↓
Invariants
    ↓
State Machines
    ↓
Ownership
    ↓
Responsibilities
    ↓
Interfaces
    ↓
Data Structures
    ↓
Concurrency / Failures
    ↓
Extensibility
```

---

# 1. Universal LLD Funnel

| Step                   | Core Question                      | Output               |
| ---------------------- | ---------------------------------- | -------------------- |
| 1. Semantics           | What is the system actually doing? | Core domain          |
| 2. Invariants          | What must NEVER break?             | Constraints          |
| 3. States              | What changes over time?            | FSM/workflow         |
| 4. Ownership           | Who controls mutation?             | Authority boundaries |
| 5. Responsibilities    | Which component owns which logic?  | Cohesion             |
| 6. Dependencies        | What depends on what?              | Architecture DAG     |
| 7. Interfaces          | What contracts exist?              | Abstractions         |
| 8. Data Structures     | What operations dominate?          | Performance          |
| 9. Concurrency/Failure | What races/fails?                  | Robustness           |
| 10. Extensibility      | What changes frequently?           | Stable evolution     |

---

# 2. The Core Mental Compression

Most candidates think:

```text id="mjlwmz"
classes → methods → patterns
```

Strong engineers think:

```text id="0r5drs"
constraints → states → ownership → dependencies
```

Objects are secondary.

---

# 3. The Real Order of Thinking

---

## STEP 1 — Extract Semantics

Ask:

```text id="m4l9lj"
What problem is being constrained?
```

Examples:

| Problem       | Actual Semantics                |
| ------------- | ------------------------------- |
| Parking lot   | constrained allocation          |
| Elevator      | scheduling                      |
| ATM           | transactional state transitions |
| Food delivery | workflow orchestration          |
| Cache         | bounded retrieval optimization  |
| Rate limiter  | flow control                    |

---

# STEP 2 — Find Invariants (Highest ROI)

Ask:

```text id="c4o8r9"
What states are illegal?
```

Examples:

* no double booking
* one active ticket per vehicle
* order cannot be both cancelled and delivered
* cache capacity bounded

This determines architecture more than classes do.

---

# STEP 3 — Draw State Machine

Ask:

```text id="3bkn3j"
What transitions are legal?
```

Example:

```text id="u06gv2"
CREATED -> PAID -> SHIPPED
         -> CANCELLED
```

This reveals:

* workflows
* edge cases
* retries
* invalid transitions

---

# STEP 4 — Define Ownership

Ask:

```text id="j8j4m0"
Who is allowed to mutate state?
```

Bad:

```text id="ijv0c0"
everyone mutates Order
```

Good:

```text id="38j6ew"
OrderService owns transitions
```

This prevents:

* coupling
* race conditions
* invariant leaks

---

# STEP 5 — Allocate Responsibilities

Ask:

```text id="5eql6v"
Which logic naturally changes together?
```

Goal:

* high cohesion
* low coupling

Avoid:

* god objects
* anemic models

---

# STEP 6 — Shape Dependency Graph

Ask:

```text id="qsg7sl"
What should remain stable?
```

Rule:

```text id="qz4i2m"
Business logic
    ↓
Interfaces
    ↓
Implementations
```

NOT:

```text id="5axj5i"
Business logic
    ↓
MySQL / Redis / Stripe directly
```

---

# STEP 7 — Introduce Interfaces

Only after responsibilities stabilize.

Use interfaces when:

* implementations vary,
* mocking/testing needed,
* external systems involved.

Avoid premature abstraction.

---

# STEP 8 — Add Data Structures

Ask:

```text id="r6jv4j"
What operations dominate runtime?
```

Examples:

| Problem      | DS                   |
| ------------ | -------------------- |
| scheduler    | heap                 |
| autocomplete | trie                 |
| cache        | hashmap + DLL        |
| matching     | ordered map          |
| rate limiter | sliding window queue |

---

# STEP 9 — Model Concurrency & Failures

Ask:

```text id="fjlwm4"
What can race, retry, fail, duplicate?
```

Think:

* idempotency
* retries
* locking
* ordering
* queues
* backpressure

---

# STEP 10 — Extensibility Pass

ONLY at the end.

Ask:

```text id="hv2jq6"
What future changes are likely?
```

Examples:

* more notification channels
* new payment providers
* additional workflow states

Now patterns emerge naturally.

---

# 4. Pattern Emergence Rule

Never begin with patterns.

Patterns emerge from forces.

| Force                    | Natural Pattern |
| ------------------------ | --------------- |
| interchangeable behavior | Strategy        |
| layered functionality    | Decorator       |
| event propagation        | Observer        |
| workflow transitions     | State           |
| coordinated interactions | Mediator        |
| complex construction     | Builder         |

---

# 5. Universal LLD Heuristic

Separate:

| Concern       | Why                   |
| ------------- | --------------------- |
| state         | correctness           |
| transitions   | workflow clarity      |
| policy        | stable business rules |
| side effects  | external systems      |
| persistence   | storage concerns      |
| orchestration | coordination logic    |

Most bad designs mix all 6.

---

# 6. Fast Interview Workflow (Practical)

Use this exact sequence:

---

## 1. Clarify requirements

* scale
* assumptions
* edge cases

---

## 2. Define invariants

This impresses interviewers heavily.

---

## 3. Draw state machine

Do this early.

---

## 4. Identify core entities

Only major ones.

---

## 5. Assign responsibilities

Avoid god objects.

---

## 6. Define interfaces

Only where variation exists.

---

## 7. Add DS + concurrency

Performance + robustness.

---

## 8. Discuss extensibility

Future-proofing.

---

# 7. Deep Compression (Most Important)

LLD reduces to 4 things:

| Core Concept | Mathematical Compression |
| ------------ | ------------------------ |
| invariants   | constraint system        |
| workflows    | state machine            |
| dependencies | DAG/partial order        |
| ownership    | authority graph          |

Everything else is implementation detail.

---

# 8. One-Line Mental Model

When solving LLD:

> **Find the constrained state machine, isolate mutation ownership, and minimize dependency entropy.**
