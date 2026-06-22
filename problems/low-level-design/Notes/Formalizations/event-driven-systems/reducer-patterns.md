A reducer is the core semantic object:

$$
S \times E \to S
$$

Everything “around reducers” is basically about **how you structure composition, side effects, distribution, and state evolution while preserving that transition purity (or deliberately breaking it in controlled ways).**

So the design patterns cluster into a few causal layers:

---

# 1. Pure Reducer Pattern (baseline kernel)

### Intent: single-step state transition

```python
def reducer(state, event):
    if event["type"] == "inc":
        return {**state, "x": state["x"] + 1}
    return state
```

### Structural property

* deterministic
* referentially transparent
* monoid action over event stream

This is the “ground object” everything else modifies or wraps.

---

# 2. Reducer Composition (function algebra)

### Intent: build large systems from small transitions

## (a) Sequential composition (pipeline folding)

```python
def compose(r1, r2):
    return lambda s, e: r2(r1(s, e), e)
```

More common form: layered reducers

```python
def base(state, event): ...
def with_logging(state, event):
    print(event)
    return base(state, event)
```

### Insight

* reducers form a **category of endofunctions on state space**
* composition = morphism composition

---

# 3. Higher-Order Reducers (middleware pattern)

### Intent: transform reducers themselves

```python
def logger(reducer):
    def wrapped(state, event):
        print("event:", event)
        return reducer(state, event)
    return wrapped
```

### Structural meaning

* reducer transformer:
  $$
  R \to R
  $$

This is the “decorator pattern” but semantically grounded in transition systems.

---

# 4. Partitioned Reducers (event routing / modularity)

### Intent: split state space into independent transition subspaces

```python
def user_reducer(state, event):
    if event["type"] == "user":
        return {...}
    return state

def order_reducer(state, event):
    if event["type"] == "order":
        return {...}
    return state
```

### Combined:

```python
def root_reducer(state, event):
    state = user_reducer(state, event)
    state = order_reducer(state, event)
    return state
```

### Formal view

State decomposes:

$$
S = S_1 \times S_2 \times ... \times S_n
$$

Reducers act on projections.

---

# 5. Combined Reducer (Redux-style aggregation)

### Intent: merge independent reducers into one global transition

```python
def combine(reducers):
    def root(state, event):
        for r in reducers:
            state = r(state, event)
        return state
    return root
```

### Meaning

* product system of LTSs
* interleaved transitions on shared event

### Risk

* hidden coupling via shared state mutation order

---

# 6. Event-Sourcing Reducer (history fold)

### Intent: state is derived, not primary

```python
def rebuild(events, initial, reducer):
    state = initial
    for e in events:
        state = reducer(state, e)
    return state
```

### Dual object view

* state = fold(E*)
* event log = primary artifact

### Pattern variants

* snapshot + replay
* partial recomputation

---

# 7. Reducer with Side Effects (controlled impurity)

### Intent: isolate effects from pure transition logic

```python
def reducer(state, event, emit):
    if even["type"] == "payment":
        emit({"type": "charge_card"})
        return state
    return state
```

### Structural decomposition

Split into:

* pure transition: $ S \times E \to S $
* effect interpreter: $ E \to IO $

This is the **separation of algebra from runtime**.

---

# 8. Effectful Reducer Interpreter Pattern

### Intent: treat effects as a second semantic layer

```python
def run(reducer, state, events, effect_handler):
    for e in events:
        state, effects = reducer(state, e)
        for eff in effects:
            effect_handler(eff)
    return state
```

### Formalization

$$
S \times E \to (S, \mathcal{E}_{ff})
$$

This is a **coalgebraic reducer** (state + emissions).

---

# 9. Async Reducer / Stream Reducer

### Intent: continuous event ingestion

```python
async def reducer_loop(queue, state):
    while True:
        event = await queue.get()
        state = reducer(state, event)
```

### Meaning

* reducer becomes a **stream processor**
* event stream replaces finite list

---

# 10. Reducer with Local State Partitioning (Entity reducers)

### Intent: isolate per-key state evolution

```python
def reducer(state, event):
    id = event["id"]
    sub = state.get(id, {})

    if event["type"] == "update":
        sub["value"] = event["value"]

    state[id] = sub
    return state
```

### Structural meaning

* state = map of independent LTS instances
* common in:

  * chat systems
  * game entities
  * distributed caches

---

# 11. Optimizing Reducers (memoization / idempotence)

### Intent: avoid recomputation in event folds

### Pattern: checkpointing

```python
snapshots = {100: state_at_100_events}
```

Then:

$$
S_n = \text{fold}(S_k, E_{k:n})
$$

### Key property

* reduces full trace recomputation complexity

---

# 12. Reducer Algebra (deep structure)

Reducers naturally form:

### (a) Monoid action

* event list monoid: $ (E^*, \cdot) $
* state transformation action:
  $$
  S \times E^* \to S
  $$

### (b) Endofunction semigroup

* each event defines:
  $$
  f_e: S \to S
  $$

composition:
$$
f_{e_1} \circ f_{e_2}
$$

### (c) Category structure

* objects: state spaces
* morphisms: reducers

---

# 13. Common failure modes in reducer architectures

### 1. Hidden mutation

Breaks referential transparency → destroys replay correctness

### 2. Order coupling

Reducer correctness depends on event ordering that should be irrelevant

### 3. Cross-reducer leakage

Shared state violates decomposition $ S = \prod S_i $

### 4. Non-idempotent events

Replay produces divergence

---

# 14. Mental compression map

Think of reducer patterns as 4 axes:

### Axis 1: purity

* pure → side-effecting

### Axis 2: structure

* monolithic → partitioned → combined

### Axis 3: time model

* batch → stream → infinite loop

### Axis 4: history usage

* state-only → event-sourced → snapshot + replay

---

If you want, I can next map these directly to:

* Redux / Flux / Elm architecture formal equivalences
* actor-model reducers (how Akka or Erlang implement them)
* or a minimal “universal reducer framework” in Python that supports all of these patterns cleanly
