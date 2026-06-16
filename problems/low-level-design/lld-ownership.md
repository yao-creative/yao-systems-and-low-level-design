Your intent is **formalizing software structure and ownership decisions**: "how do I decide where state lives, who can mutate it, and how objects/modules should relate?"

The central idea is:

$$
\text{Structure} = \text{Dependency Graph}
$$

$$
\text{Ownership} = \text{Authority over State Transitions}
$$

A good design minimizes the number of nodes that can perform a transition:

$$ 
\text{Mutation Surface Area} \downarrow \Rightarrow \text{Correctness} \uparrow 
$$

---

# 1. Ownership Formalisms (Who may change state?)

## A. Information Expert (GRASP)

### Rule

> Place behavior where the information needed to perform it already exists.

### Use when

* One object already has the data required for a decision.

### Example

Parking:

```
ParkingFloor
    spots[]
```

Therefore:

```
ParkingFloor.findAvailableSpot()
```

not:

```
ParkingLot.scanAllSpots()
```

because the floor is the local expert.

---

## B. Encapsulation / Representation Hiding

### Rule

> The owner of state is the only object allowed to mutate it.

### Use when

* The state has invariants.

Example:

```
ParkingSpot
---------
occupant
```

Invariant:

```
occupant ∈ {None, Vehicle}
```

Good:

```
spot.assign(vehicle)
spot.release()
```

Bad:

```python
spot.occupant = vehicle
```

---

## C. Aggregate (DDD)

### Rule

> Group state that must remain consistent under one transaction.

Formal:

```
Aggregate = Consistency Boundary
```

Use when:

```
A and B must always satisfy invariant I
```

then:

```
A and B belong to the same aggregate.
```

---

### Example

Order system:

```
Order
 |
 +-- OrderItem[]
```

Invariant:

```
order.total = sum(items.price)
```

Therefore:

```
Order.addItem()
Order.removeItem()
```

owns item mutation.

---

## D. Controller Pattern

### Rule

> A separate object coordinates a multi-object use case.

Use when a transition touches many aggregates.

Example:

```
ParkingLot.park(vehicle)

1. Ask floor for spot
2. Assign spot
3. Create ticket
4. Update indexes
```

No single spot knows the whole workflow.

---

# Ownership Decision Table

| Situation                                | Owner                          |
| ---------------------------------------- | ------------------------------ |
| Data needed is local                     | Information Expert             |
| State has invariants                     | Encapsulate in the owner       |
| Several objects share a consistency rule | Aggregate root                 |
| Operation spans many objects             | Controller/Application Service |
| External side effects (payment, email)   | Service                        |
| Shared read-only data                    | Value Object                   |

---

# 2. Structural Formalisms (How things are connected?)

Think of your code as a graph:

```
Objects/modules = vertices

uses/owns/calls = edges
```

The goal is to control the graph.

---

# A. Composition (Strong Ownership)

Formal:

```
A contains B
```

Lifetime:

```
destroy(A) → destroy(B)
```

Use when:

* B has no independent meaning.
* A controls B's lifecycle.

Example:

```python
class ParkingSpot:
    def __init__(self, spot_id: str):
        self.spot_id = spot_id

class ParkingFloor:
    def __init__(self, floor_id: int, num_spots: int):
        # Composition: Spots are created here and tied to Floor's lifetime
        self.spots = [ParkingSpot(f"{floor_id}-{i}") for i in range(num_spots)]
```

A spot does not exist without a floor.

---

# B. Aggregation (Weak Ownership)

Formal:

```
A references B
```

but:

```
B exists independently
```

Use when:

* Objects have independent lifecycles.

Example:

```python
class Player:
    def __init__(self, name: str):
        self.name = name

class Team:
    def __init__(self, team_name: str):
        self.players = [] # Weak ownership (references)

    def add_player(self, player: Player):
        self.players.append(player)
```

A player can move teams.

---

# C. Association

General relationship.

```
A <----> B
```

Use when:

* Objects collaborate.
* Neither owns the other.

Example:

```python
class Payment:
    def __init__(self, amount: float):
        self.amount = amount

class ParkingTicket:
    def __init__(self, ticket_id: str):
        self.ticket_id = ticket_id
        self.payment = None # Association: Ticket and Payment collaborate
```

---

# D. Dependency

Formal:

```
A → B
```

means:

```
A requires B's interface
```

Goal:

```
Stable modules ← dependencies
```

Use interfaces when the implementation may vary.

Example:

```python
class ParkingLot:
    def __init__(self, fee_strategy):
        # Dependency: ParkingLot depends on a FeeStrategy instance
        self.fee_strategy = fee_strategy

    def calculate(self, hours):
        return self.fee_strategy.calculate(hours)
```

---

# E. Interface / Abstraction Boundary

Formal:

```
Behavior Contract
```

Use when:

* Multiple implementations are possible.
* You want to vary policy.

Example:

```python
from abc import ABC, abstractmethod

class FeeStrategy(ABC):
    @abstractmethod
    def calculate(self, hours): pass

class HourlyFee(FeeStrategy):
    def calculate(self, hours): return hours * 5.0

class DailyFee(FeeStrategy):
    def calculate(self, hours): return 20.0
```

---

# Structure Decision Table

| Situation                          | Relationship   |
| ---------------------------------- | -------------- |
| Child has no independent existence | Composition    |
| Child has independent lifecycle    | Aggregation    |
| Objects simply interact            | Association    |
| Need to swap implementations       | Interface      |
| Module only requires behavior      | Dependency     |
| Need to isolate external systems   | Adapter/Facade |

---

# 3. Combining Ownership + Structure

The strongest design heuristic:

| Question                                     | Formal concept       |
| -------------------------------------------- | -------------------- |
| Who knows the data?                          | Information Expert   |
| Who protects invariants?                     | Encapsulation        |
| What must update atomically?                 | Aggregate            |
| Who coordinates a workflow?                  | Controller           |
| Who owns lifetime?                           | Composition          |
| Who merely references?                       | Aggregation          |
| What can change independently?               | Interface boundary   |
| What should not know implementation details? | Dependency inversion |

---

# The "object graph + state machine" view

A mature designer thinks of the system as:

```
          Commands
             |
             v
       Controller
             |
             v
       Aggregate Root
             |
        owns state
             |
      Object Graph
             |
        Invariants
```

or mathematically:

$$
\text{Transition}
\to
\text{Authority}
\to
\text{Owned State}
\to
\text{Invariant Preservation}
$$

---

## Ultimate compression

For almost every LLD decision, ask these five questions in order:

1. **Where does the state naturally belong?**
   → Information Expert

2. **Who must guarantee its correctness?**
   → Encapsulation

3. **What data must change together?**
   → Aggregate

4. **Who orchestrates the use case?**
   → Controller/Service

5. **What is the lifetime relationship?**
   → Composition vs Aggregation

This is arguably the closest practical "decision algorithm" for object ownership and structure in real-world software design.
