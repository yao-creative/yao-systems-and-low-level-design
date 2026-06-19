Your confusion is actually very common. The root issue is that you are mixing **three different kinds of statements**:

1. **Model assumptions** — "What world are we simulating?"
2. **System policies** — "How do we choose among valid options?"
3. **Invariants** — "What states must be impossible?"

The interviewer is being quite rigorous. Let's analyze each one.

---

# What is an invariant formally?

An invariant is a predicate (P(S)) over the system state (S) such that:

$$
\forall t,\ P(S_t)=\text{true}
$$

Meaning:

* At initialization: true
* After every operation: true
* During every possible execution: true

It is a **safety property**:

> "The system can never enter an illegal state."

Not:

* how it behaves over time
* how it chooses actions
* what algorithm it uses

---

# Your original Invariant 1

> Total people in elevators + queues = requests - completions

### Why you thought this was reasonable

Actually, this is a perfectly valid invariant **for a passenger simulation**.

Example state:

```
Floor 1 queue: 3 people
Elevator A: 2 people
Completed trips: 5 people
```

Conservation law:

```
Current passengers + waiting passengers + completed passengers
=
Total generated requests
```

This is analogous to conservation of mass.

---

## Why the interviewer rejected it

Because your requirements explicitly said:

> Treat requests as events, not full person simulation.

Your state does not have:

```
Floor.people[]
Elevator.passengers[]
```

Therefore your invariant refers to state variables that do not exist.

You cannot enforce:

```
people_in_system = requests - completions
```

because the system is not tracking people.

---

# Your original Invariant 2

> Scheduled requests should be processed in order of arrival or nearest-car policy.

This is mixing policy with correctness.

Let's separate them.

---

## Scheduling policy

Example:

```python
chooseElevator(request):
    return nearestElevator()
```

This is a strategy.

You could replace it:

```python
return roundRobin()
```

or:

```python
return leastLoaded()
```

The system is still correct.

---

## What is the invariant behind it?

Something like:

> Every assigned request is assigned to exactly one elevator.

Formally:

For every request (r):

$$
|{e \mid r \in e.assignedRequests}| \leq 1
$$

Meaning a request cannot be owned by two elevators.

---

# Your 2C Invariant 1

> Elevator moves at a constant speed.

This is a modeling assumption.

It defines the simulator.

Example:

```
speed = 1 floor / second
```

The system is not incorrect if:

```
speed = 2 floors / second
```

or:

```
speed varies
```

Therefore it is not an invariant.

---

# Your 2C Invariant 2

> Elevator moves only when there is a request to serve.

This one is actually close.

The interviewer was not saying the idea was wrong.

The wording was muddy.

You wrote:

> moving to boarding floor or during boarding

But boarding means:

```
doors open
passengers entering/exiting
```

An elevator should not be moving while boarding.

You accidentally combined two mutually exclusive states.

---

## A better invariant

```
If elevator.state == MOVING,
then elevator.targetFloor exists.
```

or:

$$
moving \implies targetFloor \neq null
$$

Meaning:

A moving elevator must have a legitimate destination.

---

# Your 2C Invariant 3

> Elevator cannot move to a floor that is not in the building.

This is correct.

Formal:

$$
0 \leq currentFloor < numberOfFloors
$$

and:

$$
0 \leq targetFloor < numberOfFloors
$$


Good invariant.

---

# Your 2C Invariant 4

> Each elevator owns a request upon request dispatching.

Your intuition is right.

The issue is the phrase:

> upon request dispatching

That describes a **transition**.

An invariant should describe a **state**.

---

Bad:

```
When dispatch happens,
elevator gets the request.
```

That is a workflow.

---

Good:

```
Every assigned request belongs to exactly one elevator.
```

or:

$$
\forall r,\ |\text{owners}(r)| = 1
$$

after assignment.

---

# A strong final invariant set

## 1. Floor legality

```
An elevator's current and target floors
must be within the building bounds.
```

---

## 2. Capacity safety

```
The elevator occupancy cannot exceed its maximum capacity.
```

$$
occupancy \leq capacity
$$

---

## 3. Motion legality

```
A moving elevator must have a valid target floor.
```

or:

$$
state = MOVING \Rightarrow targetFloor \neq null
$$

---

## 4. Door-motion exclusivity

```
An elevator cannot move while its doors are open.
```

$$
moving \Rightarrow door = CLOSED
$$

---

## 5. Request ownership uniqueness

```
A request can be assigned to at most one elevator.
```

$$
|\{e : r \in e.requests\}| \leq 1
$$

---

# The deeper lesson for LLD interviews

Before writing an invariant, ask:

### Is this a state property?

Good:

```
elevator.currentFloor <= maxFloor
```

Bad:

```
Elevator chooses the nearest request.
```

That is a decision policy.

---

### Is this part of the modeled state?

Good:

```
occupancy <= capacity
```

because `occupancy` exists.

Bad:

```
Total people in the building are conserved.
```

if you never modeled people.

---

### Is it true before and after every operation?

Good:

```
A request has one owner.
```

Bad:

```
When a request arrives it is assigned.
```

because there may be a moment before scheduling.

---

So the interviewer is largely correct. Your reasoning was not "wrong"; you were identifying legitimate properties, but they belonged to **a different abstraction layer**:

| Your idea              | Actually is                                           |
| ---------------------- | ----------------------------------------------------- |
| People conservation    | Domain invariant for a passenger simulator            |
| FIFO / nearest-car     | Scheduling policy                                     |
| Constant speed         | Modeling assumption                                   |
| Move only with purpose | Correct invariant but needed better state formulation |
| Floor bounds           | Correct invariant                                     |
| Request ownership      | Correct invariant but needed state-based wording      |

This distinction—**state invariant vs transition rule vs policy vs assumption**—is one of the most important mental models in strong LLD and distributed systems design.



| Gap                                                                | What you are implicitly thinking                        | Why it is wrong for invariants                                                          | Fix / Correct mental model                                                          | Elevator example                                                                                                                 |
| ------------------------------------------------------------------ | ------------------------------------------------------- | --------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------- |
| **1. Not anchoring invariants to the state model**                 | "This seems like an important truth of the real world." | An invariant can only constrain variables that exist in the system state.               | First define `State S`. Every invariant is a predicate `P(S)` over those variables. | ❌ "Total people in building is conserved." (no people modeled) <br><br> ✅ `0 ≤ elevator.currentFloor < numFloors`                |
| **2. Confusing conservation laws with system invariants**          | "Things should balance over time."                      | Conservation is only valid if you model the entities being conserved.                   | Ask: "Do I have an explicit object whose count I track?"                            | In a passenger simulator: `waiting + riding + completed = total requests` is valid. In event-based elevator dispatch: invalid.   |
| **3. Confusing policies with correctness constraints**             | "This is how the system should behave."                 | Many policies can be correct. Invariants describe legal states independent of strategy. | Ask: "Could I swap the algorithm and still have a correct system?"                  | ❌ "Requests are handled FIFO." <br> ❌ "Nearest elevator must be chosen." <br> ✅ "A request is assigned to at most one elevator." |
| **4. Confusing assumptions with invariants**                       | "This is true in my simulation."                        | A model parameter can change without violating correctness.                             | Ask: "If I changed this value, would the system become illegal?"                    | ❌ "Elevator moves at constant speed." <br> ✅ "An elevator cannot move outside building floors."                                  |
| **5. Writing transition descriptions instead of state properties** | "When X happens, Y should occur."                       | A transition is a rule about change; an invariant is a rule about every snapshot.       | Convert verbs into predicates over current state.                                   | ❌ "When dispatched, the elevator owns the request." <br> ✅ "Each assigned request has exactly one owner."                        |
| **6. Mixing states that should be mutually exclusive**             | "The elevator moves while boarding."                    | States in a finite state machine should have clear boundaries.                          | Define legal combinations of states.                                                | ❌ `MOVING && DOORS_OPEN` <br> ✅ `MOVING ⇒ doors = CLOSED`                                                                        |
| **7. Not distinguishing liveness from safety**                     | "A request should eventually be served."                | This is about future progress, not current legality.                                    | Invariants are safety; eventual service is liveness.                                | ❌ "Every request gets served." (not an invariant) <br> ✅ "A request cannot be owned by two elevators."                           |
