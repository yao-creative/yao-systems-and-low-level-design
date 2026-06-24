Start from the *control objective*, then derive the structure.

## 1. Intent (what event sourcing is solving)

At its core, the system goal is:

> Maintain a **consistent evolving state under concurrent writes**, while preserving **full historical reconstructability**.

This decomposes into three constraints:

1. **Correctness constraint**: current state must be derivable deterministically.
2. **Audit constraint**: every mutation must be explainable post hoc.
3. **Scalability constraint**: reads and writes must decouple under load.

Traditional CRUD stores optimize for (1), partially support (3), and weaken (2). Event sourcing flips that.

---

## 2. Formal model (minimal abstraction)

We model a system as:

* State space: ( S )
* Event space: ( E )
* Transition function:
  [
  f: S \times E \rightarrow S
  ]

But event sourcing removes direct state mutation as primary storage.

### Core invariant

Instead of storing ( S_t ), we store an ordered log:

[
\mathcal{L} = [e_1, e_2, ..., e_n]
]

and define:

[
S_n = f(f(...f(S_0, e_1), e_2)..., e_n)
]

So:

> State is a **materialized projection of a pure fold over an event stream**.

This makes state *derivative*, not primary.

---

## 3. System decomposition (causal pipeline view)

Event sourcing systems factor into a 4-stage pipeline:

### (A) Command layer (intent)

Input: user action / request
Output: validated intent

[
c \rightarrow \text{validation}(c)
]

### (B) Decision layer (domain logic)

Transforms command → event(s)

[
c \rightarrow e
]

Constraint: **no state mutation here**, only event emission.

---

### (C) Event log (source of truth)

Append-only structure:

* total order (or partial with causality metadata)
* immutable entries

[
\mathcal{L} := \mathcal{L} \cup {e}
]

This is the *system of record*.

---

### (D) Projection layer (materialization)

Derived views:

[
S^{(i)} = \sum_{e \in \mathcal{L}} f_i(e)
]

Each projection is:

* a different interpretation of same event stream
* independently recomputable

Examples:

* user profile view
* recommendation counters
* leaderboard state

---

## 4. Compression perspective (why it works)

Event sourcing is fundamentally a **lossless compression scheme over state evolution**.

### Naive storage:

Store full state snapshots:
[
S_1, S_2, ..., S_n
]

Redundancy: high (repeated structure)

---

### Event sourcing storage:

Store deltas:
[
e_t = S_t - S_{t-1}
]

But more generally:

* not numeric delta
* semantic delta (“UserLikedItem”, “BalanceDebited”)

So compression is:

> Replace redundant full-state encoding with minimal sufficient causal differences.

This yields:

* **temporal compression** (only changes stored)
* **structural compression** (domain-level events vs raw fields)
* **reconstruction ability retained**

---

## 5. Key tradeoff structure

### Gains

* Full auditability (perfect lineage)
* Time travel queries (replay)
* Natural streaming integration
* Horizontal write scalability (append-only log)
* Decoupled read models

### Costs

* Replay complexity (O(n) reconstruction unless optimized)
* Event schema evolution problem
* Event log growth
* Eventually consistent projections
* Higher conceptual overhead

---

## 6. Applications (mapped by system pressure)

### 6.1 High audit / compliance systems

* banking transactions
* trading systems
* medical records

Why:

* need immutable history
* reconstruct any state at time ( t )

---

### 6.2 Collaborative / concurrent systems

* document editors
* multiplayer games

Why:

* concurrent events must be ordered and replayable
* conflict resolution is event-level

---

### 6.3 Recommendation / analytics pipelines

* user interaction tracking
* feature generation systems

Why:

* events become training signals
* projections become feature stores

Key pattern:
[
\text{event stream} \rightarrow \text{feature projections} \rightarrow \text{model inputs}
]

---

### 6.4 Distributed system coordination (CDC-like systems)

* replication systems
* microservice synchronization

Why:

* log acts as single source of truth for propagation

---

## 7. Relationship to neighboring abstractions

### vs CRUD

* CRUD: state is primitive
* Event sourcing: state is derived

### vs CQRS

* CQRS splits:

  * write model (event generation)
  * read model (projections)
    Event sourcing is often the **write backbone** of CQRS.

### vs WAL (Write-Ahead Log)

* WAL is *internal durability mechanism*
* Event log is *externalized domain truth*

Key distinction:

> WAL is infrastructure; event log is semantic history.

---

## 8. Compression summary (one-line formalization)

Event sourcing is:

> A system design that replaces state storage with a **lossless causal log**, where state is a **reducible projection over event sequences**, enabling decoupled computation of time-indexed system behavior.

---

If you want next step, I can map this directly into:

* distributed system design (Kafka-style log architecture)
* or LeetCode-style problems that are isomorphic (replay, dedup, idempotency, ordering guarantees)
