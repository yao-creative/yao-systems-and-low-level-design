If your goal is a rigorous Low-Level Design (LLD) curriculum, concurrency can be organized into a small set of mathematical concepts and implementation patterns.

# 1. Core Mathematical Model

A concurrent program is fundamentally:

[
S \xrightarrow{a} S'
]

where:

* (S) = global state
* (a) = action/event
* multiple actors can perform actions

Formally:

[
(T_1 \parallel T_2 \parallel \cdots \parallel T_n)
]

where (T_i) are independent transition systems.

The challenge is that transitions can interleave.

Example:

```text
T1: read x
T1: write x+1

T2: read x
T2: write x+1
```

Possible execution:

```text
T1 read
T2 read
T1 write
T2 write
```

This creates a race condition.

---

# 2. Fundamental Problems

Everything in concurrency is solving one of:

| Problem          | Meaning                                |
| ---------------- | -------------------------------------- |
| Mutual Exclusion | Only one actor enters critical section |
| Coordination     | Actors synchronize progress            |
| Communication    | Actors exchange information            |
| Scheduling       | Decide who runs next                   |
| Consistency      | Shared state remains valid             |
| Liveness         | Something eventually happens           |
| Fault Tolerance  | Survive failures                       |

These are the primitive goals.

---

# 3. Core Concepts

## Process

Independent execution context.

```text
Memory A
CPU state A
```

OS abstraction.

---

## Thread

Lightweight process.

```text
Process
 ├─ Thread
 ├─ Thread
 └─ Thread
```

Share memory.

---

## Task

Logical unit of work.

```python
async def send_email():
    ...
```

Not necessarily an OS thread.

---

## Coroutine

Suspendable computation.

```python
await database_query()
```

Control explicitly yields.

---

## Actor

Owns state.

Communicates only via messages.

```text
Actor A
  mailbox

Actor B
  mailbox
```

No shared memory.

---

## Event

Immutable occurrence.

```python
OrderPlaced
```

Represents "something happened".

---

## Message

Data transferred between actors/tasks.

```python
SendEmail(user)
```

---

# 4. Synchronization Primitives

These are concurrency's "data structures."

---

## Mutex

Guarantees:

[
\text{at most one holder}
]

```python
with lock:
    update_balance()
```

Protects invariants.

---

## Semaphore

Generalized mutex.

Counter:

[
0 \le c \le N
]

Allows N simultaneous holders.

```python
Semaphore(10)
```

Example:

```text
Only 10 requests at once.
```

---

## Condition Variable

Wait for predicate.

```python
while queue.empty():
    cond.wait()
```

Represents:

```text
Wait until P(state)=True
```

---

## Barrier

All participants meet.

```text
T1 ----\
T2 ----- Barrier
T3 ----/
```

No one proceeds early.

---

## Read Write Lock

Many readers.

One writer.

```text
R R R allowed
W exclusive
```

---

# 5. Communication Patterns

## Shared Memory

```text
Thread A
      |
 shared state
      |
Thread B
```

Requires synchronization.

---

## Message Passing

```text
A --> queue --> B
```

No shared mutable state.

Actor systems use this.

---

## Publish Subscribe

```text
Publisher
   |
   +--> S1
   +--> S2
   +--> S3
```

Decouples producers and consumers.

---

## Request Response

```text
A -> Request
B -> Response
```

RPCs.

HTTP.

gRPC.

---

# 6. Scheduling Patterns

Scheduling decides execution order.

---

## FIFO Queue

```text
A B C D
```

Most common.

---

## Priority Queue

```text
Urgent first
```

OS schedulers.

Task systems.

---

## Round Robin

```text
A
B
C
A
B
C
```

Fair sharing.

---

## Work Stealing

Used by modern runtimes.

```text
Worker1 queue
Worker2 queue
Worker3 queue
```

Idle workers steal work.

Examples:

* ForkJoin
* Rayon
* Tokio

---

# 7. High-Level Concurrency Patterns

These appear repeatedly in real systems.

---

## Producer Consumer

```text
Producer -> Queue -> Consumer
```

Most important pattern.

Examples:

* Kafka
* RabbitMQ
* Celery

---

## Worker Pool

```text
Jobs
 |
Queue
 |
Workers
```

Bound parallelism.

---

## Pipeline

```text
Stage1
  |
Stage2
  |
Stage3
```

Examples:

* video encoding
* ETL
* compilers

---

## Fan Out

```text
      /-> W1
Job --|
      \-> W2
```

Parallelize work.

---

## Fan In

```text
W1 \
W2  -> merge
W3 /
```

Collect results.

---

## Map Reduce

```text
Map
 ↓
Shuffle
 ↓
Reduce
```

Distributed fan-out/fan-in.

---

## Event Loop

```text
while True:
    event = queue.pop()
    dispatch(event)
```

Examples:

* asyncio
* Node.js

---

## Reactor Pattern

```text
Socket Ready
      |
   Dispatcher
      |
   Handler
```

Nginx.

asyncio.

libuv.

---

# 8. Correctness Concepts

## Race Condition

Outcome depends on timing.

```python
x += 1
```

from multiple threads.

---

## Deadlock

Cycle of waiting.

```text
A waits B
B waits A
```

Graph-theoretically:

[
\text{cycle in wait-for graph}
]

---

## Livelock

Everyone moving.

Nobody progressing.

---

## Starvation

One task never gets service.

---

## Fairness

Eventually scheduled.

---

## Linearizability

Operations appear atomic.

Most important correctness property in distributed systems and concurrent data structures.

---

## Happens-Before

Formal ordering relation.

Introduced by Leslie Lamport.

[
a \rightarrow b
]

means:

```text
a definitely occurred before b
```

Foundation of modern concurrency reasoning.

---

# 9. The Five LLD Concurrency Primitives

Nearly every system can be decomposed into:

```text
1. State
2. Transition
3. Scheduler
4. Communication Channel
5. Synchronization Policy
```

Example: Elevator

```text
State:
    floor
    direction
    requests

Transition:
    move_up()
    move_down()

Scheduler:
    choose next request

Channel:
    hall button events

Synchronization:
    lock state updates
```

Example: Uber

```text
State:
    drivers
    riders

Transition:
    assign ride

Scheduler:
    matching algorithm

Channel:
    event bus

Synchronization:
    transactional updates
```

From a formal perspective, most concurrency frameworks (threads, actors, asyncio, distributed systems, operating systems) are different implementations of:

[
(State,\ Transition,\ Scheduler,\ Channel,\ Synchronization)
]

Understanding those five primitives gives a unifying model for reasoning about everything from `asyncio` to distributed databases.
