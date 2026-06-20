## 1. Core idea

The **State Pattern** is an object-oriented implementation of a **finite state machine (FSM)** where each state is represented as an object, and the **transition logic and state-specific behavior are encapsulated inside the state objects**.

The intent from the GoF pattern:

> Allow an object to alter its behavior when its internal state changes. The object appears to change its class.

---

# 2. Formal mathematical model

A finite state machine can be modeled as:

$$
FSM = (S, E, T, A)
$$

where:

* $S$: finite set of states
* $E$: set of events/inputs
* $T: S \times E \rightarrow S$: transition function
* $A: S \times E \rightarrow Behavior$: action/output function

Example: a turnstile.

### States

$$
S = {\text{Locked}, \text{Unlocked}}
$$

### Events

$$
E = {\text{Coin}, \text{Push}}
$$

### Transition table

| Current State | Event | Next State | Action        |
| ------------- | ----- | ---------- | ------------- |
| Locked        | Coin  | Unlocked   | Unlock gate   |
| Locked        | Push  | Locked     | Reject entry  |
| Unlocked      | Coin  | Unlocked   | Return coin   |
| Unlocked      | Push  | Locked     | Allow passage |

This table completely defines the machine.

---

# 3. Naive implementation (conditional state machine)

A common implementation is a giant `switch`.

```python
class Turnstile:
    def __init__(self):
        self.state = "LOCKED"

    def coin(self):
        if self.state == "LOCKED":
            self.unlock()
            self.state = "UNLOCKED"
        else:
            self.return_coin()

    def push(self):
        if self.state == "LOCKED":
            self.alarm()
        else:
            self.lock()
            self.state = "LOCKED"
```

### Problem

The object knows:

* every possible state
* every transition
* every state-specific behavior

As the number of states grows:

$$
Complexity \approx O(|S| \times |E|)
$$

The transition logic becomes a large conditional matrix.

---

# 4. State Pattern decomposition

The key transformation is:

## Before

```
                Turnstile
                    |
          +---------+----------+
          |                    |
      state data          all transition rules
```

One class contains the entire state graph.

---

## After

```
                    Turnstile (Context)
                            |
                            v
                        State
                          |
          +---------------+----------------+
          |                                |
      LockedState                    UnlockedState
```

Responsibilities:

### Context

Holds:

* current state
* shared resources
* delegates events

### State object

Defines:

* legal events
* state-specific behavior
* transitions to next states

---

# 5. State Pattern implementation

## State interface

```python
from abc import ABC, abstractmethod

class State(ABC):
    @abstractmethod
    def coin(self, turnstile):
        pass

    @abstractmethod
    def push(self, turnstile):
        pass
```

---

## Locked state

```python
class LockedState(State):

    def coin(self, turnstile):
        print("Unlocking")
        turnstile.state = UnlockedState()

    def push(self, turnstile):
        print("Alarm!")
```

---

## Unlocked state

```python
class UnlockedState(State):

    def coin(self, turnstile):
        print("Returning coin")

    def push(self, turnstile):
        print("Passing through")
        turnstile.state = LockedState()
```

---

## Context

```python
class Turnstile:

    def __init__(self):
        self.state = LockedState()

    def coin(self):
        self.state.coin(self)

    def push(self):
        self.state.push(self)
```

---

# 6. The key inversion of control

In the naive design:

```
Turnstile
    |
    +-- "If I am locked and receive coin,
         go to unlocked."
```

The context controls the states.

---

In the State Pattern:

```
LockedState
    |
    +-- "When I receive coin,
         I become UnlockedState."
```

The states control themselves.

This is **encapsulation of transition legality**.

> **Inversion of control, full graph**
>
> Before inversion of control, the context owns the full transition graph.
>
> ```
> Context / Turnstile
>   |
>   +-- current_state = Locked
>   +-- transition(Locked, Coin) = Unlocked
>   +-- transition(Locked, Push) = Locked
>   +-- transition(Unlocked, Coin) = Unlocked
>   +-- transition(Unlocked, Push) = Locked
> ```
>
> After inversion of control, the context owns only the current pointer.
> Each state owns its outgoing edges.
>
> ```
> Context / Turnstile
>   |
>   +-- current_state ----> LockedState
>
> LockedState
>   +-- coin() -> UnlockedState
>   +-- push() -> LockedState
>
> UnlockedState
>   +-- coin() -> UnlockedState
>   +-- push() -> LockedState
> ```
>
> Full-graph shift:
> before: `Context -> all transitions`
> after: `Context -> current_state`, `State -> outgoing transitions`
>
> Category-theory style span view:
>
> Before:
>
> ```
>          owned by Context
>
> States  <---src---  Transitions  ---dst--->  States
>   S                    T = all edges           S
>
> where Context owns the whole span:
> Context -> (S <- T -> S)
> ```
>
> After:
>
> ```
> Context -> current_state in S
>
> For each state s:
>
>   {s}  <---src---  Out(s)  ---dst--->  States
>
> where State(s) owns only its local span:
> State(s) -> ({s} <- Out(s) -> S)
> ```
>
> So the decomposition is:
> `S <- T -> S`
> becomes
> `Context -> current_state`
> plus
> `coproduct over s in S of ({s} <- Out(s) -> S)`.

---

# 7. Relation to OOP principles

## Encapsulation

Each state owns:

* its valid operations
* its transition rules
* its behavior

Example:

```
LockedState:
    coin()  ✓
    push()  ✗ (alarm)
```

The legality is local.

---

## Open/Closed Principle

Add a new state:

```
MaintenanceState
```

without modifying existing states.

You extend the graph by adding a node.

---

## Polymorphism

Instead of:

```python
if state == "LOCKED":
    ...
elif state == "UNLOCKED":
    ...
```

You use dynamic dispatch:

```python
state.coin()
```

The correct behavior is selected by the runtime type.

---

# 8. A deeper way to think about it

The State Pattern turns a **transition table**:

```
             Event
          Coin   Push
State
Locked    S2     S1
Unlocked  S2     S1
```

into an **object graph**:

```
            State
              |
      +-------+-------+
      |               |
   Locked         Unlocked

Locked:
    coin() -> Unlocked
    push() -> Locked

Unlocked:
    coin() -> Unlocked
    push() -> Locked
```

The transition matrix is distributed across the state objects.

---

# 9. When to use it

Use the State Pattern when:

* The object has **a finite number of meaningful states**
* Behavior changes drastically by state
* There are many `if state == ...` conditionals
* You need to enforce legal transitions
* You expect new states to be added

Examples:

* Orders: `Created → Paid → Shipped → Delivered`
* Network connections: `Disconnected → Connecting → Connected`
* TCP protocol states
* Elevator modes
* Vending machines
* Game character states (idle, running, attacking)

---

## The deepest formalization

A normal object is:

$$
Object = Data + Methods
$$

A stateful domain object is:

$$
Object = (CurrentState,\ TransitionFunction,\ Invariants)
$$

The **State Pattern** decomposes:

$$
TransitionFunction = \bigcup_{s \in S} TransitionRules(s)
$$

meaning each state object owns the subset of the transition function that originates from itself.

This is why the pattern scales: **you partition the state machine by state, making each state the authority over its outgoing edges.**


# Algorithm for Construction

Yes. In the classic **State Pattern**, there is usually **one class (or object type) per meaningful state**. More precisely:

$$
\text{FSM State } s_i \in S \quad \longrightarrow \quad \text{Concrete State class } C_i
$$

So the **state graph becomes an object graph**.

For example:

```
Order FSM

Created
   |
 pay()
   v
Paid
   |
 ship()
   v
Shipped
```

becomes:

```
             State
               |
    ----------------------
    |          |          |
CreatedState PaidState ShippedState
```

where:

```python
class CreatedState:
    def pay(self, order):
        order.state = PaidState()

    def ship(self, order):
        raise InvalidTransition()
```

---

# Algorithm for constructing a State Pattern

You can think of it as a **mechanical transformation from an FSM specification to an OOP design**.

## Step 1: Enumerate the state space (S)

Ask:

> What are the qualitatively different modes where the object obeys different rules?

Example:

```
Order:
- Created
- Paid
- Shipped
- Cancelled
```

Do **not** create states for incidental data:

Bad:

```
UserAge17State
UserAge18State
```

Good:

```
UnverifiedUser
VerifiedUser
BannedUser
```

The key criterion:

[
\text{Different state} \iff \text{different legal operations or behaviors}
]

---

## Step 2: Identify events/commands (E)

Ask:

> What can the outside world ask this object to do?

Example:

```
pay()
ship()
cancel()
refund()
```

These become the **State interface**.

```python
class OrderState:
    def pay(order): ...
    def ship(order): ...
    def cancel(order): ...
```

---

## Step 3: Build the transition table (T)

This is the most important design artifact.

| Current | Event  | Next      | Behavior            |
| ------- | ------ | --------- | ------------------- |
| Created | pay    | Paid      | charge card         |
| Created | cancel | Cancelled | release reservation |
| Paid    | ship   | Shipped   | schedule shipment   |
| Shipped | refund | Error     | not allowed         |

This table is your specification.

---

## Step 4: Create one concrete class per row group

Partition the transition table by **source state**.

For `Created`:

```
CreatedState

pay():
    charge card
    -> PaidState

cancel():
    release inventory
    -> CancelledState

ship():
    invalid
```

For `Paid`:

```
PaidState

ship():
    create shipment
    -> ShippedState

cancel():
    issue refund
    -> CancelledState
```

This is the key mathematical transformation:

[
T = \bigcup_{s \in S} T_s
]

where (T_s) contains all outgoing transitions from state (s).

Each class owns one (T_s).

---

## Step 5: Create the Context object

The domain object keeps a pointer to the current state.

```
Order
 |
 +-- state: OrderState
 +-- items
 +-- paymentInfo
 +-- customer
```

It delegates:

```python
def ship():
    state.ship(self)
```

The context does **not** ask:

```python
if status == "PAID":
    ship()
```

That would re-centralize the FSM.

---

## Step 6: Decide where data lives

A subtle but important rule:

```
Context = long-lived data
State = behavior + transition logic
```

Example:

```
Order
 ├── id
 ├── items
 ├── customer
 └── state --> PaidState
```

Not:

```
PaidState
 ├── customer
 ├── items
 ├── payment history
```

State objects are usually lightweight.

---

# The design heuristic

The thought process is:

```
Does behavior change by mode?
        |
       Yes
        |
Do I have many conditionals on state?
        |
       Yes
        |
Can I model a finite set of modes?
        |
       Yes
        |
Create an FSM
        |
Enumerate states S
        |
Enumerate events E
        |
Build transition table T
        |
Partition T by source state
        |
Create one class per partition
        |
Delegate from context to state
```

---

# A deeper connection: "classes as regions of a transition graph"

The State Pattern is essentially **graph decomposition**.

You start with a directed graph:

```
Created ----pay----> Paid ----ship----> Shipped
   |
cancel
   v
Cancelled
```

Then assign each node ownership of its outgoing edges:

```
CreatedState
   owns:
       pay -> PaidState
       cancel -> CancelledState

PaidState
   owns:
       ship -> ShippedState
```

So a very precise way to understand the pattern is:

> **A State Pattern is a decomposition of a finite state machine where each state object is responsible for the outgoing transition function from that state.**

This framing is also why it appears heavily in protocol design, operating systems, networking, compilers, and complex domain models.
