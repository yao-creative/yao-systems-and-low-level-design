High-level system design (HLD) is basically LLD “lifted up one categorical level”: instead of deciding **how objects and methods implement transitions**, you decide **how whole subsystems coordinate state, events, and data under scale constraints**.

A good compression is:

> LLD = *local state-transition mechanics*
> HLD = *global state distribution + coordination + consistency model*

---

# 1. First-principles decomposition of HLD

Any large system reduces to 4 coupled questions:

## (1) State

Where does truth live?

* DB (Postgres, MySQL)
* log (Kafka)
* cache (Redis)
* distributed replicas

---

## (2) Flow of changes (write path)

How does state change propagate?

* synchronous RPC
* async events
* pipelines (ETL / stream)

---

## (3) Read model (query path)

How do we observe state?

* direct DB query
* cached views
* materialized views
* precomputed aggregates

---

## (4) Consistency model

What guarantees exist between (2) and (3)?

* strong consistency
* eventual consistency
* causal consistency
* no guarantee (best effort)

---

# 2. HLD is NOT “bigger LLD”

This is the key mistake people make.

LLD answers:

> “How do I implement a state machine correctly?”

HLD answers:

> “How do I distribute, replicate, and evolve many state machines under latency/failure constraints?”

So the abstraction jumps from:

* single transition system
  to
* **network of interacting transition systems**

---

# 3. The unified formal model for HLD

You can model most systems as:

[
\text{System} = (\mathcal{N}, \mathcal{E}, \mathcal{S}, \mathcal{F})
]

Where:

| Component     | Meaning                             |
| ------------- | ----------------------------------- |
| (\mathcal{N}) | nodes (services)                    |
| (\mathcal{E}) | event channels (Kafka, RPC, queues) |
| (\mathcal{S}) | local state per node                |
| (\mathcal{F}) | transition functions                |

Each node is itself:

[
S_i \times E_i \to S_i
]

So HLD = **composition of many LTSs over a communication graph**

---

# 4. Core HLD architectures (the real canonical set)

There are only ~5 fundamental ones.

---

## (1) Request-Response Systems (synchronous graph)

### Model

[
Client \to Service \to DB
]

### Properties

* blocking flow
* tight coupling
* strong consistency easier

### Examples

* REST APIs
* monolith backend
* CRUD services

### Control style

→ controller-driven (central orchestration)

---

## (2) Event-Driven Systems (async state propagation)

### Model

[
Service \to EventBus \to Consumers
]

### Properties

* decoupled writes/reads
* eventual consistency
* replayable logs possible

### Examples

* Kafka systems
* Stripe event pipelines
* microservice architectures

### Control style

→ dispatcher-driven (reactive)

---

## (3) Stream Processing Systems (continuous dataflow)

### Model

[
Stream \to Transform \to Stream
]

### Properties

* infinite data
* windowed state
* incremental computation

### Examples

* Flink
* Spark streaming
* Kafka Streams

### Control style

→ dataflow graph

---

## (4) Distributed State Systems (replication-first)

### Model

[
S_1 \leftrightarrow S_2 \leftrightarrow S_3
]

### Properties

* replication
* partition tolerance
* consensus required

### Examples

* databases
* Raft systems
* distributed caches

### Control style

→ consensus + synchronization protocols

---

## (5) Actor Systems (isolated concurrent nodes)

### Model

[
Actor(S) + Msg \to Actor(S')
]

### Properties

* no shared state
* message passing only
* natural concurrency

### Examples

* Erlang
* Akka
* Orleans

### Control style

→ message-driven autonomy

---

# 5. The real missing unification (important insight)

All HLD systems are combinations of:

## A. State placement

Where is truth stored?

* centralized (DB)
* distributed (replicas)
* ephemeral (cache)
* derived (views)

---

## B. State propagation

How does change move?

* push (events)
* pull (queries)
* hybrid (CQRS)

---

## C. Coordination model

Who ensures correctness?

* controller (central service)
* dispatcher (event routing)
* consensus (Raft/Paxos)
* actor isolation

---

## D. Time model

How does system evolve?

* synchronous steps
* async events
* streaming continuous updates

---

# 6. Mapping HLD ↔ LLD (important bridge)

| LLD concept    | HLD equivalent              |
| -------------- | --------------------------- |
| state machine  | service                     |
| transition     | API call / event            |
| controller     | orchestrator service        |
| dispatcher     | message bus                 |
| scheduler      | job system / queue consumer |
| semaphore/lock | consistency constraint      |

---

# 7. Why HLD feels harder than LLD

Because LLD is:

> local correctness problem

HLD is:

> **distributed consistency under partial failure**

So you now add:

* latency
* partitioning
* replication lag
* idempotency
* retries
* ordering guarantees

---

# 8. Core HLD design choice (the single most important axis)

Almost every system design question reduces to:

## Push vs Pull

### Push (event-driven)

* changes broadcast outward
* Kafka, pub/sub

### Pull (query-driven)

* systems fetch state when needed
* REST APIs, DB reads

### Hybrid (modern default)

* CQRS
* event sourcing + materialized views

---

# 9. The hidden duality (this connects to your earlier questions)

HLD is governed by:

## Two dual views of truth

### (1) State-centric

> “What is the current value?”

### (2) Event-centric

> “What happened to produce this value?”

So:

| View   | System type          |
| ------ | -------------------- |
| state  | databases, caches    |
| events | Kafka, logs, streams |

---

# 10. One-line compression

High-level system design is:

> the study of **how multiple interacting state-transition systems share, propagate, and agree on state under constraints of time, failure, and scale**

---

# 11. Practical mental model (interview usable)

When given any system, classify:

### Step 1: What is the unit of state?

* user, order, session, device

### Step 2: Where does state live?

* DB, service, log, memory

### Step 3: How does it change?

* sync call / async event / stream

### Step 4: How is consistency enforced?

* locking / consensus / eventual / none

### Step 5: How is it observed?

* query / cache / projection

---

If you want next step, I can:

* map **Uber / WhatsApp / Stripe / YouTube** into these 5 HLD primitives
* or connect HLD → LLD formally (how each HLD architecture induces a different LLD pattern set)
* or build a **decision tree for system design interviews (what architecture to choose from constraints)**
