A **cache stampede** (also called a **dogpile effect**) is a failure mode where many concurrent requests all observe a cache miss or expired entry and simultaneously recompute or fetch the same expensive value.

Formally:

[
\text{Cache Stampede} \iff
\exists k \gg 1 :
\forall i \in [1,k],
\text{request}_i \rightarrow \text{same missing key}
]

Instead of:

```
1000 requests
    ↓
1 database query
    ↓
999 cache hits
```

you get:

```
1000 requests
    ↓
1000 database queries
    ↓
database overload
```

This is fundamentally a **coordination problem over shared state under concurrency**.

---

# Canonical Example

## Product Page

Suppose:

```text
GET /product/123
```

takes:

```text
DB query = 500ms
Cache TTL = 5 minutes
Traffic = 2000 requests/sec
```

At:

```text
12:00:00
```

cache expires.

Then:

```text
12:00:00.001
```

200 requests arrive.

All observe:

```text
cache miss
```

All issue:

```sql
SELECT * FROM products WHERE id=123
```

Database receives:

```text
200 identical expensive queries
```

which increases latency:

```text
500ms → 10s
```

which causes:

```text
more requests waiting
→ more retries
→ even more load
```

potentially causing cascading failure.

---

# Real Systems Examples

## 1. Homepage Feed Generation

A recommendation feed:

```text
feed:user:123
```

expires every minute.

At expiration:

```text
5000 app servers
```

all regenerate recommendations.

Common in:

* social networks
* video recommendations
* news feeds

---

## 2. ML Feature Store

Feature:

```text
user_embedding:123
```

expires.

Thousands of inference workers recompute embeddings simultaneously.

---

## 3. Leaderboards

Gaming leaderboard cache:

```text
leaderboard:global
```

expires.

Millions of players refresh ranking simultaneously.

---

## 4. Authentication Metadata

JWT public keys:

```text
jwks.json
```

expire.

Every service fetches new keys from auth server.

Auth server dies.

---

## 5. Flash Sale

Product inventory cache:

```text
inventory:iphone17
```

expires during launch.

Every request hits database.

Classic e-commerce outage scenario.

---

# Solutions

## 1. Request Coalescing / Single Flight

Only one request computes.

Others wait.

```text
request 1 → acquire lock → compute
request 2 → wait
request 3 → wait
request 4 → wait

request 1 finishes
↓
everyone reads cache
```

Complexity:

```text
N DB queries
↓
1 DB query
```

Examples:

* Go singleflight
* distributed locks
* promise deduplication

---

## 2. Stale While Revalidate

Serve expired value temporarily.

```text
cache expired
↓
serve old value
↓
background refresh
```

Users see slightly stale data but avoid overload.

Used heavily by:

* CDNs
* API gateways
* browsers

---

## 3. Probabilistic Early Expiration

Instead of:

```text
expire exactly at 12:00
```

randomize:

```text
11:58
11:59
12:00
12:01
```

Removes synchronization.

---

## 4. TTL Jitter

Instead of:

```text
TTL = 300s
```

use:

```text
TTL = 300 ± random(0,60)
```

Prevents mass expiration.

---

## 5. Never Expire Hot Keys

Instead:

```text
cache forever
```

Refresh asynchronously.

Very common for:

* configurations
* product catalog
* feature flags

---

## 6. Rate Limiting Regeneration

Allow only:

```text
5 cache rebuilds/sec
```

Everything else receives stale value.

---

# Relationship To Other Distributed Systems Problems

| Problem           | Failure Mode                           |
| ----------------- | -------------------------------------- |
| Cache stampede    | Too many readers regenerate same value |
| Thundering herd   | Many waiters awaken simultaneously     |
| Retry storm       | Retries amplify failures               |
| Hot key           | One key dominates traffic              |
| Cache avalanche   | Many keys expire simultaneously        |
| Cache penetration | Requests for nonexistent keys          |

A useful hierarchy is:

```text
Cache failures
│
├── Stampede
│   └── one hot key expires
│
├── Avalanche
│   └── many keys expire together
│
├── Penetration
│   └── nonexistent key bypasses cache
│
└── Hot key
    └── single key receives disproportionate traffic
```

---

# Practice Problems

## Level 1 — Identify the Failure Mode

1. A cache entry for:

```text
weather:singapore
```

expires every hour.

10,000 requests arrive immediately afterward.

What happens?

---

2. A CDN caches an image for:

```text
600 seconds
```

and all edge nodes refresh simultaneously.

What distributed systems issue occurred?

---

3. An API retries on timeout while cache regeneration is ongoing.

What secondary failure mode can emerge?

---

# Level 2 — Design Questions

## Problem 1

Design:

```text
GET /product/:id
```

Requirements:

* 50k RPS
* DB query = 200ms
* cache TTL = 5 min
* avoid stampede

What algorithm do you use?

---

## Problem 2

Design a cache for:

```text
global leaderboard
```

Requirements:

* updates every second
* 5 million reads/minute
* users tolerate 5 second staleness

---

## Problem 3

Design:

```text
user recommendations
```

Requirements:

* generation takes 3 seconds
* requests arrive every 100ms
* recommendations may be stale by 10 minutes

---

# Coding Exercises

## Easy

Implement:

```python
get_or_compute(key, fn)
```

such that:

```text
100 concurrent requests
↓
only one fn() executes
```

---

## Medium

Implement:

```python
singleflight(key, fn)
```

using:

* mutex
* condition variable
* futures/promises

---

## Medium

Implement:

```python
cache.get_stale(key)
```

which returns:

```text
(value, expired)
```

and refreshes in background.

---

## Hard

Build a distributed cache with:

* Redis
* distributed lock
* stale-while-revalidate
* lock timeout handling

---

## Very Hard

Design a cache layer handling:

* 1M RPS
* hot keys
* regional failover
* retry storms
* cache stampede
* cache avalanche

This is essentially the caching layer behind companies like Netflix, Amazon, or Meta.

---

# Interview Compression

A cache stampede is:

> **Many concurrent requests attempting the same expensive recomputation because coordination around cache invalidation is missing.**

The standard mitigation hierarchy is:

```text
Best:
    stale-while-revalidate
        +
    request coalescing

Good:
    distributed lock

Helpful:
    TTL jitter

Additional:
    rate limiting
```

Most production systems use **multiple layers simultaneously** rather than relying on a single technique.
