This is a question of **where an invariant should live**. The general principle from **domain-driven design (DDD)** and object-oriented design is:

> Put an invariant where the information and authority needed to enforce it naturally resides.

For **global constraints**, the answer is usually: **the aggregate/context, not the individual state**.

---

## 1. State-specific legality → State object

The State Pattern should own **behavior that depends on the current state**.

Example: elevator modes.

```text
Current State: Moving

Events:
- stop()    → allowed
- openDoor() → forbidden
```

The `MovingState` knows:

```text
Moving --stop--> Idle
Moving --openDoor--> illegal
```

So it makes sense for the state object to enforce it.

---

## 2. Global invariants → Aggregate/Context

Suppose an elevator has:

```text
MIN_FLOOR = 1
MAX_FLOOR = 100
```

The invariant:

[
1 \leq \text{currentFloor} \leq 100
]

is true **regardless of whether the elevator is Idle, Moving, or Maintenance**.

It is not a property of any particular state.

So the aggregate owns it:

```
          Elevator (Aggregate)
          /                  \
   Global Invariants       Current State
    floor bounds             Idle
    capacity                 Moving
    safety rules             Maintenance
```

Example:

```python
class Elevator:
    def move_to(self, floor):
        if floor < 1 or floor > 100:
            raise InvalidFloor()

        self.state.move_to(self, floor)
```

Then:

```python
class MovingState:
    def move_to(self, ctx, floor):
        ctx.current_floor = floor
```

The state decides **whether movement is allowed**.

The aggregate decides **whether the requested floor is valid**.

---

# 3. Shared invariants should not be duplicated in states

A bad design:

```text
IdleState:
    check floor bounds

MovingState:
    check floor bounds

MaintenanceState:
    check floor bounds
```

Problems:

* Duplicate logic
* Risk of inconsistent rules
* Harder to change (e.g., 100 floors → 200 floors)

The invariant has the wrong owner.

---

# 4. In DDD terms: the aggregate is the consistency boundary

An aggregate is responsible for ensuring:

> "No externally visible state of this entity violates business rules."

So every mutation should pass through the aggregate boundary:

```
Command
   |
   v
Aggregate
   |
   +-- Check global invariants
   |
   +-- Delegate state-dependent behavior
          |
          v
       State object
```

This gives a very clean separation:

| Responsibility                        | Owner                     |
| ------------------------------------- | ------------------------- |
| Is this event legal in this mode?     | State                     |
| Are all domain invariants maintained? | Aggregate                 |
| Cross-field constraints               | Aggregate                 |
| Transition logic                      | State or transition table |
| Temporal synchronization              | Lock/transaction          |

---

## 5. A more advanced rule: layered guards

In sophisticated systems (trading engines, kernels, robotics), you often see **multiple layers of invariants**:

```
Mutex
  ↓
Aggregate invariant checks
  ↓
Current state transition rules
  ↓
Resource/physics constraints
  ↓
State mutation
```

For example, a robot:

* `WalkingState`: "Can I start running?"
* `Robot`: "Is battery > 10%?"
* `MotorController`: "Is torque within safe limits?"

Each layer enforces the invariant closest to its own authority.

---

So for your example of **global movement bounds**, the most principled design is:

**Keep movement bounds in the aggregate/context. Let the state object decide whether movement is allowed in the current mode.**

This is exactly the distinction between a **state-dependent transition invariant** and a **global aggregate invariant**.
