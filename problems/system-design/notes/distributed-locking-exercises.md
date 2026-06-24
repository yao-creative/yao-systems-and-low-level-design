Intent classification: **constraint-satisfaction / system design skill acquisition (distributed coordination primitive: mutual exclusion under partial failure)**

From first principles, distributed locking exists to enforce a **global invariant** (“only one actor may enter critical section”) while the system is split across nodes with **no shared memory, unreliable latency, and partial failure**. Exercises should therefore stress different failure modes: crash, network partition, clock skew, duplicate execution, and lease expiry.

A good training set is structured by **failure model → coordination mechanism → correctness property**.

---

## 1) Core Single-Resource Locking (baseline correctness)

### Exercise 1 — Redis SETNX Lock (single resource)

Design a lock for:

* one shared resource (e.g., “inventory row update”)
* multiple workers

**Constraints:**

* workers may crash mid-critical-section
* network retries may duplicate requests

**Requirements:**

* mutual exclusion
* bounded lock lifetime

**Goal property:** safety (no concurrent entry), partial liveness

**Extension:**

* implement TTL-based auto-release

---

### Exercise 2 — Lease-Based Lock with Heartbeat

Upgrade Exercise 1:

* lock must be extended via heartbeat

**Failure mode introduced:**

* slow worker vs fast TTL expiry → “split brain ownership”

**Goal property:**

* prevent stale holder from continuing execution

---

## 2) Idempotency + Duplicate Execution (semantic correctness layer)

### Exercise 3 — Exactly-once critical section execution

Simulate:

* lock acquisition succeeds
* process crashes after side-effect starts (e.g., payment charge)

**Task:**
Design a system where re-execution does not double-apply side effects.

**Key idea:**

* lock is not enough → need **idempotency key**

**Target invariant:**

* at-most-once effect execution

---

### Exercise 4 — Request Deduplication Lock

Model:

* API gateway receives repeated identical requests

**Goal:**

* ensure only one downstream execution per logical request

This is equivalent to distributed lock over:

* request hash space

---

## 3) ZooKeeper / Consensus-Based Locking (strong correctness)

### Exercise 5 — Ephemeral Sequential Node Lock

Using a coordination system like Apache ZooKeeper:

**Design:**

* clients create sequential nodes
* smallest sequence gets lock
* failure deletes ephemeral node

**Failure model tested:**

* session drop
* leader re-election of lock holder

**Property:**

* strong fairness + ordering guarantee

---

### Exercise 6 — Compare Redis vs ZooKeeper Lock

Build a decision table:

| Property            | Redis Lock | ZooKeeper Lock |
| ------------------- | ---------- | -------------- |
| partition tolerance | medium     | high           |
| fairness            | weak       | strong         |
| latency             | low        | higher         |

**Goal:**

* reason about trade-offs, not implementation

---

## 4) Multi-Resource / Hierarchical Locking

### Exercise 7 — Two-Level Lock (Shard + Row)

Scenario:

* distributed database update requires:

  * shard lock
  * row lock inside shard

**Failure mode:**

* deadlock cycle across nodes

**Task:**

* design global ordering rule or lock hierarchy

**Target invariant:**

* acyclic acquisition graph

---

### Exercise 8 — Distributed Deadlock Detection

Extend Exercise 7:

* locks can be acquired dynamically

**Task:**

* detect cycles in wait-for graph across nodes

---

## 5) Lease + Time + Partial Failure Modeling

### Exercise 9 — Clock Skew Safe Lock

Assume:

* nodes have unsynchronized clocks

**Problem:**

* TTL expiry is inconsistent across nodes

**Task:**

* design lock using monotonic lease tokens

**Key idea:**

* versioned ownership (fencing tokens)

---

### Exercise 10 — Fencing Token Protection

Introduce external system (e.g., database writer)

**Goal:**
Prevent stale lock holder from writing after lease expiry.

**Mechanism:**

* monotonic increasing token enforced by downstream system

---

## 6) Real System Design Integrations

### Exercise 11 — Distributed Job Scheduler

Design:

* multiple workers pulling from job queue

Constraints:

* no double execution
* retry on crash
* load balancing

Lock is used for:

* job claiming OR partition assignment

---

### Exercise 12 — Leader Election via Locking

Build:

* one active coordinator in a cluster

Failure modes:

* split brain
* network partition

Compare:

* lock-based leader election vs consensus-based election in etcd

---

## 7) Advanced Stress Scenarios (edge correctness)

### Exercise 13 — Lock Under Network Partition

Simulate:

* two partitions both believe they hold lock

Task:

* define safety vs availability trade-off explicitly

---

### Exercise 14 — Cascading Lock Expiry Failure

Scenario:

* many locks expire simultaneously

Effect:

* thundering herd re-acquisition

Task:

* add jitter / backoff / token queueing

---

## 8) Meta Exercise (Modeling Skill)

### Exercise 15 — Formal Spec of a Distributed Lock

Write invariant in pseudo-form:

* Safety: ∀t, at most one holder
* Liveness: if holder crashes, lock eventually reassigns
* Freshness: only valid holder can produce side effects

Then map each invariant → mechanism:

* TTL
* fencing token
* consensus log

---

## Compression View (what this exercise set is really training)

You are progressively stress-testing 4 orthogonal axes:

1. **Mutual Exclusion Model**

   * SETNX / consensus / ordering

2. **Failure Semantics**

   * crash, retry, partition, skew

3. **State Freshness**

   * TTL vs lease vs token

4. **Side-effect Safety**

   * idempotency + fencing

---

If you want, next step I can convert this into:

* LeetCode-style problem sheet with inputs/outputs
* or a full system design interview rubric (what to say vs what breaks)
* or a “distributed lock canonical implementation zoo” in Redis / ZooKeeper / etcd style comparison table
