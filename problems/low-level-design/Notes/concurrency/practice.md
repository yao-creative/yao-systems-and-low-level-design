The LLD versions are much more interesting because you're no longer implementing a synchronization primitive—you are designing a system whose correctness depends on concurrency.

A useful distinction:

```text
Concurrency LeetCode
=
Implement a protocol

Concurrency LLD
=
Design a system that contains many protocols
```

---

# Level 1: Single Shared Resource

## Rate Limiter

Design:

```text
allow(user)?
```

Requirements:

* concurrent requests
* thread-safe counters
* token bucket or leaky bucket

Concepts:

```text
State:
    tokens

Transition:
    consume token

Synchronization:
    mutex / atomic

Scheduler:
    refill timer
```

Tests:

* races
* atomicity
* lock granularity

---

## Parking Lot

Usually taught as OOP.

Interesting version:

```text
1000 cars arriving simultaneously
```

Questions:

* who owns occupancy state?
* lock per floor?
* lock per spot?
* transactional allocation?

Tests:

* contention
* state ownership

---

## Elevator System

Your earlier question is exactly a concurrency LLD.

```text
Hall requests
Cab requests
Scheduler
Elevator controller
Clock
```

Concurrency questions:

* who mutates elevator state?
* scheduler thread?
* controller thread?
* event queue?

Formal decomposition:

```text
State
Scheduler
Dispatcher
Transition System
```

---

# Level 2: Producer Consumer Systems

These appear everywhere.

---

## Notification Service

```text
User Action
    |
Queue
    |
Workers
    |
Email/SMS
```

Tests:

* retries
* queues
* backpressure

Concepts:

```text
Producer
Consumer
Worker Pool
```

---

## Log Processing System

```text
Applications
      |
 Log Queue
      |
 Aggregators
      |
 Storage
```

Tests:

* batching
* throughput
* worker pools

---

## Video Processing Pipeline

```text
Upload
 ↓
Transcode
 ↓
Thumbnail
 ↓
Store
```

Tests:

* fan out
* fan in
* orchestration

---

# Level 3: Scheduler-Centric Systems

These are the closest to operating systems.

---

## Task Scheduler

Design:

```text
schedule(job, time)
```

Concepts:

```text
Priority Queue
Dispatch
Workers
Clock
```

Questions:

* delayed execution
* fairness
* retries

Very common interview.

---

## Job Queue (Celery Clone)

```text
enqueue()
dequeue()
ack()
retry()
```

Concepts:

```text
Producer
Broker
Worker
Acknowledgement
```

Concurrency becomes central.

---

## CPU Scheduler

Design:

```text
submit(process)
```

Implement:

```text
Round Robin
Priority
Shortest Job First
```

This is basically concurrency theory.

---

# Level 4: Multi-Actor Coordination

---

## Ride Matching (Uber)

Actors:

```text
Driver
Rider
Matcher
```

Concurrency issues:

```text
2 drivers claim rider
2 riders claim driver
```

Need:

```text
reservation protocol
atomic assignment
```

---

## Food Delivery

Actors:

```text
Restaurant
Driver
Customer
```

State machine:

```text
Created
Accepted
Cooking
PickedUp
Delivered
```

Concurrency issue:

```text
multiple updates simultaneously
```

This becomes distributed concurrency.

---

## Stock Exchange

Actors:

```text
Buy Orders
Sell Orders
Matching Engine
```

Tests:

* ordering
* fairness
* latency

One of the hardest concurrency systems.

---

# Level 5: Distributed Concurrency

---

## Distributed Lock Service

Design:

```text
acquire()
release()
```

Concepts:

* leases
* consensus
* failures

Think Redis lock or ZooKeeper.

---

## Kafka

Actors:

```text
Producer
Broker
Consumer
```

Concepts:

* partition ownership
* ordering
* offsets

---

## Distributed Task Queue

Actors:

```text
Client
Broker
Workers
```

Need:

```text
exactly once?
at least once?
```

Classic distributed systems problem.

---

# Formal Pattern Catalog

Almost every concurrency-heavy LLD reduces to one of these:

| Pattern           | Examples                   |
| ----------------- | -------------------------- |
| Shared Resource   | Parking Lot, Rate Limiter  |
| Producer Consumer | Notification Service       |
| Worker Pool       | Web Crawler, Email Service |
| Scheduler         | Elevator, Task Scheduler   |
| Event Loop        | Chat Server                |
| State Machine     | Order System, Elevator     |
| Matching Engine   | Uber, Exchange             |
| Pipeline          | Video Processing           |
| Pub/Sub           | Kafka                      |
| Actor System      | Chat, Multiplayer Game     |
| Distributed Lock  | Coordination Service       |

---

From a formal perspective, the most valuable concurrency LLD exercises are:

1. Elevator System
2. Rate Limiter
3. Task Scheduler
4. Job Queue (Celery)
5. Notification Service
6. Uber Matching
7. Kafka-like Event Bus

Together they force you to reason about:

```text
State ownership
Synchronization
Scheduling
Message passing
Backpressure
Fault tolerance
```

which are the five major dimensions of real-world concurrency design. The elevator is particularly valuable because it sits exactly at the intersection of finite-state machines, scheduling, event-driven architecture, and concurrency control. It is essentially a miniature operating system scheduler disguised as an LLD problem.
