
# Formalization of State choices.

Your question touches a very important LLD modeling concept: **state decomposition**. The mistake many people make is to put every adjective into a single enum:

```text
ElevatorState =
    IDLE,
    MOVING_UP,
    MOVING_DOWN,
    DOOR_OPEN,
    BOARDING,
    ALIGHTING
```

This looks reasonable but is actually a poor state machine because it **mixes orthogonal dimensions** of state.

---

# 1. Formalism: State = Cartesian product of independent dimensions

A good state machine starts by asking:

> What independent aspects of reality can vary separately?

Mathematically:

$$
S = S_1 \times S_2 \times \cdots \times S_n
$$

where each $S_i$ is an independent state dimension.

For an elevator:

$$
ElevatorState = MotionState \times DoorState \times ServiceState
$$


---

# 2. Dimension 1 — Motion State (where is the elevator moving?)

This answers:

> What is the physical movement status?

```java
enum MotionState {
    IDLE,
    MOVING_UP,
    MOVING_DOWN
}
```

Invariant:

$$
MOVING \Rightarrow Door = CLOSED
$$

---

# 3. Dimension 2 — Door State

This answers:

> What is the door mechanism doing?

```java
enum DoorState {
    OPEN,
    CLOSED,
    OPENING,
    CLOSING
}
```

Why include transitions?

Because these are real states with different behavior.

Example:

```
OPENING
   |
   v
OPEN
   |
   v
CLOSING
```

You may decide:

```
OPENING/CLOSING ⇒ no movement
```

---

# 4. Dimension 3 — Service State

This answers:

> What is the elevator doing with requests?

A simple model:

```java
enum ServiceState {
    IDLE,
    SERVING_REQUEST
}
```

or a richer model:

```java
enum StopActivity {
    NONE,
    BOARDING,
    ALIGHTING
}
```

---

# 5. Should boarding and alighting be states?

This is where modeling judgment matters.

## Option A: Model them as states

Good if:

* you simulate time,
* need capacity updates,
* have passenger flows.

Example:

```
DOOR_OPEN
    |
    +-- BOARDING
    |
    +-- ALIGHTING
```

Useful for:

* train simulation,
* robotics,
* traffic simulation.

---

## Option B: Model them as events (better for most LLD interviews)

Because your requirements say:

> Do not model passengers.

Then:

```
arrive floor
      |
      v
OPEN_DOOR
      |
      v
process requests
      |
      v
CLOSE_DOOR
```

Boarding and alighting are not persistent states.

They are operations.

---

# 6. A good elevator model for this interview

## State Variables

```
Elevator
├── currentFloor
├── targetFloor
├── direction
├── motionState
├── doorState
├── assignedRequests
└── occupancy
```

---

## Main state machine

### Motion

```
           request assigned
IDLE ------------------------> MOVING_UP
  ^                                |
  |                                |
  |                                v
  +--------------------------- MOVING_DOWN
           no pending requests
```

---

### Door

```
CLOSED
   |
   v
OPENING
   |
   v
OPEN
   |
   v
CLOSING
   |
   v
CLOSED
```

---

# 7. The "single enum smell"

Bad:

```java
enum ElevatorState {
    IDLE,
    MOVING_UP,
    MOVING_DOWN,
    OPEN,
    BOARDING,
    ALIGHTING
}
```

Why?

Because it loses information.

Question:

> Is the elevator idle with doors open?

You cannot represent it.

---

Question:

> Is it moving up while the doors are opening?

You cannot express that it is invalid.

---

# 8. Invariants become easier with decomposed state

With a decomposed state:

### Door-motion safety

$$
Motion \in {UP, DOWN}
\Rightarrow Door=CLOSED
$$

---

### Movement target


$$
Motion \neq IDLE
\Rightarrow targetFloor \neq null
$$

---

### Stop behavior

$$
Door=OPEN
\Rightarrow Motion=IDLE
$$

---

# 9. Compression matrix: What should be a state?

Use this interview heuristic.

| Candidate            | State                 | Event     | Attribute | Reason                                         |
| -------------------- | --------------------- | --------- | --------- | ---------------------------------------------- |
| Moving up/down       | ✅                     | ❌         | ❌         | Persistent behavior affecting transitions      |
| Idle                 | ✅                     | ❌         | ❌         | Persistent mode                                |
| Door open/closed     | ✅                     | ❌         | ❌         | Long-lived physical condition                  |
| Door opening/closing | ✅ (if timing matters) | ⚠️        | ❌         | Transitional physical state                    |
| Boarding             | ⚠️                    | ✅ usually | ❌         | In your scope no passenger simulation          |
| Alighting            | ⚠️                    | ✅ usually | ❌         | Same reason                                    |
| Direction            | ❌                     | ❌         | ✅         | Derivable from target vs current floor (often) |
| Current floor        | ❌                     | ❌         | ✅         | Continuous attribute, not a mode               |
| Capacity             | ❌                     | ❌         | ✅         | Constraint parameter                           |
| Occupancy            | ❌                     | ❌         | ✅         | Numeric quantity                               |
| Request assignment   | ❌                     | ❌         | ✅         | Relationship data                              |

---

# 10. Senior-level rule of thumb

A very useful hierarchy:

$$
\text{Entity State} = 
\text{Mode}
+
\text{Attributes}
+
\text{Relationships}

$$ 

Where:

| Category                        | Example                                 |
| ------------------------------- | --------------------------------------- |
| **Mode (finite state machine)** | Moving, idle, doors open                |
| **Attributes (values)**         | Floor number, occupancy                 |
| **Relationships**               | Assigned requests, current passengers   |
| **Events (transitions)**        | Arrive floor, open door, assign request |

For LLD interviews, the strongest designs usually use **multiple orthogonal state machines** rather than one giant enum. This gives you cleaner invariants, simpler transitions, and a much more extensible design.
