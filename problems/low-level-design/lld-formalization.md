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

$$ S = (E, \Sigma, X, I, \delta, \Phi, \Omega, D) $$

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

## 2.1 Notation Conventions

To keep the formulas readable:

- `×` means product / tuple input
- `→` means total map or function
- `⇀` means partial function
- `⊆` means subset
- `∀` means "for all"
- `∈` means "is an element of"
- `Result[A, B]` means either success value `A` or failure value `B`

When exact mathematical precision is less important than readability, the notation stays slightly informal on purpose.

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

$$ i_k : \Sigma \times X \to \{\mathrm{true}, \mathrm{false}\} $$

Examples:

- a booked seat cannot be allocated twice
- one vehicle has at most one active ticket
- a delivered order cannot return to paid
- cache size never exceeds capacity

Correctness requires:

$$ \forall \, (s,x) \in \mathrm{Reachable}(S), \; \forall i_k \in I,\; i_k(s,x) = \mathrm{true} $$

## 3.5 Transition Relation

`delta` defines how the system evolves.

$$ \delta : \Sigma \times X \times E \to \Sigma \times X $$

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

$$ \forall i_k \in I,\; i_k(s_0, x_0) = \mathrm{true} $$

## 4.2 Transition preservation

For every legal event `e`:

$$ \Big( \forall i_k \in I,\; i_k(s,x) = \mathrm{true} \Big) \land \delta(s,x,e) = (s',x') \implies \Big( \forall i_k \in I,\; i_k(s',x') = \mathrm{true} \Big) $$

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

$$ A : (\mathrm{mutation}\ \mathrm{or}\ \mathrm{invariant}) \to \mathrm{component} $$

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

# 7. Extensibility Formalization

Extensibility is not "the code can be edited later."

Formally, extensibility is about whether a new requirement can be absorbed by adding or substituting a bounded part of the model without forcing broad modification of already-correct structure.

Let a requirement change be:

$$ r : (E, \Sigma, X, I, \delta, \Phi, \Omega, D) \to (E', \Sigma', X', I', \delta', \Phi', \Omega', D') $$

A design is more extensible when `r` has low propagation cost.

## 7.1 Change propagation surface

Define:

$$ P(r) = \{ c \mid c \text{ is an existing component or function changed by } r \} $$

Then a practical extensibility objective is:

$$ minimize |P(r)| $$

for common foreseeable changes.

Examples:

- new vehicle type
- new pricing rule
- new allocation policy
- new notification mechanism

## 7.2 Stable core vs variable policy

Some parts of the model should remain structurally stable:

- core state transitions
- core invariants
- aggregate ownership boundaries

Other parts are natural variation points:

- pricing policy
- selection policy
- formatting or notification policy

A good LLD isolates likely change axes so that:

$$ \delta_{\mathrm{core}} \text{ stays stable while } \Phi_{\mathrm{var}}, \text{ policy modules, or bounded subgraphs of } D \text{ vary} $$

In practice:

> good extensibility means new variants change a narrow policy boundary, not the whole transition graph

## 7.3 Extensibility as graph-local change

In dependency terms, a design is extensible when new behavior can be introduced by:

- adding a node
- swapping a node behind a stable edge
- extending local state without violating global invariants

and does not require rewiring large portions of `D`.

Weak extensibility usually looks like:

- enum growth plus `if/else` edits scattered everywhere
- invariant checks duplicated across many nodes
- workflow changes coupled directly to representation details

Strong extensibility usually looks like:

- one stable orchestration boundary
- one explicit variation boundary
- preserved invariants under substitution

## 7.4 Structure-preserving change

The deeper formal question is:

> when does a system change preserve the design shape rather than forcing redesign?

Let:

$$ S = (E, \Sigma, X, I, \delta, \Phi, \Omega, D), \quad S' = (E', \Sigma', X', I', \delta', \Phi', \Omega', D') $$

A requirement change is *shape-preserving* when there exist embeddings:

$$ f_E : E \to E', \quad f_{\Sigma} : \Sigma \to \Sigma', \quad f_X : X \to X', \quad f_D : D \to D' $$

such that the old model still commutes inside the new one:

$$ f_{\Sigma}(\delta(s,x,e)) = \delta'(f_{\Sigma}(s), f_X(x), f_E(e)) $$

for all previously legal `e` in the unchanged portion of the system.

This is the useful LLD notion of "I added capability without breaking the old conceptual machine."

In plain terms:

- old legal workflows still mean the same thing
- old invariants remain valid or lift naturally
- the previous dependency shape embeds into the new one

That is close to an isomorphism idea, but usually not full isomorphism.
It is more often a **homomorphic extension** or **conservative extension**:

- old behavior is preserved
- new behavior is added
- the original model remains intelligible inside the larger one

## 7.5 Partial order of design changes

Requirement changes are not all equally invasive.

Define a preorder over changes:

$$ r_1 \le r_2 \iff \text{every component changed by } r_1 \text{ is also changed by } r_2 $$

or more abstractly:

$$ r_1 \le r_2 \iff P(r_1) \subseteq P(r_2) $$

This induces a rough ordering of extensibility cost.

Examples:

- adding one pricing strategy is typically `<` rewriting the parking workflow
- adding one spot subtype is typically `<` changing aggregate ownership

So extensibility can be thought of as biasing the system toward a lower-order set of expected requirement changes.

## 7.6 Type-level view of extensibility

Another formalization is through sum and product structure.

Examples:

- adding a new variant to `VehicleType` is a sum-type extension
- adding a new field to a ticket is a product-type extension

These have different blast radii.

For sum-like variation:

$$ \mathrm{VehicleType} = \mathrm{Bike} \mid \mathrm{Car} \mid \mathrm{Truck} $$

naive branching often forces edits everywhere the sum is inspected.

For policy-like variation:

$$ \mathrm{FeePolicy} : \mathrm{Session} \to \mathrm{Money}, \quad \mathrm{AllocationPolicy} : \mathrm{LotState} \times \mathrm{Vehicle} \to \mathrm{Spot}\ \mathrm{option} $$

the change is localized by moving variability into a typed boundary.

So a key extensibility heuristic is:

> if a change axis is expected to grow, prefer a stable function boundary over repeated case analysis

## 7.7 Conservative extension of invariants

Suppose `I` is the original invariant set and `I'` is the new one after change.

A well-structured extension often satisfies:

$$ I \subseteq I' $$

or at least:

$$ all old reachable states that remain valid still satisfy the lifted form of I $$

This matters because some changes add constraints while others weaken them.

Good extensibility avoids gratuitous invariant rewriting.
It prefers extending the invariant system in a way that preserves the old correctness story.

---

# 8. Requirement Change Formalization

Requirements are not just prose.
They define the admissible evolution of the model.

Let `R` be the requirement set.

Then a design problem is better written as:

$$ \mathrm{Design} : R \to S $$

where `S` is the formal system model.

## 8.1 Requirement changes as model transformations

A requirement update:

$$ \rho : R \to R' $$

induces a model transformation:

$$ T(\rho) : S \to S' $$

The quality of the original design can be partly judged by the properties of `T(rho)` for foreseeable `rho`.

Questions:

1. does `T(rho)` preserve prior workflows
2. does `T(rho)` preserve prior invariants
3. does `T(rho)` only extend policy boundaries
4. does `T(rho)` force ownership redistribution

## 8.2 Taxonomy of requirement changes

A requirement change usually lands in one or more of these buckets:

1. event extension
   - add a new command to `E`
2. state extension
   - add new lifecycle states or fields to `Sigma`
3. representation refinement
   - change indexes or storage layout in `X`
4. invariant extension
   - add or tighten correctness rules in `I`
5. guard or policy change
   - modify `Phi`
6. side-effect change
   - modify `Omega`
7. dependency or authority change
   - rewire `D` or `A`

These are not equally expensive.
Changes to `Phi` or bounded parts of `Omega` are often cheaper than changes to `Sigma`, `I`, or `A`.

## 8.3 Requirement volatility and design pressure

Let:

$$ V : \mathrm{requirement\_dimension} \to \mathrm{expected\_change\_frequency} $$

Then design quality depends partly on aligning abstraction boundaries to high-volatility dimensions.

In plain terms:

- stable requirements may be encoded directly
- volatile requirements should be isolated

This is the formal reason the same design can be elegant under one requirement distribution and poor under another.

---

# 9. Data Structure Formalization

Data structures are not primary semantics.
They are a representation choice constrained by state shape and operation load.

## 9.1 Representation relation

Let:

$$ \mathrm{Rep} : X \to \Sigma $$

map concrete representation to abstract state.

Correctness of a data structure choice requires that operations over `X` faithfully realize the desired transitions over `Sigma`.

That is:

$$ \mathrm{Rep}(x) = s \land \mathrm{update}_X(x,e) = x' \implies \mathrm{Rep}(x') = \mathrm{abstract\_state}(\delta(s,x,e)) $$

up to the intended abstraction boundary.

## 9.2 Operation cost model

Let the workload be:

$$ W : E \to \mathrm{frequency\_or\_weight} $$

and let:

$$ C_X(e) = \mathrm{cost\ of\ realizing\ event}\ e\ \mathrm{under\ representation}\ X $$

Then a representation choice is partly an optimization problem:

$$ \text{choose } X \text{ to minimize } \sum_{e \in E} W(e)\cdot C_X(e) $$

subject to:

1. invariant preservation
2. acceptable memory cost
3. acceptable update complexity
4. acceptable concurrency properties

This is the formal reason data structure choice comes after operation and workload clarification.

## 9.3 Indexes as derived structures

Many designs need both a primary representation and derived indexes.

Example:

- canonical map from `ticketId -> Ticket`
- derived map from `vehicleId -> ticketId`
- priority structure for nearest spot lookup

Each derived structure introduces a consistency obligation:

$$ \mathrm{index}(x) \text{ must agree with } \mathrm{canonical}(x) $$

So every index is not just a performance decision.
It is also a new invariant.

## 9.4 Representation tradeoff formalization

Every representation introduces a tradeoff surface across:

1. lookup cost
2. mutation cost
3. ordering guarantees
4. memory overhead
5. synchronization complexity

This means:

> a data structure is part of the correctness-performance-concurrency compromise, not just a coding convenience

---

# 10. Concurrency Formalization

Sequential transition systems are often not enough.
Some LLD problems implicitly require concurrent reasoning.

Let:

$$ \delta_e : \Sigma \times X \to \Sigma \times X $$

be the transition induced by event `e`.

Under concurrency, the system executes interleavings of transitions:

$$ delta_e2(delta_e1(s, x)) $$

and

$$ delta_e1(delta_e2(s, x)) $$

may differ.

## 10.1 Commutativity and conflict

Two operations commute if:

$$ delta_e2(delta_e1(s, x)) = delta_e1(delta_e2(s, x)) $$

for all relevant states.

If they do not commute, the operations conflict and require serialization, isolation, or compensation.

Examples:

- two reads usually commute
- two allocations of the same spot do not
- fee calculation and spot release may need atomic ordering

## 10.2 Atomicity boundary

Concurrency design requires choosing an atomicity scope:

$$ \mathrm{Atomic}(op) = \text{the smallest state slice that must change as one unit} $$

A good atomicity boundary aligns with invariant boundaries.

If an invariant spans:

- spot occupancy
- active ticket table
- vehicle-to-session link

then splitting those mutations across independently visible steps creates race risk.

## 10.3 Linearizability intuition

For user-facing correctness, many systems aim for an illusion:

> each operation appears to take effect at one logical instant between call and return

That is the practical intuition behind linearizable APIs.

Even without full formal proof, LLD answers improve when they identify:

1. the linearization point
2. the protected state
3. the lock or compare-and-swap boundary

## 10.4 Concurrency control choices

Concurrency control is a choice of how to preserve invariants under non-commuting operations.

Typical mechanisms:

1. coarse-grained locking
2. fine-grained locking
3. optimistic validation
4. immutable snapshots plus swap
5. queue-based serialization

The right choice depends on:

1. contention shape
2. invariant span
3. latency tolerance
4. failure recovery model

## 10.5 Deadlock and ordering discipline

If multiple locks or resources are acquired, define an order:

$$ L1 < L2 < ... < Ln $$

and require acquisition to follow that order.

This is a simple partial-order argument against deadlock:

if all acquisitions respect one global order, circular wait is prevented.

---

# 11. Happy Path and Failure Path Formalization

Designs are usually explained as flows.
Formally, a flow is a path through the transition system.

## 11.1 Happy path

A happy path is a sequence:

$$ \pi_h = [e_1, e_2, \ldots, e_n] $$

such that every transition is legal and every post-state preserves invariants.

Execution means:

$$ (s_0, x_0) \to (s_1, x_1) \to \cdots \to (s_n, x_n) $$

with no rejection and expected outputs in `Omega`.

Happy paths validate:

1. state reachability
2. API coherence
3. ownership continuity
4. representation adequacy

## 11.2 Failure path

A failure path is a sequence where some event hits guard failure, dependency failure, invariant risk, or explicit error state:

$$ \pi_f = [e_1, e_2, \ldots, e_k] $$

with:

$$ \delta_f(s_k, x_k, e_{k+1}) = \mathrm{failure} $$

or transition into a modeled compensating state.

Failure-path design is good when it answers:

1. what partial work has happened
2. what state must be rolled back or marked pending
3. what the caller observes
4. whether invariants still hold

## 11.3 Path coverage heuristic

For each major API, the minimum useful reasoning set is:

1. one happy path
2. one guard-failure path
3. one dependency-failure or contention path if relevant

This is often enough to expose missing state, missing indexes, or broken ownership.

---

# 12. Failure Mode Formalization

Failure modes are not just bugs.

They are the formally meaningful ways the system can reject, degrade, or violate expected behavior under an event, state, or dependency condition.

Extend transition semantics to:

$$ \delta_f : \Sigma \times X \times E \to \mathrm{Result}[(\Sigma \times X), F] $$

Where `F` is a set of explicit failure outcomes.

Examples:

- capacity exceeded
- invalid lifecycle transition
- authorization denied
- missing dependency
- payment failed

## 12.1 Sources of failure

A failure can arise from at least four places:

1. guard failure in `Phi`
2. invariant risk detected before mutation
3. representation or lookup failure in `X`
4. side-effect or dependency failure in `Omega` or `D`

This is useful because many vague LLD answers talk only about the happy path.
Formal reasoning improves when failure is attached to the layer that produced it.

## 12.2 Failure-state discipline

For any event `e`, one of these should hold:

1. `e` is legal and preserves invariants
2. `e` is rejected with explicit failure `f in F`
3. `e` transitions into a modeled recovery or pending state

But the system should not:

1. partially mutate and pretend success
2. leave ownership unclear after failure
3. violate invariants silently

So failure modeling is part of correctness, not an afterthought.

## 12.3 Failure locality

Good LLD keeps failure handling near the boundary that can explain or recover from it.

Examples:

- guard failures near orchestration entry points
- storage lookup failures near repositories or indexes
- external effect failures near adapters

This avoids a design where all failures collapse into one vague global error path.

## 12.4 Failure mode checklist

For any core operation, ask:

1. what precondition failure can reject this event
2. what invariant could be threatened
3. what dependency can fail mid-flow
4. whether mutation is atomic, compensating, or staged
5. what state the system is left in after failure

That is the minimal formal failure pass for an LLD design.

---

# 13. Interface Formalization

Interfaces are not just language syntax.
Formally, they are boundary contracts that expose an admissible observation and command surface while hiding representation and internal dependency choices.

An interface defines:

1. what operations are visible
2. what inputs and outputs are allowed
3. what semantic guarantees callers may rely on
4. what implementation details remain hidden

## 13.1 Interface as boundary projection

Let a component have internal model:

$$ C = (\Sigma_c, X_c, I_c, \delta_c, \Omega_c) $$

An interface is a projection:

$$ \pi_c : C \to I_c $$

where `Ic` exposes only:

- a set of callable operations
- visible result types
- selected guarantees

and deliberately omits:

- hidden state layout
- internal helper transitions
- private indexes
- internal dependency wiring

So an interface is a controlled loss of information.

That is good.
It reduces coupling by preventing callers from depending on accidental structure.

## 13.2 Interface as a behavioral contract

A useful interface is not just a method list.
It is a contract over traces.

For each exposed operation `op`, specify:

$$ op : \mathrm{Input} \to \mathrm{Result}[\mathrm{Output}, \mathrm{Error}] $$

together with:

1. preconditions
2. postconditions
3. failure cases
4. side-effect guarantees
5. concurrency guarantees if relevant

So the real semantic interface is closer to:

$$ \mathrm{Contract}(op) = (\mathrm{Pre}, \mathrm{Post}, \mathrm{Fail}, \mathrm{Effects}, \mathrm{ConcurrencySemantics}) $$

This is why two interfaces with identical method signatures may still be radically different designs.

## 13.3 Interface refinement

Suppose two interfaces `I1` and `I2` represent versions of the same boundary.

`I2` refines `I1` if:

1. every valid `I1` client remains valid against `I2`
2. `I2` preserves the guarantees of `I1`
3. `I2` may strengthen internal structure without exposing new accidental coupling

This is the interface-level version of conservative extension.

It matters because good LLD often depends on being able to swap implementations or extend behavior without breaking callers.

## 13.4 Dependency inversion as interface factoring

If component `A` depends on component `B` only through an interface `IB`, then the dependency edge is not really:

$$ A \to B $$

but rather:

$$ A \to I_B \leftarrow B $$

This matters formally because it changes the dependency graph.

The caller depends on:

- the contract
- not the implementation node

This weakens coupling and localizes change.

So interface extraction is not mere style.
It is graph factoring that can reduce propagation of requirement change.

## 13.5 Interface width and leakiness

Define the width of an interface roughly as the amount of behavior and structure it exposes.

Very wide interfaces are risky because:

1. more callers can couple to more semantics
2. substitution becomes harder
3. change propagation surface increases

Very narrow interfaces are risky when:

1. they hide required invariants too aggressively
2. orchestration code must reconstruct missing semantics externally

So interface design is an optimization problem:

> expose enough to preserve correctness and usability, but not enough to leak representation or unnecessary policy

## 13.6 Interface placement heuristic

Introduce an interface when at least one of these is true:

1. multiple implementations are plausible
2. the dependency points toward a volatile policy
3. the caller should not observe representation details
4. the boundary is important for testing, substitution, or isolation

Do not introduce an interface just because a language allows one.

An interface with only one implementation can still be useful if it isolates a high-volatility or side-effecting boundary.
An interface around a perfectly stable, purely internal entity often adds little.

## 13.7 Interfaces and invariants

Interfaces should align with invariant ownership.

If a component owns an invariant, its interface should expose operations that preserve that invariant as a unit.

Bad sign:

- callers must orchestrate several low-level calls in the right order to avoid breaking the component's invariant

Better sign:

- the interface exposes one operation that preserves the invariant internally

So interface granularity should track mutation closure, not just object vocabulary.

---

# 14. Classes as a Representation Layer

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

# 15. Why the Interview Funnel Exists

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

# 16. Minimal Formal Compression

If you want the shortest possible statement:

$$ \mathrm{LLD} = \text{choose state, constrain it with invariants, define legal transitions, localize mutation authority, and embed the result in a low-coupling dependency graph} $$

---

# 17. What This Model Does Not Cover Well

This formalization is strong for bounded local design.

It is weaker for:

- distributed consistency
- network partitions
- deployment evolution
- observability systems
- socio-technical constraints

Those require higher-level architecture models in addition to local design formalization.
