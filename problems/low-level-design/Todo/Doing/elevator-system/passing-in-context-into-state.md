Yes. In the classic State Pattern, the state object receives a **reference to the context/aggregate** and mutates the context's current state. This is a form of **double dispatch**: the context delegates the operation to the current state, and the state decides how the context should evolve.

Let's trace the call stack.

---

## Example: Idle → Moving transition

### 1. User sends a command

```python
elevator.request_move(10)
```

Call stack:

```
request_move()
```

---

### 2. Elevator validates global invariants

```python
class Elevator:
    def request_move(self, floor):
        if floor < 1 or floor > 100:
            raise InvalidFloor()

        self.state.move(self, floor)
```

At this moment:

```
Elevator
 ├── floor = 3
 ├── state = Idle()
 └── target = None
```

The stack:

```
request_move()
    |
    v
Idle.move(elevator, 10)
```

The context (`self`) is passed into the state method.

---

### 3. State object decides transition

```python
class Idle(State):
    def move(self, elevator, target):
        elevator.target = target
        elevator.state = Moving()
```

The state object has access to the **aggregate's mutable reference**.

Before:

```
Elevator
 └── state → Idle object
```

After:

```
Elevator
 └── state → Moving object
```

The call stack is still:

```
request_move()
    |
    +--> Idle.move()
           |
           +--> elevator.state = Moving()
```

No object is returned. The mutation happens through the reference.

---

### 4. Control returns

The stack unwinds:

```
Idle.move() returns
        |
request_move() returns
        |
caller resumes
```

Now the elevator's state has permanently changed:

```
Elevator
 ├── floor = 3
 ├── target = 10
 └── state = Moving()
```

---

# Why pass the context?

The state object itself should usually be **stateless**.

For example:

```python
Idle()
Moving()
Open()
```

are just behavior objects that encode:

```
What events are legal?
What is the next state?
What side effects occur?
```

The actual data lives in the aggregate:

```
             Elevator
         +----------------+
         | current floor  |
         | target floor   |
         | door status    |
         | current state  |
         +----------------+
                  |
                  v
          Current State object
        (Idle / Moving / Open)
```

This preserves a clean ownership model:

* Aggregate owns data and invariants.
* State objects own transition rules.

---

## Alternative: return the next state

Some designs avoid mutating the context directly:

```python
class Idle(State):
    def move(self, target):
        return Moving(target)
```

Then:

```python
class Elevator:
    def request_move(self, target):
        self.state = self.state.move(target)
```

The transition becomes a pure function:

```
(current_state, event) → next_state
```

which is closer to the mathematical definition of a finite state machine:

[
\delta(s, e) = s'
]

This style is common in:

* functional programming,
* event-sourced systems,
* deterministic simulations.

---

## Which is better?

For an object-oriented aggregate (DDD, Java/C++ style):

```
Elevator → delegates to State → State mutates Elevator
```

is the classic State Pattern.

For systems where correctness, replayability, or testing are important:

```
next_state = transition(current_state, event)
```

is often cleaner.

The deeper principle is that a state machine is a **transition relation**. The main design decision is simply **where the transition function lives and who owns the authority to apply it**.
