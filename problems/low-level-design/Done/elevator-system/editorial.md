In Low-Level Design (LLD), it helps to separate three layers that people often blur:

> **Intent → domain model → coordination mechanism**

Your question (“are they just methods in the aggregate?”) is pointing at the middle layer, but real systems split behavior across multiple patterns.

I’ll ground this using **Parking Lot** and **Elevator**, because they expose the full structure.

---

# 1. First-principles model (what LLD is really doing)

An LLD system is:

[
\text{State} + \text{Events} + \text{Rules} + \text{Orchestration}
]

So behavior is NOT “just methods in one object”.

Instead:

| Concern               | Typical location          |
| --------------------- | ------------------------- |
| invariants            | Aggregate (domain object) |
| transitions           | State / Strategy          |
| sequencing            | Controller / Service      |
| external coordination | Manager / System          |

---

# 2. Aggregate vs Service vs Controller (core distinction)

## Aggregate (domain consistency boundary)

> owns invariants, not orchestration

Example responsibilities:

* “Can I park this car?”
* “Can I move from Idle → Moving?”

✔ holds rules
❌ should NOT run global flow

---

## Service / Manager (orchestration layer)

> decides *what happens next*

* assigns parking spot
* dispatches elevator requests
* routes events

✔ controls sequence
❌ should not embed deep rules

---

## Controller (entry point / API layer)

> translates external input → domain events

* user presses button
* car enters lot

---

# 3. Parking Lot LLD (clean decomposition)

## Core entities

* `ParkingLot` (aggregate root)
* `Level`
* `Spot`
* `Vehicle`
* `Ticket`

---

## Pattern mapping

### (A) Aggregate pattern

`ParkingLot` is NOT just a bag of methods — it enforces invariants:

```python
class ParkingLot:
    def can_park(self, vehicle):
        return any(level.has_space(vehicle) for level in self.levels)

    def park(self, vehicle):
        for level in self.levels:
            if level.has_space(vehicle):
                return level.assign(vehicle)
```

👉 Aggregate = **invariant guard + local decision making**

---

### (B) Strategy pattern (VERY common here)

Different parking logic:

* nearest spot
* cheapest spot
* size-based allocation

```python
class NearestSpotStrategy:
    def select(self, levels, vehicle):
        ...
```

👉 Strategy = “pluggable rule for decision making”

---

### (C) Factory pattern

Vehicle / ticket creation:

```python
class TicketFactory:
    def create(vehicle, spot):
        return Ticket(...)
```

---

### (D) Repository pattern

Persistence abstraction:

```python
class TicketRepository:
    def save(ticket): ...
```

---

### (E) Service layer (orchestrator)

```python
class ParkingService:
    def park_vehicle(self, vehicle):
        spot = strategy.select(...)
        ticket = factory.create(vehicle, spot)
        repo.save(ticket)
```

👉 THIS is where most flow logic lives

---

# 4. Elevator LLD (more interesting: it is an event system)

Elevator is basically:

> a **state machine + scheduler + event-driven controller**

---

## Core entities

* ElevatorCar (aggregate)
* ElevatorController (system brain)
* RequestQueue (event buffer)
* Direction state machine

---

## (A) State Pattern (VERY central)

Elevator is a canonical state machine:

States:

* Idle
* MovingUp
* MovingDown
* DoorOpen

```python
class State:
    def handle(self, elevator, event):
        pass
```

👉 Each state defines allowed transitions

This is NOT just methods in aggregate — it is **Strategy-as-State**

---

## (B) Strategy pattern (dispatch logic)

How to choose next floor:

* FCFS
* LOOK / SCAN algorithm
* priority-based scheduling

```python
class ScanScheduler:
    def next(self, queue, current_floor):
        ...
```

👉 This is the “brain” of movement optimization

---

## (C) Command pattern (button presses)

Each request is an event object:

```python
class FloorRequest:
    floor: int
    direction: Up/Down
```

👉 Commands = event representation

---

## (D) Observer pattern (buttons → controller)

* button press notifies controller
* controller updates queue

---

## (E) Controller / Dispatcher (core orchestrator)

```python
class ElevatorController:
    def step(self):
        request = scheduler.next(queue)
        elevator.move(request)
```

👉 This is the “event loop”

---

# 5. So are these “just methods in aggregate”?

## Answer: no — because responsibilities split like this:

---

## 1. Aggregate = correctness boundary

* enforces invariants
* ensures valid transitions

---

## 2. Strategy = decision logic

* “which move should happen next?”

---

## 3. State pattern = transition rules

* “what does this event do in this state?”

---

## 4. Service/Controller = orchestration

* “in what order do we apply things?”

---

# 6. Clean mental model (very important)

Think of LLD as 4 orthogonal axes:

### (1) RULES

→ Aggregate / State

### (2) DECISIONS

→ Strategy

### (3) FLOW

→ Service / Controller

### (4) EVENTS

→ Command / Request objects

---

# 7. Elevator vs Parking Lot (why they differ)

| Aspect            | Parking Lot          | Elevator          |
| ----------------- | -------------------- | ----------------- |
| structure         | allocation system    | dynamic scheduler |
| core pattern      | Strategy + Aggregate | State + Scheduler |
| complexity source | capacity constraints | time + ordering   |
| event-driven?     | mild                 | strong            |

---

# 8. Key insight (what interviewers actually test)

They are not testing “do you know patterns”.

They are testing:

> Can you separate:

* invariants (aggregate)
* decisions (strategy)
* transitions (state machine)
* orchestration (service)

---

# 9. One-line compression

* Aggregate = “what is valid”
* State = “what is currently true”
* Strategy = “what should I do next”
* Controller = “when do I do it”

---

If you want next step, I can:

* design a **fully clean elevator system with event-driven architecture**
* or show **common LLD mistakes (where everything gets shoved into one class)**
* or map both problems into a **formal state machine + reducer model (very powerful for interviews)**
