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

# 3. Formalization Frame

If you want the hardcore theory version, treat an LLD problem as a constrained transition system:

```text id="lld-formal-system"
System S = (E, Sigma, X, I, delta, Phi, Omega, D)

E      = external events / commands
Sigma  = abstract states
X      = data representation / memory layout
I      = invariants over Sigma and X
delta  = transition relation: Sigma x E -> Sigma
Phi    = policies / guards / authorization predicates
Omega  = side effects / emitted observations
D      = dependency graph over components
```

The design task is:

1. define `Sigma` and `E` precisely,
2. constrain them with `I`,
3. make `delta` total on legal inputs and rejecting on illegal ones,
4. localize authority over `delta`,
5. embed `delta` into a low-coupling dependency graph `D`.

Every step in the funnel refines one part of this tuple.

---

# 4. The Real Order of Thinking

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

### Formalization

This step defines the problem as an abstract machine.

```text id="semantics-formal"
Domain = (Entities, Operations, Observations)
```

You are trying to identify:

* the carrier set of meaningful objects,
* the admissible operations over them,
* the observable behavior that users care about.

Formally, this is where you choose the algebra of the domain:

```text id="semantics-algebra"
For each operation op:
op : Input x State -> Output x State
```

If you get semantics wrong, every later class/interface is a faithful implementation of the wrong model.

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

### Formalization

Invariants are predicates over reachable system states:

```text id="invariant-formal"
I = { i1, i2, ..., in }, where each ik : Sigma -> {true, false}
```

Correctness requirement:

```text id="invariant-proof"
For every reachable state s, all ik(s) = true
```

Operational proof obligation:

```text id="induction"
1. Initial state s0 satisfies I
2. If I(s) and transition delta(s, e) = s', then I(s')
```

This is just inductive safety. In interviews, "good design" usually means "the mutation surface makes invariant preservation obvious."

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

### Formalization

Model workflow as a labeled transition system:

```text id="lts"
TS = (Sigma, E, ->)
where s -e-> s' iff event e legally transforms s into s'
```

Useful partitions:

* terminal states,
* transient states,
* failure states,
* compensating states.

Design questions become formal questions:

* Safety: can an illegal state be reached?
* Liveness: can the workflow get permanently stuck?
* Determinism: for a given `(s, e)`, is the next state unique?
* Idempotence: does replaying `e` preserve the same post-state?

This is why state modeling comes before classes: classes are a representation of transitions, not a substitute for them.

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

### Formalization

Ownership defines an authority relation over state variables.

```text id="ownership-formal"
owns : Component -> P(StateSlice)
mutate : Component x StateSlice -> allowed | forbidden
```

Desired property:

```text id="single-writer"
For each critical mutable slice m, there exists a minimal authority set A(m),
ideally |A(m)| = 1
```

This is the single-writer principle. Once multiple peers can mutate the same logical fact, the design must pay for consensus, locking, or conflict resolution.

In practical LLD terms:

* ownership is about who can commit transitions,
* not who can read,
* not who has a pointer/reference.

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

### Formalization

A responsibility partition is a decomposition of behavior into modules that minimizes cross-module information flow.

```text id="responsibility-formal"
Behavior set B = {b1, b2, ..., bn}
Partition P = {B1, B2, ..., Bk}
```

Heuristic objective:

```text id="cohesion-objective"
maximize  intra-module semantic affinity
minimize inter-module causal dependency
```

This is an optimization problem, not a naming exercise.

A useful test:

* if two rules change for the same business reason, keep them near each other,
* if two rules depend on different clocks, policies, or failure modes, separate them.

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

### Formalization

Dependencies should form a directed acyclic graph over stability layers.

```text id="dependency-formal"
D = (V, A)
u -> v means u depends on v
```

Desirable properties:

* acyclicity,
* inward dependency toward more stable abstractions,
* no edge from policy-rich code directly to volatile infrastructure details.

You can think of this as a partial order by volatility:

```text id="volatility-order"
if volatility(u) < volatility(v), prefer not to depend on v directly
```

Bad LLD often fails because the dependency graph destroys substitutability and test isolation before the class model is even evaluated.

# STEP 7 — Introduce Interfaces

Only after responsibilities stabilize.

Use interfaces when:

* implementations vary,
* mocking/testing needed,
* external systems involved.

Avoid premature abstraction.

---

### Formalization

An interface is a behavioral contract over an equivalence class of implementations.

```text id="interface-formal"
Interface I := set of operations + preconditions + postconditions + error semantics
```

Introduce an interface only if there exists meaningful implementation variance under the same contract:

```text id="variance-rule"
Impl1 ~= Impl2 with respect to client-observable semantics
```

Otherwise the abstraction is fake and just adds indirection.

The key theoretical point:

* interfaces encode substitutability,
* not "Java-ness",
* not mocking by habit.

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

### Formalization

At this step, behavior is already modeled. Now choose a representation that minimizes the cost of dominant operations.

```text id="ds-formal"
Choose representation R such that:
total_cost(R) = sum(freq(op_i) * cost_R(op_i))
```

This is representation selection under workload assumptions.

If the workload vector changes, the right structure changes. That is why DS choice is late in the funnel: complexity without an operation profile is noise.

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

### Formalization

Concurrency turns the transition system into an interleaving system:

```text id="concurrency-formal"
Global execution = interleaving of local event sequences
```

Now correctness must survive:

* reordering,
* duplication,
* partial failure,
* delayed observation,
* concurrent writers.

Useful proof obligations:

* linearizability for critical shared objects,
* serializability for transaction-like flows,
* at-least-once / at-most-once / exactly-once delivery assumptions,
* idempotence of externally retried commands.

The hidden interview question is usually:

```text id="failure-question"
which guarantees are local, and which require distributed coordination?
```

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

### Formalization

Extensibility is controlled variation under invariant preservation.

```text id="extensibility-formal"
Given future delta_f to requirements,
minimize rewrite(delta_f) subject to preserving I and stable client contracts
```

This means you isolate axes of likely change:

* policy variation,
* workflow variation,
* integration variation,
* representation variation.

Patterns are valid only when they reduce the cost of these expected deltas without obscuring current invariants.

# 5. Pattern Emergence Rule

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

# 6. Universal LLD Heuristic

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

# 7. Step-To-Theory Map

| Funnel Step       | Formal Object                     | Main Failure Mode                  |
| ----------------- | --------------------------------- | ---------------------------------- |
| semantics         | abstract machine / algebra        | solving the wrong problem          |
| invariants        | safety predicates                 | illegal states become representable |
| states            | labeled transition system         | workflow ambiguity                 |
| ownership         | authority graph                   | uncontrolled mutation              |
| responsibilities  | behavior partition                | low cohesion / god objects         |
| dependencies      | DAG / partial order               | cyclic or unstable architecture    |
| interfaces        | contracts / substitutability      | fake abstraction                   |
| data structures   | representation under workload     | asymptotic mismatch                |
| concurrency       | interleaving / consistency model  | races and replay bugs              |
| extensibility     | controlled system morphism        | change amplification               |

---

# 8. Fast Interview Workflow (Practical)

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

# 9. Deep Compression (Most Important)

LLD reduces to 4 things:

| Core Concept | Mathematical Compression |
| ------------ | ------------------------ |
| invariants   | constraint system        |
| workflows    | state machine            |
| dependencies | DAG/partial order        |
| ownership    | authority graph          |

Everything else is implementation detail.

---

# 10. One-Line Mental Model

When solving LLD:

> **Find the constrained state machine, isolate mutation ownership, and minimize dependency entropy.**
