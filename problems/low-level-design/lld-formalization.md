# LLD Formalization

This file isolates the **formal model** behind low-level design.

It is not the interview delivery order.
It is the abstract system model that the interview flow is trying to reconstruct.

---

# 1. Core Claim

A low-level design problem can be modeled as a **constrained state transition system with localized mutation authority over a dependency graph**.

In other words:

> LLD is not fundamentally about classes first.
> It is about defining legal state, legal transitions, and who is allowed to cause them.

---

# 2. System Model

Let a system be:

```text
S = (E, Sigma, X, I, delta, Phi, Omega, D)
```

Where:

- `E` = external events, commands, or inputs
- `Sigma` = abstract system states
- `X` = data representation / memory layout
- `I` = invariants over `Sigma` and `X`
- `delta` = transition relation
- `Phi` = policies, guards, or authorization predicates
- `Omega` = side effects / emitted observations
- `D` = dependency graph over components

This gives the semantic backbone of an LLD problem.

---

# 3. Meaning of Each Component

## 3.1 Events

`E` is the set of externally visible operations or commands.

Examples:

- `park(vehicle)`
- `pay(orderId)`
- `startTrip(driverId, tripId)`
- `makeMove(player, cell)`

These are not implementation details.
They are the stimuli that try to move the system.

## 3.2 Abstract State

`Sigma` is the logical state space of the domain.

This is not yet the class layout.
It is the set of meaningful system conditions.

Examples:

- order lifecycle state
- parking spot occupancy state
- trip assignment state
- board state in a game

If `Sigma` is underspecified, the design will be vague no matter how clean the code looks.

## 3.3 Representation

`X` is the concrete stored representation used to realize the abstract state.

Examples:

- maps
- lists
- heaps
- queues
- indexes
- object fields

Multiple choices of `X` may represent the same abstract `Sigma`.

This is why data structure choice is downstream of semantics and operations.

## 3.4 Invariants

`I = {i1, i2, ..., in}` is the set of predicates that must hold for every reachable legal state.

Each:

```text
ik : Sigma x X -> {true, false}
```

Examples:

- a booked seat cannot be allocated twice
- one vehicle has at most one active ticket
- a delivered order cannot return to paid
- cache size never exceeds capacity

Correctness requires:

```text
For every reachable state s with representation x:
for all ik in I, ik(s, x) = true
```

## 3.5 Transition Relation

`delta` defines how the system evolves.

```text
delta : Sigma x X x E -> Sigma x X
```

For legal operations, `delta` returns the next abstract state and representation.

For illegal operations, one of two things should happen:

- the transition is undefined and must be rejected
- the transition maps to an explicit failure state

This is the formal version of lifecycle and workflow modeling.

## 3.6 Policies and Guards

`Phi` constrains whether a transition may fire.

Examples:

- authorization
- availability checks
- capacity checks
- business rules
- preconditions

Think of `Phi` as the condition layer around `delta`.

## 3.7 Side Effects

`Omega` captures observable outputs and external consequences.

Examples:

- send notification
- emit receipt
- reserve payment
- log audit event
- publish message

This separates internal state change from external effects.

## 3.8 Dependency Graph

`D` is the directed graph of components and their dependencies.

Nodes may be:

- entities
- services
- repositories
- strategies
- adapters

Edges represent:

- calls
- ownership
- data flow
- authority flow

The graph matters because bad LLD is usually a failure of dependency structure:

- cyclic dependencies
- diffuse mutation authority
- low cohesion
- unstable abstractions

---

# 4. Reachability and Correctness

Let `s0, x0` be the initial state and representation.

The system is correct if:

## 4.1 Initial correctness

```text
For all ik in I, ik(s0, x0) = true
```

## 4.2 Transition preservation

For every legal event `e`:

```text
if ik(s, x) = true for all ik in I
and delta(s, x, e) = (s', x')
then ik(s', x') = true for all ik in I
```

This is the core proof obligation:

> legal operations must preserve invariants

## 4.3 Illegal transition handling

For any event not permitted by guards or lifecycle rules:

- reject it
- or route it to an explicit failure model

But do not silently mutate into an invalid state.

---

# 5. Ownership Formalization

Formal LLD is not only about transitions.
It is also about **who has authority to trigger them**.

Define an ownership mapping:

```text
A : Mutation or invariant -> Component
```

Examples:

- ticket issuance invariant -> `ParkingLot`
- occupancy mutation -> `ParkingSpotAllocator`
- fare computation policy -> `PricingStrategy`

Good designs minimize the number of components authorized to mutate a given slice of state.

That gives:

- lower coupling
- easier local reasoning
- simpler invariant preservation
- safer future changes

In practical terms:

> state should live near invariant enforcement, and mutation authority should be narrow

---

# 6. Dependency Graph Constraints

Let `D = (V, Edges)` be the component graph.

Good LLD tries to make `D` satisfy:

## 6.1 Low cycle pressure

Cycles should be rare and justified.

Why:

- they increase reasoning cost
- they spread mutation effects
- they weaken modularity

## 6.2 Localized authority

If many nodes mutate the same conceptual state, invariants leak.

## 6.3 Stable abstractions

Edges should point toward interfaces or stable responsibilities where variability exists.

## 6.4 Clear orchestration boundaries

Cross-entity workflows should have an identifiable coordination point.

---

# 7. Classes as a Representation Layer

Classes are not the first-class object of the formal model.

They are one implementation of:

- `Sigma` through owned state
- `delta` through methods
- `I` through invariant checks
- `Phi` through guards
- `Omega` through side-effect boundaries
- `D` through references and call edges

So class design is best understood as:

> an encoding of the formal system into maintainable program structure

---

# 8. Why the Interview Funnel Exists

The usual interview steps reconstruct this model in a practical order:

- requirements clarify `E`
- invariants clarify `I`
- state machine clarifies `Sigma` and `delta`
- entities and ownership map `Sigma` and authority boundaries into components
- interfaces and dependencies shape `D`
- data structures refine `X`
- flow tracing validates that `delta` preserves `I`

That is why the interview flow works.

It is not arbitrary ceremony.

---

# 9. Minimal Formal Compression

If you want the shortest possible statement:

```text
LLD = choose state, constrain it with invariants, define legal transitions,
localize mutation authority, and embed the result in a low-coupling dependency graph.
```

---

# 10. What This Model Does Not Cover Well

This formalization is strong for bounded local design.

It is weaker for:

- distributed consistency
- network partitions
- deployment evolution
- observability systems
- socio-technical constraints

Those require higher-level architecture models in addition to local design formalization.
