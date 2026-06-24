# Sharding: Formalization

## Definition

**Sharding** is a **horizontal partitioning strategy** over a dataset.

Formally:

Given a dataset:

[
D = {r_1, r_2, ..., r_n}
]

we define a partition function:

[
P : D \rightarrow {S_1, S_2, ..., S_k}
]

such that:

[
S_i \cap S_j = \emptyset \quad (i \neq j)
]

and

[
\bigcup_{i=1}^{k} S_i = D
]

Each partition (S_i) is called a **shard**.

---

## Goal

Sharding exists to solve:

| Problem            | Without sharding       | With sharding       |
| ------------------ | ---------------------- | ------------------- |
| Storage capacity   | Limited by one machine | Sum of all machines |
| Read throughput    | Limited by one machine | Parallel reads      |
| Write throughput   | Limited by one machine | Parallel writes     |
| Hotspots           | Entire DB overloaded   | Localized overload  |
| Geographic latency | Single region          | Regional shards     |

---

## Canonical abstraction

```
request
   |
routing function
   |
+--+--+--+--+
|     |     |
S1    S2    S3
```

The routing function determines:

[
shard = f(key)
]

---

# Types of shard functions

## 1. Hash sharding

[
shard = hash(user_id) \bmod N
]

Example:

```
user_id=123
hash(123)=987654321

987654321 % 4 = 1

-> shard 1
```

Properties:

* good load balancing
* poor range queries
* difficult resharding

Used by:

* Cassandra
* DynamoDB

---

## 2. Range sharding

```
0 - 1M users -> shard0
1M - 2M users -> shard1
```

Properties:

* efficient range scans
* hotspot risk

Used by:

* MySQL deployments
* MongoDB

---

## 3. Directory sharding

```
user123 -> shard7
user124 -> shard3
```

Lookup table:

| key | shard |
| --- | ----- |
| 123 | 7     |
| 124 | 3     |

Properties:

* flexible
* extra metadata service needed

Used by:

* Instagram style deployments
* Discord

---

## 4. Geographic sharding

```
APAC users -> Singapore
EU users -> Frankfurt
US users -> Virginia
```

Optimizes latency and regulatory requirements.

---

# Consistent hashing

Classic hash sharding fails when adding machines:

```
hash(key) % 4
```

Adding a fifth machine becomes:

```
hash(key) % 5
```

Almost every key moves.

Consistent hashing instead maps:

```
key -> hash ring -> nearest server clockwise
```

Only roughly:

[
\frac{1}{N}
]

of keys move during resharding.

Used heavily in:

* Amazon style distributed systems
* Cassandra

---

# Related concepts

| Concept            | Question answered                   |
| ------------------ | ----------------------------------- |
| Replication        | How many copies exist?              |
| Sharding           | Which machine owns this data?       |
| Partitioning       | Generic splitting mechanism         |
| Consistent hashing | How do ownership changes occur?     |
| Load balancing     | Which machine serves request?       |
| Caching            | Can request avoid storage entirely? |

---

# Local mini-sharding

Yes. This is extremely useful.

You do **not** need multiple machines.

## Method 1: Multiple SQLite files

```
users_0.db
users_1.db
users_2.db
users_3.db
```

Router:

```python
shard = user_id % 4

dbs = {
    0: users0,
    1: users1,
    2: users2,
    3: users3
}

dbs[shard].execute(...)
```

This emulates production hash sharding surprisingly well.

---

## Method 2: Multiple local databases

Run:

```text
localhost:5432
localhost:5433
localhost:5434
localhost:5435
```

using:

* multiple Docker containers
* multiple local processes

Router service chooses destination shard.

This is essentially a miniature distributed database.

---

## Method 3: Container cluster

Run:

* shard1 container
* shard2 container
* shard3 container
* shard4 container
* router container

This approximates real production architecture.

---

## Method 4: Embedded KV stores

Use:

* RocksDB
* LevelDB
* BadgerDB

Create multiple instances:

```text
shard0/
shard1/
shard2/
shard3/
```

---

# Exercises

## Level 1 — Static hash sharding

Build:

```text
put(user_id, value)
get(user_id)
delete(user_id)
```

using:

```python
shard = user_id % 4
```

---

## Level 2 — Add replication

Store every write:

```
primary shard
backup shard
```

Exercise:

```
user_id % 4 -> primary
(user_id + 1) % 4 -> replica
```

---

## Level 3 — Resharding

Move:

```
4 shards
→
8 shards
```

Measure:

* keys moved
* migration duration
* downtime

---

## Level 4 — Consistent hashing

Implement:

```text
hash ring
virtual nodes
clockwise lookup
```

Measure:

| Shards Added       | Keys Moved |
| ------------------ | ---------- |
| modulo hashing     | ~75%       |
| consistent hashing | ~25%       |

---

## Level 5 — Hot shard simulation

Generate:

```text
90% traffic -> user_id=42
10% traffic -> random users
```

Observe:

```
Shard 2 overloaded
Shard 0 idle
Shard 1 idle
Shard 3 idle
```

Then implement:

* key salting
* celebrity splitting
* cache layer

---

## Level 6 — Build mini-Instagram storage

Requirements:

* users
* posts
* followers

Questions:

* shard by user?
* shard by post?
* shard by region?
* where do follower edges live?

This is very close to real interview discussions.

---

## Level 7 — Build a shard router

Architecture:

```text
client
  |
router
  |
+----+----+----+----+
|    |    |    |    |
S0   S1   S2   S3
```

Support:

* adding shards
* removing shards
* moving data
* monitoring imbalance

---

# Canonical interview progression

1. Single database
2. Replication
3. Read replicas
4. Vertical scaling
5. Sharding
6. Consistent hashing
7. Multi-region sharding
8. Automatic rebalancing

This sequence mirrors the evolution path of many large-scale systems such as Facebook, Instagram, and Uber as load grows from one machine to thousands.
