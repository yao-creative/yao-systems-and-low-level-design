# Projected State Stores vs Runtime State: Trade-offs, Patterns, and Formalization

The architectural question is:

> Should derived/projected state be persisted as a first-class store, or should it exist only inside runtime execution state?

This is a **state materialization design problem** involving:

* durability
* recomputation cost
* latency
* consistency
* operational complexity

---

# 1. Core Model

A projection transforms a source history into a derived state:

$$
P:E^* \rightarrow S_p
$$

where:

* $E^*$ = sequence of events
* $P$ = projection function
* $S_p$ = projected state

Example:

$$
(Event_1,Event_2,...,Event_n)
\xrightarrow{P}
UserView
$$

The two choices are:

1. Keep $S_p$ only in memory.
2. Persist $S_p$ as a materialized store.

---

# 2. Runtime-only State

Architecture:

$$
Events \rightarrow Runtime \rightarrow State
$$

Example:

```rust
let mut balance = 0;

for event in events {
    balance = apply(balance, event);
}
```

The state exists only during execution.

The computation is:

$$
apply:S \times E \rightarrow S
$$

and replay is:

$$
fold(apply,s_0,[e_1,...,e_n])=s_n
$$

## Advantages

### 1. Strong consistency

There is only one representation:

$$
S_{runtime}
$$

The invariant is:

$$
I(S_{runtime})
$$

There is no synchronization problem.

---

### 2. Simpler correctness model

You avoid consistency conditions such as:

$$
S_{projection}=P(S_{source})
$$

because no second state exists.

---

### 3. Suitable for small or temporary state

Examples:

* request context
* compiler intermediate state
* game simulation state
* transaction execution state

## Disadvantages

The main cost is recovery.

After failure:

$$
S_{runtime}=\emptyset
$$

The system must reconstruct:

$$
Replay(E^*) \rightarrow S_{runtime}
$$

Recovery cost:

$$
T_{recovery}=T_{replay}+T_{execution}
$$

Large histories make this expensive.

---

# 3. Persistent Projection Store

Architecture:

$$
Events
\rightarrow
Projection Engine
\rightarrow
Projection Database
\rightarrow
Runtime
$$

Example:

Event stream:

```
UserCreated
EmailChanged
OrderPlaced
PaymentReceived
```

Projection:

```
users_view

id | email | total_spent
------------------------
1  | a@x   | 500
```

The projection becomes:

$$
S_p=P(E^*)
$$

stored permanently.

---

# Advantages

## 1. Fast reads

Without projection:

$$
Read = Replay(E)
$$

Cost:

$$
O(|E|)
$$

With projection:

$$
Read = Lookup(S_p)
$$

Cost:

$$
O(1)
$$

---

## 2. Multiple read models

One source can generate many projections:

$$
P_1(E)=UserView
$$

$$
P_2(E)=AnalyticsView
$$

$$
P_3(E)=SearchIndex
$$

Architecture:

$$
\begin{array}{c}
Event\ Store\
|\
+---+---+---+\
|   |   |\
User Search Analytics
\end{array}
$$

This is the foundation of CQRS.

---

## 3. Independent scaling

The write model and read models scale separately:

$$
WriteLoad \neq ReadLoad
$$

---

# Disadvantages

You introduce distributed consistency.

Instead of:

$$
S_{source}=S_{projection}
$$

you have:

$$
Eventually:
S_{projection}=P(S_{source})
$$

Problems:

* projection lag
* replay bugs
* schema migration
* synchronization failures

---

# 4. Hybrid Pattern (Most Production Systems)

Most systems combine both approaches:

$$
Persist\ Projection + Hydrate\ Runtime
$$

Flow:

$$
Request
\rightarrow
Projection Store
\rightarrow
Runtime Object
\rightarrow
Mutation
\rightarrow
Events
\rightarrow
Projection Update
$$

Example:

Stored:

```
account_projection

balance = 1000
```

Runtime:

```rust
struct Account {
    id: AccountId,
    balance: Money,
    rules: Rules
}
```

Hydration:

$$
hydrate:S_p \rightarrow S_r
$$

Mutation:

$$
f:S_r \times Command \rightarrow (S_r,E)
$$

---

# 5. Common Patterns

## Pattern 1: Event Sourcing Aggregate

Source of truth:

$$
EventStore
$$

Runtime:

$$
Aggregate
$$

Projection:

$$
EventStore \rightarrow ReadModel
$$

Example:

Events:

```
OrderCreated
ItemAdded
PaymentReceived
```

Aggregate:

```
Order {
 status,
 items,
 payment
}
```

Projection:

```
orders_table
```

---

## Pattern 2: Cache as Projection

Example:

$$
Database \rightarrow Redis
$$

Redis represents:

$$
Redis=P(Database)
$$

Redis is not the source of truth.

If Redis fails:

$$
Redis \rightarrow Rebuild(Database)
$$

---

## Pattern 3: Materialized View

Database view:

$$
View=Query(Database)
$$

Materialized view:

$$
MaterializedView=Snapshot(Query(Database))
$$

Trade-off:

$$
Storage \leftrightarrow Computation
$$

---

## Pattern 4: Stream Processor State

Example:

Kafka Streams / Flink:

$$
Events
\rightarrow
Operator
\rightarrow
StateStore
$$

State:

$$
S_t=fold(apply,S_0,E_{0:t})
$$

Checkpoint:

$$
checkpoint(S_t)
$$

The state is both:

* runtime execution state
* recoverable persistent state

---

# 6. Decision Rule

Use runtime-only state when:

$$
ReplayCost \ll StorageCost
$$

Examples:

* request handlers
* small aggregates
* simulations

Use persistent projections when:

$$
ReadFrequency \times ReplayCost \gg StorageCost
$$

Examples:

* dashboards
* search systems
* analytics
* user profiles

Use hybrid when:

$$
Need(Durability)+Need(FastExecution)
$$

This is the dominant pattern in large-scale systems.

---

# 7. Category-Theoretic Formalization

Let:

* $\mathcal{E}$ = category of event histories
* $\mathcal{S}$ = category of states

Projection:

$$
P:\mathcal{E}\rightarrow\mathcal{S}
$$

The projection functor maps:

$$
EventHistory \mapsto State
$$

Runtime state is a temporary object:

$$
S_r \in \mathcal{S}
$$

Persistent projection is the retained image:

$$
store(P(E))
$$

Therefore the architectural question is:

> Do we retain the image of the projection functor, or recompute it from the source object when required?

The practical answer:

* retain expensive projections
* keep cheap execution state ephemeral
* use event history as the reconstruction mechanism
