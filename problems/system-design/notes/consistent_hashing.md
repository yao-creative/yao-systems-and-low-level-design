## 1. Formalization of Consistent Hashing

### Problem Definition

You have:

* A set of **objects** (keys, requests, users, sessions, files):
  $$
  K = {k_1,k_2,...}
  $$

* A set of **storage or compute nodes**:
  $$
  N = {n_1,n_2,...,n_m}
  $$

You need a placement function:

$$
f: K \rightarrow N
$$

such that:

1. **Load balance**
   $$
   P(f(k)=n_i) \approx \frac{1}{m}
   $$

2. **Minimal movement on topology changes**
   If a node is added or removed:

   $$
   |M| \ll |K|
   $$

   where (M) is the set of keys that move.

---

## 2. Naive Hashing

Traditional modulo sharding:

$$
f(k)=H(k)\bmod m
$$

Example:

| Key | Hash | Mod 4 |
| --- | ---- | ----- |
| A   | 15   | 3     |
| B   | 22   | 2     |
| C   | 11   | 3     |
| D   | 7    | 3     |

Add a fifth node:

$$
f(k)=H(k)\bmod 5
$$

Almost every key changes ownership.

Movement fraction:

$$
P(\text{move}) \approx 1
$$

This is catastrophic for caches.

---

## 3. Consistent Hashing Construction

Define a hash space:

$$
$$0,2^{32})
$$

Map both:

* nodes
* keys

into the same space.

$$
h:N\cup K\rightarrow $$0,2^{32})
$$

Example ring:

```
0 ---------------------------------- 2^32
|                                    |
|                                    |
|                                    |
+------------------------------------+
```

Suppose:

```
Node A -> 100
Node B -> 500
Node C -> 900
```

and keys:

```
key1 -> 120
key2 -> 450
key3 -> 700
key4 -> 980
```

Assignment rule:

> Traverse clockwise until finding first node.

Thus:

```
120 -> B
450 -> B
700 -> C
980 -> A
```

because after 980 the ring wraps around.

Formally:

$$
f(k)=\min_{clockwise}(n)
$$

or:

$$
f(k)=
\arg\min_{n \in N}
((h(n)-h(k)) \bmod 2^{32})
$$

---

## 4. Node Addition

Initial:

```
A(100)
B(500)
C(900)
```

Add:

```
D(650)
```

Only keys in interval:

$$
(500,650$$
$$

move.

Expected movement:

$$
P(move)=\frac{1}{m+1}
$$

compared with:

$$
P(move)\approx1
$$

for modulo hashing.

---

## 5. Node Removal

Remove node B:

Only keys owned by B move:

```
(100,500$$
```

Those keys now move to C.

Movement:

$$
P(move)=\frac1m
$$

---

## 6. Virtual Nodes (VNodes)

Real systems rarely place a node once.

Instead:

```
Node A:
A1
A2
A3
A4

Node B:
B1
B2
B3
B4
```

Each physical machine owns many virtual positions.

Formally:

$$
V_i={v_{i1},...,v_{ik}}
$$

Placement:

$$
f(k)=owner(v_j)
$$

where (v_j) is the next vnode clockwise.

---

### Why?

Without vnodes:

```
A owns 60%
B owns 20%
C owns 20%
```

With 100 vnodes:

```
A owns 33%
B owns 34%
C owns 33%
```

Variance decreases approximately as:

$$
\sigma^2 \propto \frac1V
$$

where (V) is vnode count.

---

## 7. Weighted Consistent Hashing

Suppose:

| Node | Capacity |
| ---- | -------- |
| A    | 1x       |
| B    | 2x       |
| C    | 4x       |

Assign vnode count proportional to weight:

```
A -> 100 vnodes
B -> 200
C -> 400
```

Then expected traffic:

$$
P(f(k)=n_i)
===========

\frac{w_i}
{\sum_j w_j}
$$

---

## 8. Alternative Algorithms

### Rendezvous Hashing

Choose:

$$
f(k)
====

\arg\max_i H(k,n_i)
$$

No ring needed.

Properties:

* O(number of nodes)
* simple implementation
* excellent balance

Widely used in modern systems.

Examples:

* Apache Kafka partition leadership
* Envoy load balancing
* Apache Ignite

---

### Jump Hash

Complexity:

$$
O(1)
$$

Memory:

$$
O(1)
$$

Common in large-scale storage systems.

---

## 9. Canonical Uses

| System                   | Object         | Node            |
| ------------------------ | -------------- | --------------- |
| Cache cluster            | key            | cache server    |
| CDN                      | object         | edge server     |
| Session store            | session id     | redis shard     |
| Database sharding        | user id        | shard           |
| Object storage           | object id      | storage node    |
| Recommendation systems   | user embedding | embedding shard |
| Vector DB                | vector id      | partition       |
| Distributed lock service | lock key       | coordinator     |

---

## 10. Local Simulation Exercises

### Exercise 1: Build Ring

Implement:

```python
add_node(node)
remove_node(node)
get_node(key)
```

Goal:

```text
1000 keys
3 nodes
```

Measure:

```text
keys per node
standard deviation
```

---

### Exercise 2: Measure Rebalancing

1. Create:

```text
10,000 keys
3 nodes
```

2. Add node D.

3. Compute:

```text
moved_keys / total_keys
```

Expected:

```text
~25%
```

---

### Exercise 3: Compare Against Modulo Hashing

Run:

```text
hash(key) % n
```

versus consistent hashing.

Measure:

```text
fraction of keys moved
```

Expected:

| Method     | Moved    |
| ---------- | -------- |
| Modulo     | ~80-100% |
| Consistent | ~25%     |

---

### Exercise 4: Virtual Nodes

Try:

```text
1 vnode
5 vnodes
20 vnodes
100 vnodes
```

Measure:

```text
stddev(keys_per_node)
```

Observe convergence.

---

### Exercise 5: Weighted Nodes

Simulate:

```
A weight 1
B weight 2
C weight 4
```

Verify:

```
14%
29%
57%
```

distribution.

---

### Exercise 6: Node Failure Storm

Start:

```text
20 nodes
1 million keys
```

Randomly kill:

```text
5 nodes
```

Measure:

* moved keys
* new load distribution
* hotspot formation

This approximates real cache clusters.

---

## 11. Mini-Cluster Practice

The easiest local setup:

```text
Docker container 1 -> cache node A
Docker container 2 -> cache node B
Docker container 3 -> cache node C
Client -> consistent hash router
```

Route requests by:

```python
node = ring.get_node(user_id)
```

Then:

```bash
docker stop node2
```

Observe:

* only node2's keys migrate
* other keys remain stable

This reproduces production behavior in systems such as:

* Amazon Web Services cache layers
* Cloudflare edge routing
* Discord gateway sharding
* Uber service partitioning

---

## Compression

Consistent hashing is fundamentally:

> Construct a metric space over nodes and keys such that topology mutations induce local perturbations rather than global reassignment.

Modulo hashing:

$$
f(k)=H(k)\bmod n
$$

changes globally.

Consistent hashing:

$$
f(k)
====

\arg\min_{clockwise}
((h(n)-h(k))\bmod M)
$$

changes locally.
