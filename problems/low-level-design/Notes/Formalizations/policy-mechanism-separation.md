Yes. This has a very deep formalization. What you're seeing is not just "moving the mutex out"; it is a general architectural principle:

> **Separate the coordination policy from the computation or domain logic.**

In software architecture this appears under several formal names:

* **Dependency Inversion Principle (DIP)** — high-level policies depend on abstractions, not concrete mechanisms.
* **Separation of Concerns (SoC)** — isolate independent dimensions of change.
* **Policy–Mechanism Separation** — decide *what guarantee is needed* separately from *how it is implemented*.
* **Aspect-Oriented Programming (AOP)** — cross-cutting concerns (locking, logging, transactions, security) are woven around business logic.
* **Orthogonalization / Modularization** — make independent axes of variation separate modules.

Concurrency control is one instance of a much broader design pattern.

---

# General formal model

A computation can be viewed as:

[
Result = Mechanism(Policy, State, Operation)
]

or more architecturally:

```
           Application Logic
                  |
                  v
          Abstract Capability
                  |
     ----------------------------
     |            |             |
  Mutex       Transaction    Actor Runtime
  Lock        Manager        Scheduler
```

The application specifies a capability:

```
"I need atomicity"
"I need mutual exclusion"
"I need consistency"
```

The infrastructure provides the implementation.

---

# 1. Database transactions

Probably the canonical example.

Business logic:

```python
transfer(A, B, 100)
```

Desired policy:

```
Atomic
Consistent
Isolated
Durable
```

Mechanism chosen by the database:

* Two-phase locking
* MVCC
* Optimistic concurrency control
* Timestamp ordering

The application never says:

```
lock row A
lock row B
unlock
```

The transaction manager controls it.

---

# 2. Memory management

Old approach:

```cpp
void f() {
    Object* x = new Object();
    delete x;
}
```

Business logic controls memory.

Inverted approach:

```
Application
     |
     v
Allocator Interface
     |
---------------------
|          |         |
GC     Arena     Reference Count
```

Examples:

* Garbage collectors
* Region allocators
* Object pools
* Custom allocators

The code expresses:

> "I need an object lifetime."

The runtime decides how memory is reclaimed.

---

# 3. Scheduling

A thread says:

```text
I have work to execute
```

It does not decide:

* which CPU core
* when to run
* how long to run

The operating system scheduler controls:

* round robin
* priority scheduling
* fair scheduling
* work stealing

Formalization:

```
Task
 |
Scheduler abstraction
 |
Scheduling algorithm
```

---

# 4. Networking

Application:

```
send(message)
```

It does not manage:

* packet fragmentation
* retransmission
* congestion control
* routing

The network stack provides:

```
Application
     |
Transport API
     |
TCP / QUIC
     |
IP
     |
Link Layer
```

The policy is "reliable ordered delivery."

The mechanism is TCP.

---

# 5. Security

Business code:

```java
@RequiresRole("admin")
void deleteUser()
```

The code expresses policy:

```
Only admins may perform this operation
```

Security infrastructure decides:

* authentication
* token verification
* permission lookup
* audit logging

---

# 6. Caching

Application:

```
getPrice(stock)
```

The application does not decide:

```
Check L1 cache
Check Redis
Query database
```

The caching layer controls:

```
Request
 |
Cache abstraction
 |
-----------------
|       |       |
L1     L2     Distributed
```

---

# 7. Distributed systems

A service says:

```
replicate this state
```

The system decides:

* leader election
* quorum
* consensus
* log replication

Examples:

* Paxos
* Raft
* Zab

The application specifies consistency requirements.

The runtime implements them.

---

# 8. Parallel programming frameworks

You write:

```python
map(f, data)
```

You do not decide:

* thread creation
* chunk sizes
* task stealing
* CPU assignment

The framework manages execution.

Examples:

* OpenMP
* MapReduce
* Spark

---

# A more mathematical way to see it

A program usually has two types of logic:

## Functional logic

$$
f : State \rightarrow State
$$

Example:

$$
withdraw(account, 100)
$$


It describes the state transition.

---

## Control logic

$$
C(f)
$$

which wraps the operation with guarantees:

$$
C(f) =
\text{lock} \circ f \circ \text{unlock}
$$

or:

$$
C(f) =
\text{transaction} \circ f \circ \text{commit}
$$

or:

$$
C(f)=\text{schedule}(f)
$$

The important idea is that the domain operation (f) is **composed with a policy operator** (C).

---

# The deepest abstraction

The recurring architectural pattern is:

```
          Pure State Transition
                    |
                    v
          Required Properties
      (atomicity, durability,
       security, performance)
                    |
                    v
          Control Infrastructure
                    |
                    v
             Concrete Mechanism
```

This is why databases, operating systems, runtimes, distributed systems, and modern frameworks all look structurally similar.

The formal name that ties them all together is **policy–mechanism separation**, often implemented through **dependency inversion**, **capability abstraction**, or **cross-cutting concern separation**.

For someone studying systems design, this is one of the highest-leverage mental models: whenever you see code mixing *state transformation* with *coordination/resource-management concerns*, ask whether those concerns should be extracted into a separate policy layer.
