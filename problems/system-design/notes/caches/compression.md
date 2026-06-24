| Dimension        | Formalization        | Problem Solved                             | Typical Choices                                    |
| ---------------- | -------------------- | ------------------------------------------ | -------------------------------------------------- |
| **Keying**       | `K → V`              | Identify reusable work                     | URL → response, query → result, user_id → features |
| **Admission**    | `should_store(K,V)?` | Avoid polluting cache with low-value items | Admit all, TinyLFU admission                       |
| **Eviction**     | `choose victim(K)`   | Memory is finite                           | LRU, LFU, ARC, FIFO                                |
| **Expiration**   | `is_valid(K,t)?`     | Data becomes stale                         | TTL, versioning, CDC invalidation                  |
| **Consistency**  | `cache == source?`   | Writes create divergence                   | Cache-aside, write-through, write-back             |
| **Distribution** | `K → node`           | One machine insufficient                   | Consistent hashing, replication                    |

---

# Cache Types

| Cache Type            | Formal Model                   | Resource Saved            | Example                |
| --------------------- | ------------------------------ | ------------------------- | ---------------------- |
| Memoization           | `input → output`               | CPU computation           | Dynamic programming    |
| Response Cache        | `request → response`           | Backend compute + DB      | REST API cache         |
| Query Cache           | `SQL → result`                 | Query execution           | Database result cache  |
| Buffer Cache          | `page_id → page`               | Disk I/O                  | Database buffer pool   |
| CDN Cache             | `path → object`                | Network latency/bandwidth | Images, videos         |
| CPU Cache             | `address → cache line`         | RAM latency               | L1/L2/L3               |
| TLB                   | `virtual_page → physical_page` | Address translation       | MMU                    |
| Feature Cache         | `entity → features`            | ML feature generation     | Recommendation systems |
| Embedding Cache       | `text → embedding`             | Model inference cost      | LLM serving            |
| Request Deduplication | `request_key → future`         | Duplicate concurrent work | SingleFlight           |

---

# Eviction Policies

| Policy  | Formal Rule                   | Behavioral Assumption       | Strength                    | Weakness           |
| ------- | ----------------------------- | --------------------------- | --------------------------- | ------------------ |
| FIFO    | oldest insertion leaves       | age predicts irrelevance    | Simple                      | ignores usage      |
| LRU     | oldest access leaves          | recent ⇒ future reuse       | Great for locality          | fails on scans     |
| LFU     | lowest frequency leaves       | popular stays popular       | Great for stable popularity | stale hot keys     |
| MRU     | newest access leaves          | recent won't repeat         | Sequential scans            | poor generally     |
| Random  | random victim                 | no predictive signal        | Extremely cheap             | suboptimal         |
| ARC     | adaptive recency/frequency    | workload changes over time  | Self-tuning                 | more complexity    |
| TinyLFU | compare estimated frequencies | admit only valuable entries | excellent hit rates         | approximate counts |

---

# Expiration Strategies

| Strategy         | Formalization            | Solves                     |
| ---------------- | ------------------------ | -------------------------- |
| Infinite         | `valid forever`          | Immutable data             |
| TTL              | `t_now < t_expire`       | Eventual freshness         |
| Versioning       | `(key, version) → value` | Instant invalidation       |
| Temporal Cache   | `(key,time) → value`     | Historical queries         |
| CDC Invalidation | DB event → invalidate    | Near-real-time consistency |

---

# Write Strategies

| Strategy      | Read Path          | Write Path               | Tradeoff             |
| ------------- | ------------------ | ------------------------ | -------------------- |
| Cache Aside   | Cache → DB on miss | DB then invalidate cache | Most common          |
| Write Through | Cache first        | Cache + DB synchronously | Strong consistency   |
| Write Back    | Cache first        | DB asynchronously        | High throughput      |
| Write Around  | Cache miss only    | Write DB directly        | Avoid cold pollution |

---

# Distributed Cache Problems

| Problem           | Failure Mode                      | Solution                       |
| ----------------- | --------------------------------- | ------------------------------ |
| Cache Stampede    | many misses hit DB simultaneously | request coalescing             |
| Cache Penetration | nonexistent keys bypass cache     | negative caching, Bloom filter |
| Cache Avalanche   | many expirations simultaneously   | TTL jitter                     |
| Hot Key           | one key overloads one node        | replication, local cache       |
| Rebalancing       | node added/removed                | consistent hashing             |
| Node Failure      | cache machine dies                | replication                    |

---

# Locality Models

| Locality Type         | Meaning                            | Exploited By |
| --------------------- | ---------------------------------- | ------------ |
| Temporal Locality     | recently used → likely reused      | LRU          |
| Frequency Locality    | frequently used → likely reused    | LFU          |
| Spatial Locality      | nearby addresses likely accessed   | CPU caches   |
| Behavioral Adaptation | locality pattern changes over time | ARC          |
| Popularity Filtering  | only admit valuable items          | TinyLFU      |

---

# Unified Cache State Machine

| Step | Operation                        |
| ---- | -------------------------------- |
| 1    | Receive request for key `K`      |
| 2    | Lookup `K`                       |
| 3    | Cache hit → return value         |
| 4    | Cache miss → compute/fetch value |
| 5    | Admission decision               |
| 6    | Evict if capacity exceeded       |
| 7    | Store `(K,V)`                    |
| 8    | Serve result                     |

---

# One-line Summary

| Concept           | One-line Formalization                             |
| ----------------- | -------------------------------------------------- |
| Cache             | `memory used to reduce future cost`                |
| Eviction          | `predict what future will not need`                |
| LRU               | `future ≈ recent past`                             |
| LFU               | `future ≈ historical popularity`                   |
| ARC               | `learn whether recency or popularity matters`      |
| TinyLFU           | `store only items predicted to outperform victims` |
| Distributed Cache | `bounded shared memory across machines`            |
