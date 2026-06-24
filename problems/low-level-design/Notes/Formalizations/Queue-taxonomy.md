A clean way to organize this is to stop thinking in “queues as data structures” and instead classify them as **boundary buffers in a distributed control system**.

So the real axis is:

> **what causal interface does this queue sit on? (input, execution, output, coordination, or memory replay)**

---

# 1. Major queue families (system-level taxonomy)

## A. **Submission / Intake queues (input boundary)**

**Goal:** absorb demand before execution

* Job queues (batch systems, Celery, Sidekiq)
* IO submission queues (e.g., `io_uring` submission queue)
* GPU command buffers
* Task queues in thread pools

**Causal role:**
Producer → “here is work”

---

## B. **Execution / Scheduling queues (dispatch boundary)**

**Goal:** decide *who runs next*

* Run queues (Linux scheduler per CPU core)
* Ready queues (threads that can execute now)
* Priority queues (MLFQ, real-time schedulers)
* Actor mailboxes *partially* behave here (message becomes executable task)

**Causal role:**
System → “this is eligible to run now”

This is where **dispatch logic lives**.

---

## C. **Event / Completion queues (output boundary)**

**Goal:** propagate results of execution

* IO completion queues (e.g., `io_uring` CQ)
* Interrupt event queues
* Callback/event loop queues
* GUI event queues

**Causal role:**
Execution → “something happened”

---

## D. **Message queues (coordination layer)**

This is a *semantic umbrella*, not a stage.

* Kafka topics
* RabbitMQ queues
* Pub/sub buffers
* Actor mailboxes (often here too)

**Key distinction:**
They are not tied to execution timing, but to **information propagation topology**.

So they can behave like:

* submission (commands)
* event (facts)
* or both

---

## E. **Buffering / smoothing queues (rate decoupling layer)**

**Goal:** hide variability

* TCP send/receive buffers
* streaming buffers (audio/video)
* leaky bucket / token bucket queues
* UI render queues

**Causal role:**
“production rate ≠ consumption rate problem”

---

## F. **Replay / state reconstruction queues (temporal memory layer)**

**Goal:** reconstruct history or state

* event sourcing logs
* Kafka log retention streams
* WAL (write-ahead logs)
* audit logs

**Causal role:**
“what happened in order”

This is not runtime coordination — it’s **state derivation substrate**.

---

# 2. Reactors: API or architecture?

A reactor is **not an API**, but it *exposes APIs*.

## Formal definition (causal decomposition)

A reactor is:

> a control loop that demultiplexes event sources into handlers using a single-threaded or event-driven scheduler.

So structurally:

* **Input:** event sources (fds, timers, signals)
* **Core mechanism:** event demultiplexer
* **Output:** callbacks / handlers

---

## Key components

### 1. Event demultiplexer

* `select`, `poll`, `epoll`, `kqueue`
* maps many IO sources → ready set

### 2. Event queue (internal)

* holds “ready events”

### 3. Dispatcher

* calls the correct handler

So reactor =

```
event sources
   ↓
demultiplexer
   ↓
event queue
   ↓
dispatcher
   ↓
handlers
```

---

## Important insight

Reactors are:

> **control-flow inversion engines**

They invert:

* “call this when ready” → instead of → “I will check when ready”

---

# 3. Dispatcher (UI thread, event loop) — formal meaning

This is where most confusion happens.

## First-principles definition

A **dispatcher** is:

> a function that maps events → handlers under scheduling constraints

So:

```
dispatch(event) → handler(event)
```

But in real systems it is embedded in a loop.

---

## UI thread = serialized event dispatcher

A UI thread is:

> a single-threaded event-processing context that guarantees serialized access to mutable UI state.

### Why it exists (causal necessity)

UI state is:

* shared
* mutable
* latency-sensitive
* not concurrency-safe

So instead of locking:

> enforce a single ordered execution lane

---

## UI event loop structure

```
input devices / OS events
        ↓
   event queue
        ↓
   UI dispatcher (main thread)
        ↓
   widget handlers / rendering updates
```

---

## What dispatcher *actually does*

It performs 3 transformations:

### 1. **Dequeue**

pull next event in order

### 2. **Resolve**

find handler (routing table / vtable / callback map)

### 3. **Execute**

run handler on shared state

---

## Why it is not “just an API”

An API is:

* interface boundary
* request/response contract

A dispatcher is:

* **runtime control mechanism**
* enforces ordering, safety, and exclusivity

So:

> API = what you can call
> Dispatcher = how execution is scheduled

---

# 4. Putting it together (causal stack model)

You can unify everything like this:

```
          (Input layer)
     Submission queues
            ↓
      Scheduler / Reactor core
            ↓
     Dispatch / Execution layer
            ↓
      Event / Completion queues
            ↓
      Observers / state machines
```

And UI thread is simply:

> a *specialized dispatcher with strict serialization constraints*

---

# 5. Key insight (the real unifying principle)

All of these structures differ only by one variable:

> **where in the state-transition function they sit**

If you model a system as:

```
State(t+1) = F(State(t), Input, Events)
```

Then:

* submission queues → feed **Input**
* dispatchers → compute **F**
* event queues → expose **Events**
* reactors → implement **F as inverted control flow**

---

If you want next level clarity, I can map this into:

* OS kernel scheduling model (runqueue vs waitqueue vs completion queue)
* asyncio / Node event loop internals
* actor model mailboxes vs event loops
* or a formal state-machine + category-theory interpretation of dispatcher as morphism composition

Those make the distinctions fully “mechanically inevitable” rather than descriptive.
