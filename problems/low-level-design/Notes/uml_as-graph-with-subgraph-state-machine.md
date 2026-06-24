A useful way to formalize **substates** is through **hierarchical state machines (HSMs)** or **statecharts**.

Instead of treating a state as atomic, a state can itself contain another state machine.

---

# 1. Classical FSM

A finite state machine is:

$$
M = (S, \Sigma, \delta, s_0, F)
$$

where:

* (S) = states
* (\Sigma) = events/alphabet
* (\delta : S \times \Sigma \rightarrow S)
* (s_0) = initial state
* (F) = accepting states

Example:

```
Idle → Running → Stopped
```

States are atomic.

---

# 2. Hierarchical FSM (Statecharts)

Now let a state contain another FSM.

Example:

```
Elevator
├── Idle
└── Moving
    ├── MovingUp
    └── MovingDown
```

Formally:

$$
s = (s_{parent}, s_{child})
$$

Global state is actually a tuple.

If elevator is moving up:

$$
(Moving, MovingUp)
$$

If moving down:

$$
(Moving, MovingDown)
$$

The child state only exists when parent is active.

---

# 3. Active Configuration

In a hierarchy, the actual runtime state is the path from root to leaf.

Instead of:

$$
s \in S
$$

we have:

$$
c = (s_0,s_1,\ldots,s_n)
$$

called the **active configuration**.

Example:

```
System
 └── Connected
      └── Authenticated
           └── Streaming
```

Runtime:

$$
(System, Connected, Authenticated, Streaming)
$$

Every ancestor is implicitly active.

---

# 4. Parent-Child Semantics

Suppose:

```
Moving
├── Up
└── Down
```

Rule:

$$
active(Up) \implies active(Moving)
$$

$$
active(Down) \implies active(Moving)
$$

Child activation implies parent activation.

This creates a partial order:

$$
Up \le Moving
$$

$$
Down \le Moving
$$

---

# 5. Entry and Exit Actions

Each state can define:

$$
entry(s)
$$

$$
exit(s)
$$

Example:

```
Moving
 ├── Up
 └── Down
```

Transition:

```
Up → Down
```

execution:

```
exit(Up)
entry(Down)
```

Parent stays active.

Transition:

```
Moving → Idle
```

execution:

```
exit(Up)
exit(Moving)
entry(Idle)
```

Entire subtree exits.

---

# 6. State Refinement

Substates are often interpreted as a refinement relation.

$$
Up \subseteq Moving
$$

meaning:

"Up satisfies all properties of Moving plus additional constraints."

Example:

```
Moving
```

Property:

$$
velocity \neq 0
$$

Refinement:

```
MovingUp
```

Property:

$$
velocity > 0
$$

Refinement:

```
MovingDown
```

Property:

$$
velocity < 0
$$

Substates add information.

---

# 7. Product-State Interpretation

Another formalization:

Instead of hierarchy:

```
Moving
 ├── Up
 └── Down
```

write:

$$
Direction \in {Up,Down}
$$

$$
Motion \in {Idle,Moving}
$$

Global state:

$$
(Motion,Direction)
$$

This is the Cartesian product:

$$
S = Motion \times Direction
$$

Statecharts compress this representation.

---

# 8. Orthogonal Substates (Parallel Regions)

Statecharts allow multiple active substates simultaneously.

Example:

```
Connection
├── NetworkState
│   ├── Connected
│   └── Disconnected
└── AuthState
    ├── LoggedIn
    └── LoggedOut
```

Current state:

$$
(Connected, LoggedIn)
$$

or

$$
(Connected, LoggedOut)
$$

Formally:

$$
S = S_{network} \times S_{auth}
$$

This is equivalent to concurrent FSMs.

---

# 9. LTS Interpretation

In a labelled transition system:

$$
L = (Q,A,\rightarrow)
$$

a substate induces a state hierarchy:

$$
q_{leaf}
\subseteq
q_{parent}
$$

Transitions may be inherited:

If

$$
Moving \xrightarrow{Stop} Idle
$$

then

$$
MovingUp \xrightarrow{Stop} Idle
$$

and

$$
MovingDown \xrightarrow{Stop} Idle
$$

unless overridden.

This is similar to inheritance in OOP.

---

# 10. Most General Formal View

A state hierarchy is a rooted tree:

$$
H=(S,E)
$$

where:

* nodes = states
* edges = "is-substate-of"

The runtime state is:

$$
c \subseteq S
$$

such that:

1. exactly one leaf active per region
2. all ancestors of active leaves are active
3. active states form a connected root-to-leaf path

This is the mathematical object underlying UML Statecharts.

---

A useful mental model:

| Concept            | Formal Object                           |
| ------------------ | --------------------------------------- |
| State              | Equivalence class of behavior           |
| Substate           | Refinement of behavior                  |
| State hierarchy    | Rooted tree                             |
| Active state       | Root-to-leaf path                       |
| Orthogonal regions | Cartesian product of FSMs               |
| Entry/exit         | Transition actions                      |
| Parent state       | Abstraction                             |
| Child state        | More constrained specification          |
| Statechart         | Hierarchical labelled transition system |

From a semantics perspective, a **substate is not merely a smaller state**; it is a **refinement relation**:

$$
\text{Substate} = \text{Parent State} + \text{Additional Invariants}
$$

which is why hierarchical state machines are often viewed as a compressed representation of a much larger flat FSM.
