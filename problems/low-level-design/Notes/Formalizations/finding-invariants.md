The way to find LLD invariants systematically is to treat the system as a **transition system** and work backwards from **illegal world states**. The key is that invariants are not something you brainstorm randomly; they emerge from your **domain model, ownership boundaries, and state machine**.

The process can be formalized as follows.

---

# The LLD Invariant Discovery Pipeline

```
Requirements
     ↓
Choose abstraction boundary
     ↓
Define state model S
     ↓
Identify domain entities and relationships
     ↓
Find illegal states
     ↓
Convert illegal states into predicates P(S)
     ↓
Ensure every operation preserves P(S)
```

Formally:

[
\forall op,\quad P(S_{before}) \Rightarrow P(op(S_{before}))
]

---

# Step 1 — Fix the abstraction boundary

Ask:

> "What does this system actually remember?"

Everything outside this boundary cannot appear in your invariant.

---

### Elevator example

Scope says:

```
We model:
- Elevators
- Requests
- Floors
- Directions
- Doors
- Capacity

We do NOT model:
- Individual people
- Continuous physics
```

Therefore valid invariants can mention:

```
elevator.currentFloor
elevator.state
request.assignedElevator
occupancy
```

But not:

```
numberOfHumans
walkingTime
acceleration
```

---

# Step 2 — Write the system state

Explicitly list your state variables.

Example:

```java
Elevator {
    floor
    direction
    state
    doorState
    capacity
    occupancy
    assignedRequests
}

Request {
    sourceFloor
    destinationFloor
    status
    assignedElevator
}
```

This gives you a search space.

---

# Step 3 — Apply the invariant categories checklist

Almost every LLD invariant falls into a small number of buckets.

## A. Range / validity constraints

Question:

> "Can this value ever be out of legal bounds?"

Pattern:

[
min \le x \le max
]

Examples:

```
0 <= currentFloor < numberOfFloors
```

```
0 <= occupancy <= capacity
```

---

## B. State machine legality

Question:

> "Are there combinations of states that make no sense?"

Pattern:

```
state A implies constraint B
```

Examples:

```
MOVING → doors == CLOSED
```

```
DOOR_OPEN → elevator.state == STOPPED
```

---

## C. Ownership / uniqueness

Question:

> "Can two things own the same resource?"

Pattern:

[
count(owners(x)) \leq 1
]

Examples:

```
A request belongs to at most one elevator
```

Parking lot:

```
A parking spot has at most one vehicle
```

---

## D. Relationship consistency

Question:

> "If A points to B, does B agree?"

Pattern:

```
A references B ↔ B references A
```

Examples:

Elevator:

```
request.assignedElevator = E
⇒
request ∈ E.assignedRequests
```

Parking:

```
ticket.spotId = S
⇒
spot.currentTicket = ticket
```

---

## E. Conservation / accounting

Question:

> "Is there something being created and destroyed?"

Pattern:

```
inputs + current = outputs
```

Examples:

Banking:

```
Total money remains constant
```

Inventory:

```
stock + shipped = produced
```

Elevator (only if passengers modeled):

```
waiting + riding + completed = total generated passengers
```

---

## F. Ordering constraints

Question:

> "Can time or sequence go backwards?"

Pattern:

[
t_1 \le t_2
]

Examples:

```
entryTime <= exitTime
```

```
request.creationTime <= completionTime
```

---

# Step 4 — Reject false invariants

Every candidate should pass this filter.

| Question                                    | If "No" | It is actually          |
| ------------------------------------------- | ------- | ----------------------- |
| Is it a predicate over current state?       | Reject  | Workflow/transition     |
| Does the state contain these variables?     | Reject  | Outside scope           |
| Must it hold regardless of algorithm?       | Reject  | Policy                  |
| Is it true at every instant?                | Reject  | Liveness property       |
| Would violating it create an illegal state? | Reject  | Optimization/assumption |

---

# Step 5 — Derive operations from invariants

A strong LLD engineer actually uses invariants to design APIs.

Example:

Invariant:

```
occupancy <= capacity
```

Therefore:

```java
enterPassenger() {
    if (occupancy == capacity)
        reject();
    occupancy++;
}
```

---

Invariant:

```
Request has one owner
```

Therefore:

```java
assign(request, elevator) {
    assert request.assignedElevator == null;
    request.assignedElevator = elevator;
}
```

---

# The 60-second interview checklist

When asked "give me invariants", run this mentally:

```
1. What objects exist?
       ↓
2. What fields can be invalid?
       ↓
3. What states cannot coexist?
       ↓
4. What resources require unique ownership?
       ↓
5. What references must be consistent?
       ↓
6. What quantities must be conserved?
       ↓
7. What ordering must never be violated?
```

---

# The deepest mental model

A good way to remember the distinction:

| Concept             | Question                                         |
| ------------------- | ------------------------------------------------ |
| Invariant           | "What states must be impossible?"                |
| State transition    | "How do I move from one valid state to another?" |
| Policy              | "Which valid action do I choose?"                |
| Liveness            | "What eventually happens?"                       |
| Modeling assumption | "What world am I pretending exists?"             |

The strongest LLD candidates usually derive invariants from **entity relationships, ownership, and state machines**, not from the sequence of features. That is the same mathematical reasoning used in database constraints, distributed systems, and formal verification.
