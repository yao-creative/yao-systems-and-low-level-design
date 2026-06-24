You're touching three separate but related questions:

1. **How do we represent time?**
2. **How do we optimize speed/latency?**
3. **Why do DAG systems and Actor systems feel fundamentally different?**

---

# 1. Do we endogenize time?

Very often, yes.

The most common trick in CS is:

> Replace time by state.

Instead of:

$$
x(t)
$$

we store:

$$
(state, t)
$$

or

$$
(state, logical_clock)
$$

inside the state space.

Examples:

### FSM

Time is implicit:

$$
s_0 \to s_1 \to s_2
$$

Path length acts as time.

---

### Simulators

State:

$$
(state, tick)
$$

```python
tick += 1
```

---

### Networking

State:

$$
(state, sequence_number)
$$

instead of physical time.

---

### Distributed systems

Use logical clocks:

$$
(state, LamportClock)
$$

instead of wall-clock time.

---

# Major representations of time

## 1. Step-indexed

$$
t \in \mathbb N
$$

Used by:

* FSMs
* algorithms
* compilers
* interpreters

---

## 2. Event time

$$
e_1,e_2,e_3
$$

ordered events.

Used by:

* Kafka
* actor systems
* databases

---

## 3. Logical clocks

$$
L(e)
$$

Used by:

* distributed systems
* causal ordering

Lamport clocks.

---

## 4. Partial-order time

Most important for concurrency.

Instead of:

$$
e_1 < e_2 < e_3
$$

you have

$$
e_1 \prec e_3
$$

and

$$
e_2 \prec e_3
$$

but

$$
e_1
$$

and

$$
e_2
$$

are incomparable.

Graph:

```text
e1 \
     -> e3
e2 /
```

This is often the most accurate notion of time in parallel systems.

---

## 5. Continuous time

$$
t \in \mathbb R
$$

Used by:

* control systems
* robotics
* physics

---

# High-frequency trading: min-cut/max-flow?

Generally no.

Min-cut/max-flow is mostly a **throughput optimization** model.

HFT is dominated by:

$$
\text{latency}
$$

not flow.

Typical models are:

### Critical path

DAG longest path:

$$
T = \max_{path} \sum latency_i
$$

---

### Queueing theory

$$
M/M/1
$$

$$
M/G/k
$$

etc.

---

### Network topology optimization

Shortest path.

---

### Cache hierarchy optimization

Memory latency dominates.

---

### Lock-free algorithms

Avoid waiting.

---

### FPGA pipelines

Fixed deterministic latency.

---

HFT engineers care much more about:

* queue depth
* cache misses
* context switches
* lock contention
* network hops

than max-flow.

---

# DAGs vs Actors

Your intuition is largely correct.

---

## DAG

Example:

```text
A -> B -> D
 \-> C ->/
```

We can compute:

$$
A < B < D
$$

$$
A < C < D
$$

using a topological sort.

Kahn's algorithm.

The dependency structure is known beforehand.

---

Because of this:

Pipeline parallelism becomes straightforward.

```text
A
 \
  B
   \
    D

A
 \
  C
   /
```

As soon as dependencies are satisfied:

run.

No semantic ambiguity.

---

# Do DAGs need queues?

Physically yes.

Logically no.

The distinction matters.

---

Logically:

Topological order already exists.

No mailbox semantics needed.

---

Implementation:

Workers still need:

* ready queues
* work stealing queues

because CPUs are finite.

Example:

Ray, Dask, Airflow, Spark all use queues internally.

But the queue is an implementation artifact.

Not part of the model.

---

# Actor systems

Actor model:

```text
A --> mailbox(B)
```

Messages arrive asynchronously.

Order isn't globally predetermined.

---

Now we have:

```text
A ---> C
B ---> C
```

Question:

Which arrives first?

Cannot know statically.

Need runtime scheduling.

Need mailbox.

Need buffering.

Need waiting.

---

So actor systems naturally require:

$$
\text{state} + \text{mailbox}
$$

where mailbox is essentially a queue.

---

# Deep distinction

DAG systems model:

$$
\text{dependency}
$$

Actor systems model:

$$
\text{communication}
$$

These are fundamentally different graphs.

---

DAG:

$$
u \to v
$$

means

> v cannot start until u finishes.

---

Actor:

$$
u \to v
$$

means

> u may send messages to v.

No dependency implied.

---

# A useful hierarchy

Many modern systems fit into:

### FSM

State transitions.

---

### DAG

Dependency graph.

Topological ordering.

Kahn.

Critical path.

Pipeline parallelism.

---

### Actor graph

Communication graph.

Mailboxes.

Queues.

Dynamic scheduling.

---

### Event system

Actor graph + event queues.

Kafka, Celery, distributed systems.

---

### Queueing network

Actor graph + service rates + arrival rates.

Performance analysis.

This hierarchy explains why compilers, build systems, and ML pipelines are DAG-centric, while WhatsApp, Erlang, Akka, and distributed services are actor/message-centric. DAGs expose dependency structure statically; actor systems discover execution order dynamically through message arrival and scheduling.
