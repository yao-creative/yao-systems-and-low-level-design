The $$Hello Interview multi-step processes article$$(https://www.hellointerview.com/learn/system-design/patterns/multi-step-processes?utm_source=chatgpt.com) frames multi-step processes somewhat differently from the generic workflow/FSM perspective.

Their viewpoint is specifically:

> **"How do you reliably execute business operations that span multiple services, machines, retries, crashes, and potentially hours or days?"** ($$Hello Interview$$$$1$$)

This is a narrower but extremely important subclass of multi-step processes:

```text
Generic process theory
    ↓
Distributed business workflow
    ↓
Long-running durable workflow
```

---

# Their Formalization

Instead of:

```text
State + Event → Next State
```

they think in terms of:

```text
Step1
↓
Step2
↓
Step3
↓
...
```

where every edge has failure semantics:

```text
Step_i
    ↓
success → Step_i+1
failure → retry / compensate / wait
timeout → retry / human intervention
crash → recover and continue
```

Thus the actual transition function becomes:

$$
T(s_i,a_i) \rightarrow
{
success,
retry,
compensate,
pause,
fail
}
$$

This is much closer to production reality.

---

# The Core Problem

Suppose:

```text
1 Charge card
2 Reserve inventory
3 Create shipping label
4 Send email
```

Simple flowchart:

```text
Charge
 ↓
Reserve
 ↓
Ship
 ↓
Email
```

Real flowchart:

```text
Charge
 │
 ├─ timeout → retry
 ├─ duplicate request → deduplicate
 ├─ crash after success → recover
 └─ failure → refund

Reserve
 ├─ inventory unavailable
 ├─ warehouse offline
 └─ retry later

Ship
 ├─ carrier unavailable
 └─ retry tomorrow
```

The business logic becomes overwhelmed by infrastructure logic. ($$Hello Interview$$$$1$$)

---

# Hello Interview's Taxonomy

## 1. Local Transactions

Single database transaction.

```text
BEGIN
A
B
C
COMMIT
```

Properties:

* ACID
* simple
* not distributed

---

## 2. Saga Pattern

Each step has a compensating action.

```text
Charge Card
Reserve Inventory
Create Shipment
```

Compensations:

```text
Refund Card
Release Inventory
Cancel Shipment
```

Formally:

$$
(a_i,c_i)
$$

where:

* (a_i) = forward action
* (c_i) = compensation action

---

## 3. Choreography

No central coordinator.

Services react to events.

```text
OrderCreated
    ↓
PaymentService
    ↓
PaymentCompleted
    ↓
InventoryService
    ↓
InventoryReserved
```

This is essentially:

```text
FSM + Event Bus
```

Advantages:

* decoupled
* scalable

Disadvantages:

* difficult debugging
* emergent workflows

---

## 4. Orchestration

A workflow engine controls execution.

```text
Workflow Engine
     │
     ├── charge()
     ├── reserve()
     ├── ship()
     └── email()
```

Examples include workflow systems such as Temporal, Cadence, and AWS Step Functions. ($$Hello Interview$$$$1$$)

---

## 5. Durable Execution

This is the article's most important concept.

Instead of:

```text
process crashes
→ start from beginning
```

you get:

```text
charge ✓
reserve ✓
server crashes
restart
ship ← resume here
```

Formally:

$$
WorkflowState =
(CurrentStep, Variables, History)
$$

persisted after every transition.

This resembles:

```text
Event Sourcing
+
Checkpointing
+
FSM
```

combined into one abstraction.

---

# Their Compression

The article effectively compresses all multi-step process problems into:

```text
Long-running stateful computation
+
Failures
+
Recovery
```

or more formally:

$$
Process =
(State, Progress, Recovery)
$$

---

# Interview Heuristic

According to the article, if you hear:

* payment system
* order fulfillment
* onboarding flow
* AI agent pipeline
* email campaign
* human approval process
* video transcoding
* ride dispatch
* refund flow

you should immediately think:

```text
This is a multi-step process problem.
```

and ask:

1. Where is workflow state stored?
2. How are retries handled?
3. Are steps idempotent?
4. How are compensations performed?
5. What happens if the coordinator crashes?
6. How do we resume progress?
7. How do we version workflows? ($$Hello Interview$$$$1$$)

---

# Exercises in the Hello Interview Style

## Exercise 1

Design:

```text
Stripe payment flow
```

Requirements:

* exactly-once charging
* retries
* refunds
* crash recovery

---

## Exercise 2

Design:

```text
Video upload pipeline
```

```text
Upload
↓
Virus Scan
↓
Transcode
↓
Thumbnail
↓
Publish
```

Questions:

* Which steps are idempotent?
* Which can run in parallel?
* Where are checkpoints stored?

---

## Exercise 3

Design:

```text
LLM agent execution engine
```

```text
Retrieve Docs
↓
Search Web
↓
Generate Plan
↓
Call Tools
↓
Summarize
```

What happens if the model crashes after tool call #7?

---

## Exercise 4

Build a mini workflow engine locally:

```text
workflow.json

$$
  "charge",
  "reserve_inventory",
  "send_email"
$$
```

Implement:

* retries
* checkpoints
* resume after crash
* compensations

This exercise teaches about 70% of the practical material behind workflow systems such as Temporal. ($$Hello Interview$$$$2$$)

---

The generic CS formalization of multi-step processes is:

```text
state transition systems
```

The Hello Interview formalization is:

```text
durable state transition systems operating in unreliable environments
```

That distinction captures why this topic appears so frequently in senior and staff-level system design interviews.

$$1$$: https://www.hellointerview.com/learn/system-design/patterns/multi-step-processes?utm_source=chatgpt.com "Multi-step Processes Pattern for System Design Interviews | Hello Interview System Design in a Hurry"
$$2$$: https://www.hellointerview.com/blog/lessons-from-building-hello-interview?utm_source=chatgpt.com "14 Lessons from Building Hello Interview | Hello Interview"
