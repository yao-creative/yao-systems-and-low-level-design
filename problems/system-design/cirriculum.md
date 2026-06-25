A useful way to structure system design growth is by the **scope of optimization**:

| Level  | Primary Optimization Unit           | Typical Question                                       |
| ------ | ----------------------------------- | ------------------------------------------------------ |
| Junior | Function/Class                      | "How do I implement this?"                             |
| Mid    | Service                             | "How do I build this service?"                         |
| Senior | Distributed System                  | "How do I make this scale and remain reliable?"        |
| Staff  | Organization-wide System of Systems | "How do multiple teams evolve this safely over years?" |

Senior and Staff are not separated by intelligence or technical depth alone. The main distinction is **system boundary size**, **time horizon**, and **organizational coupling**.

---

# Senior Engineer System Design Curriculum

Senior engineers optimize:

* Reliability
* Scalability
* Availability
* Latency
* Cost
* Operability

The system boundary is usually:

```text
single service
→ several services
→ one product area
```

---

# Phase 0: Mathematical Foundations

## Complexity

* Time complexity
* Space complexity
* Amortized complexity
* Tail latency analysis
* Throughput modeling

Exercises:

* Derive QPS limits from CPU budgets.
* Estimate memory usage for caches.
* Estimate storage growth over 5 years.

---

## Probability

Topics:

* Expected value
* Variance
* Heavy-tailed distributions
* Poisson arrivals
* Queueing intuition

Exercises:

* Model traffic spikes.
* Compute cache hit rates.
* Estimate retries during failures.

---

## Queueing Theory

Core:

```text
arrival rate λ
service rate μ
utilization ρ = λ/μ
```

Learn:

* Little's Law
* M/M/1
* Backpressure
* Saturation curves

Exercises:

* Model API servers.
* Model Kafka consumers.
* Determine required worker counts.

---

## CAP and Distributed Tradeoffs

Learn:

* CAP
* PACELC
* FLP
* Byzantine vs crash failures

---

# Phase 1: Core Building Blocks

These are your "system primitives".

## Storage

### B-tree

### LSM Tree

### SSTables

### WAL

### Compaction

### MVCC

### Secondary indexes

Exercises:

* Build mini LSM tree.
* Build WAL replay.
* Implement compaction simulator.

---

## Networking

Learn:

* TCP
* UDP
* QUIC
* HTTP1
* HTTP2
* HTTP3
* gRPC
* TLS
* Connection pooling
* Keep alive

Exercises:

* Build chat server over TCP.
* Measure head-of-line blocking.
* Compare HTTP/1 and HTTP/2 multiplexing.

---

## Caching

Topics:

* Cache aside
* Read through
* Write through
* Write behind
* Refresh ahead
* Request coalescing
* Negative caching
* CDN caching

Evictions:

* LRU
* LFU
* ARC
* TinyLFU

Exercises:

* Build LRU cache.
* Simulate hit ratios.
* Implement request deduplication.

---

## Messaging

Topics:

* Queue
* Pub/sub
* Log
* Stream

Systems:

* Kafka
* RabbitMQ
* SQS

Learn:

* Ordering
* Delivery semantics
* Replay
* Offsets
* Consumer groups

Exercises:

* Build Kafka-lite.
* Build retry queues.
* Build delayed queues.

---

## Concurrency

Topics:

* Mutexes
* Semaphores
* Condition variables
* Futures
* Coroutines
* Actor model

Exercises:

* Implement thread pool.
* Build bounded queue.
* Build work stealing scheduler.

---

# Phase 2: Distributed Systems

---

## Replication

Learn:

* Leader follower
* Multi leader
* Leaderless

Topics:

* Read repair
* Hinted handoff
* Anti entropy

Exercises:

* Simulate replica lag.
* Simulate failover.

---

## Consensus

Topics:

* Paxos intuition
* Raft deeply
* Membership changes
* Split brain

Exercises:

* Build toy raft.
* Simulate election storms.

---

## Partitioning

Topics:

* Range sharding
* Hash sharding
* Consistent hashing
* Rebalancing

Exercises:

* Build consistent hashing ring.
* Simulate hot partitions.

---

## Transactions

Topics:

* 2PC
* 3PC
* Saga
* Outbox pattern
* Idempotency

Exercises:

* Implement distributed transfer.
* Handle retries safely.

---

## Coordination

Topics:

* Distributed locks
* Leases
* Fencing tokens
* Epoch numbers

Exercises:

* Build lock service.
* Simulate stale lock holders.

---

## Event Systems

Topics:

* Event sourcing
* CDC
* CQRS
* Materialized views

Exercises:

* Build event store.
* Build replay engine.

---

# Phase 3: Production Reliability

This separates mid-level from senior.

---

## Availability

Learn:

* SLO
* SLA
* SLI
* Error budgets

---

## Observability

Topics:

* Metrics
* Logs
* Traces
* Profiling

Systems:

* Prometheus
* Grafana
* OpenTelemetry

Exercises:

* Instrument service.
* Trace request path.

---

## Failure Engineering

Topics:

* Retry storms
* Cascading failures
* Circuit breakers
* Bulkheads
* Load shedding

Exercises:

* Inject failures.
* Chaos experiments.

---

## Deployment

Topics:

* Rolling deploy
* Blue green
* Canary

Exercises:

* Design zero downtime deploy.

---

## Disaster Recovery

Topics:

* RPO
* RTO
* Backup
* Multi-region failover

---

# Phase 4: Canonical Design Problems

Design:

* URL shortener
* Rate limiter
* Notification system
* Search autocomplete
* News feed
* Chat
* Metrics system
* Distributed cron
* Job scheduler
* API gateway
* Recommendation feed
* Payment system
* Ride sharing dispatch
* Real-time analytics
* CDN
* Object storage
* Distributed cache
* Kafka
* Google Docs
* Dropbox
* YouTube

---

# Staff Engineer Curriculum

Staff engineers optimize for:

```text
local optimum
→ global optimum
```

They operate on:

```text
multiple teams
multiple years
multiple systems
```

---

# Phase 1: Architecture Economics

Topics:

* Cost modeling
* Capacity planning
* Resource efficiency
* Marginal cost per request

Questions:

```text
Should we spend $2M to save 10ms?
Should we build or buy?
Should we split the monolith?
```

Exercises:

* Estimate cloud costs.
* Model infra budgets.

---

# Phase 2: Organizational Architecture

Learn:

* Conway's Law
* Team Topologies
* Platform engineering
* Internal developer platforms

Questions:

```text
How many teams own this service?
Who owns schemas?
Who owns contracts?
```

---

# Phase 3: Evolutionary Architecture

Topics:

* Strangler migration
* Incremental migration
* Versioning
* Compatibility guarantees

Exercises:

* Migrate monolith to services.
* Migrate SQL to sharded storage.

---

# Phase 4: Multi-Region Systems

Topics:

* Active-active
* Active-passive
* Geo replication
* Conflict resolution

Exercises:

* Design global messaging platform.
* Design worldwide database.

---

# Phase 5: Socio-Technical Systems

Topics:

* Incident response
* Postmortems
* Operational readiness reviews
* Runbooks

---

# Phase 6: Platform Design

Topics:

* Service mesh
* Internal APIs
* Deployment platforms
* Build systems

Design:

* Internal PaaS
* CI platform
* Feature flag platform
* Identity platform

---

# Phase 7: Data Architecture

Topics:

* Data contracts
* Data lineage
* Lakehouse architecture
* Streaming pipelines

Technologies:

* Kafka
* Flink
* Spark
* Iceberg
* Delta Lake

---

# Phase 8: Strategy

Questions:

```text
What becomes a platform?
What stays product-owned?
What becomes infrastructure?
```

---

# Staff-Level Canonical Problems

Design:

* Multi-region database
* Global cache
* Global identity system
* Platform-as-a-service
* Company-wide event bus
* Feature flag platform
* Service mesh
* Deployment platform
* Billing platform
* Search platform
* Data platform
* ML platform

---

# The Compression

Senior engineers master the following graph:

```text
Request
    ↓
Load Balancer
    ↓
Service
    ↓
Cache
    ↓
Database
    ↓
Replication
    ↓
Observability
```

Staff engineers master the graph above **plus**:

```text
Organization
    ↓
Ownership
    ↓
Boundaries
    ↓
Contracts
    ↓
Evolution
    ↓
Economics
    ↓
Strategy
```

A concise way to think about the transition:

```text
Junior:
    Can build features.

Mid:
    Can build services.

Senior:
    Can build reliable distributed systems.

Staff:
    Can evolve an ecosystem of systems and teams without losing velocity.
```

If your goal is interview preparation specifically, the ordering changes slightly toward the patterns emphasized by companies such as Meta, Google, Amazon, and Stripe, where the curriculum becomes heavily pattern-oriented rather than theory-oriented.

