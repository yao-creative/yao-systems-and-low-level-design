Below is a **clean reduction of Uber, WhatsApp, Stripe, YouTube** into the same 5 HLD primitives:

> (1) state placement
> (2) state propagation
> (3) coordination model
> (4) time model
> (5) read model

This is useful because it removes all “architecture noise” (microservices, queues, caches) and keeps only *structural invariants*.

---

# 1. UBER (ride-hailing system)

## (1) State placement

Where truth lives:

* Trip state: centralized backend DB (strong consistency per trip)
* Driver location: ephemeral distributed state (stream + cache)
* Supply/demand: derived aggregates

---

## (2) State propagation

How state changes:

* driver location → streaming updates (push)
* ride request → event (dispatch)
* matching → async event pipeline

Flow:
[
request \to match \to dispatch \to trip_state_update
]

---

## (3) Coordination model

* hybrid:

  * dispatcher (trip lifecycle events)
  * scheduler (matching engine decides driver allocation)
  * actor-like isolation per trip

Key:

> matching service = **global scheduler over spatial state**

---

## (4) Time model

* real-time streaming (high frequency location updates)
* event-driven lifecycle (ride start → end)

Mixed:

* continuous (GPS stream)
* discrete (ride events)

---

## (5) Read model

* ETA queries → precomputed heuristics + cache
* maps → stream-aggregated projections
* surge pricing → real-time aggregates

---

## Uber compressed

> distributed state machine + real-time stream + global matching scheduler

---

# 2. WHATSAPP (messaging system)

## (1) State placement

* message state: durable log (server-side)
* delivery state: replicated ephemeral state per device
* user presence: volatile distributed cache

---

## (2) State propagation

* send message → append log (event sourcing core)
* fanout → push notifications (async)
* delivery receipts → bidirectional events

Flow:
[
message \to log \to fanout \to delivery_acks
]

---

## (3) Coordination model

* actor-like per conversation or per user queue
* dispatcher at message routing layer
* no central scheduler (ordering delegated to log)

Key:

> Kafka-like log is the *primary coordinator*

---

## (4) Time model

* mostly asynchronous
* eventual consistency
* ordering per conversation partition

---

## (5) Read model

* inbox = materialized view of message log
* online status = cached ephemeral state

---

## WhatsApp compressed

> event-sourced distributed log + fanout actors + eventual consistency messaging graph

---

# 3. STRIPE (payments system)

## (1) State placement

* source of truth: ledger DB (strong consistency)
* transaction state: strictly ordered per account
* fraud state: derived scoring system

---

## (2) State propagation

* payment request → validation pipeline → ledger write → webhook events
* reconciliation jobs → background correction events

Flow:
[
charge \to authorize \to capture \to settle
]

---

## (3) Coordination model

* controller-driven for critical flows (payments)
* dispatcher-driven for webhook events
* strict transactional boundaries (ACID core)

Key:

> **centralized deterministic state machine per account**

---

## (4) Time model

* synchronous for ledger writes
* async for fraud, notifications, reporting

---

## (5) Read model

* balances → derived from ledger (CQRS pattern)
* dashboards → materialized views
* analytics → batch pipelines

---

## Stripe compressed

> strongly consistent financial state machine + event propagation layer + strict ordering per entity

---

# 4. YOUTUBE (video platform)

## (1) State placement

* video metadata: distributed DB
* video content: object storage (blob system)
* watch history: per-user event log
* ranking state: global derived model

---

## (2) State propagation

* upload → ingestion pipeline → encoding → CDN replication
* view event → streaming analytics pipeline
* engagement → ranking updates

Flow:
[
upload \to transcode \to distribute \to recommend
]

---

## (3) Coordination model

* pipeline-driven (dataflow DAG)
* scheduler (batch + streaming jobs)
* dispatcher (event ingestion)

Key:

> hybrid DAG + streaming scheduler system

---

## (4) Time model

* asynchronous pipelines (encoding, ranking)
* near-real-time (recommendation updates)
* batch + streaming hybrid

---

## (5) Read model

* homepage feed = materialized recommendation view
* search index = inverted index system
* video playback = CDN edge cache

---

## YouTube compressed

> distributed dataflow DAG + streaming event system + batch/real-time hybrid scheduler

---

# 5. Cross-system comparison (this is the real insight)

| System   | State placement          | Control model             | Time model        | Core structure                |
| -------- | ------------------------ | ------------------------- | ----------------- | ----------------------------- |
| Uber     | mixed (central + stream) | global scheduler          | real-time + event | spatial state machine         |
| WhatsApp | log-centric              | dispatcher + log ordering | async             | event-sourced messaging graph |
| Stripe   | strongly centralized     | controller-driven         | mostly sync       | transactional state machine   |
| YouTube  | distributed + derived    | pipeline/scheduler hybrid | batch + stream    | dataflow system               |

---

# 6. The deep unification (what you should actually see)

All four are variations of:

[
S \xrightarrow{\text{control}} E \xrightarrow{\text{transport}} S
]

But differ in:

### (A) where truth lives

* Stripe → centralized
* WhatsApp → log
* Uber → hybrid
* YouTube → distributed + derived

---

### (B) who decides next transition

* Stripe → controller
* WhatsApp → log ordering
* Uber → scheduler (matching engine)
* YouTube → DAG scheduler

---

### (C) how time behaves

* Stripe → transactional time
* WhatsApp → causal/event time
* Uber → real-time + event hybrid
* YouTube → batch + streaming time

---

# 7. One-line compression

* Uber = **real-time scheduler over spatial state**
* WhatsApp = **event-sourced distributed log system**
* Stripe = **centralized deterministic state machine**
* YouTube = **streaming + batch dataflow DAG**

---

If you want next step, I can:

* reduce ALL of these into a **single universal “HLD algebra”**
* or show how each system decomposes into your earlier **control/transport/execution 3-layer model**
* or build a **decision tree: given requirements → pick architecture automatically (very interview-useful)**
