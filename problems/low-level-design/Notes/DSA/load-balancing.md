Load balancing is fundamentally a **resource allocation problem**:

> Given a set of incoming requests and a set of servers with varying capacity and state, choose the server that optimizes a desired objective (latency, fairness, throughput, cost, locality, reliability).

Formally:

```
Requests (jobs)
       |
       v
Load Balancer
       |
Scheduling Algorithm
       |
Server Pool
```

This is very similar to **CPU scheduling**, **parking lot allocation**, **cache eviction**, and **distributed resource management**.

---

# 1. Round Robin (RR)

## Idea

Cycle through servers in order.

```
Servers:

S1 S2 S3 S4

Requests:

R1 -> S1
R2 -> S2
R3 -> S3
R4 -> S4
R5 -> S1
```

---

## Data Structure

Just a circular index.

```python
servers = [S1, S2, S3]
current = 0

def get_server():
    global current
    s = servers[current]
    current = (current + 1) % len(servers)
    return s
```

---

## Complexity

```
Time: O(1)
Memory: O(n)
```

---

## Advantages

* Extremely simple
* No monitoring needed
* Equal distribution if requests are similar

---

## Problems

Assumes:

```
Request cost is equal
```

but reality:

```
R1 = 1ms
R2 = 10 seconds
R3 = 50ms
```

Round robin can overload a server.

---

# 2. Weighted Round Robin

Not all machines are equal.

Example:

```
S1: 8 CPU cores
S2: 4 CPU cores
S3: 2 CPU cores
```

Assign weights:

```
S1 = 4
S2 = 2
S3 = 1
```

Distribution:

```
S1 S1 S1 S1 S2 S2 S3
```

---

## Data Structures

Usually:

* Array of server-weight pairs
* Running counters

Example:

```text
[
 (S1, 4),
 (S2, 2),
 (S3, 1)
]
```

---

## Use cases

Good when:

* Server capacities differ
* Load is roughly predictable

---

# 3. Least Connections

## Idea

Send the request to the server with the fewest active requests.

Example:

```
S1: 120 connections
S2: 30 connections
S3: 40 connections

Choose S2
```

---

## Data Structures

You need to find the minimum quickly.

## Naive

Scan all servers.

```
O(n)
```

---

## Better: Min Heap

Store:

```
(active_connections, server)
```

Example:

```
          S2:30
          /   \
      S3:40   S1:120
```

---

### Allocation

Pop the minimum:

```
S2
```

Increase:

```
30 -> 31
```

Reinsert.

---

## Complexity

```
Pick server: O(log n)
Update: O(log n)
```

---

## Good for

Long-lived connections:

* WebSockets
* Database pools
* TCP connections

---

# 4. Weighted Least Connections

Combine capacity and load.

Instead of:

```
connections
```

use:

```
load = connections / weight
```

Example:

```
S1:
100 connections
weight = 10

load = 10


S2:
20 connections
weight = 1

load = 20
```

Choose:

```
S1
```

because relative utilization is lower.

---

## Data Structure

Min heap:

```
(load, server)
```

---

# 5. Least Response Time

Choose the server with the lowest observed latency.

Metrics:

```
score =
    response_time +
    active_connections
```

Example:

```
S1: 5 ms
S2: 100 ms
S3: 20 ms

Choose S1
```

---

## Data Structures

Need metrics collection:

```
Server {
    active_connections
    avg_latency
}
```

Often:

* Min heap
* Balanced tree
* Sorted set

---

# 6. Random Selection

Pick a random server.

```python
random.choice(servers)
```

---

## Complexity

```
O(1)
```

---

Surprisingly good because of the **power of large numbers**.

---

# 7. Power of Two Choices (P2C)

One of the most important modern algorithms.

---

## Idea

Instead of looking at all servers:

1. Pick two random servers
2. Compare their load
3. Choose the better one

Example:

```
Random:
S3: 90 connections
S7: 10 connections

Choose S7
```

---

## Data Structure

Array:

```
servers[]
```

Need only:

```
random index
```

---

## Complexity

```
O(1)
```

---

## Why it is amazing

Random balancing has a maximum load roughly:

```
log n / log log n
```

above average.

With two choices:

```
log log n
```

above average.

A tiny amount of information gives a huge improvement.

---

Used by:

* Large distributed systems
* Cloud load balancers
* Databases
* Distributed caches

---

# 8. Consistent Hashing

Used when requests have affinity.

Example:

```
User 123
always goes to the same server
```

Useful for:

* Cache servers
* Session affinity
* Sharded databases

---

## Hash Ring

```
             S1
          /      \
       key        S2
          \      /
             S3
```

Hash:

```
hash(request_key)
```

Move clockwise until a server is found.

---

## Data Structures

Usually:

### Sorted array

```
[
 hash(S1),
 hash(S2),
 hash(S3)
]
```

Use binary search.

---

### Balanced tree

```
Red-Black Tree
```

Operations:

```
Lookup: O(log n)
Add server: O(log n)
Remove: O(log n)
```

---

## Why not modulo hashing?

Bad:

```
server = hash(key) % N
```

Adding one server:

```
N = 10 -> 11
```

moves almost every key.

---

Consistent hashing only moves roughly:

```
1 / N
```

of keys.

---

# 9. Queue-Based Load Balancing

Instead of choosing a server directly:

```
Requests
   |
 Queue
   |
Workers
```

Examples:

* Job systems
* Batch processing
* Event processing

---

## Data Structures

### FIFO Queue

```
deque
```

Operations:

```
enqueue O(1)
dequeue O(1)
```

---

### Priority Queue

Prioritize important jobs:

```
(priority, task)
```

Implemented with:

```
heap
```

---

# 10. Modern Adaptive Algorithms

Production systems use feedback.

Example score:

```
score =
    α * CPU
  + β * memory
  + γ * latency
  + δ * queue length
```

Choose:

```
argmin(score)
```

---

## Data Structures

```
ServerMetrics {
    cpu_usage
    memory_usage
    latency
    queue_length
}
```

Usually stored in:

* Hash maps
* Heaps
* Time-series windows

---

# Summary Table

| Algorithm                  | Data Structure   | Select Cost   | Best For                   |
| -------------------------- | ---------------- | ------------- | -------------------------- |
| Round Robin                | Circular array   | O(1)          | Uniform requests           |
| Weighted RR                | Array + counters | O(1)          | Different capacities       |
| Least Connections          | Min heap         | O(log n)      | Long connections           |
| Weighted Least Connections | Min heap         | O(log n)      | Heterogeneous clusters     |
| Least Response Time        | Heap/tree        | O(log n)      | Latency-sensitive services |
| Random                     | Array            | O(1)          | Very large systems         |
| Power of Two Choices       | Array            | O(1)          | Modern distributed systems |
| Consistent Hashing         | Hash ring + tree | O(log n)      | Caches/sharding            |
| Queue-based                | Queue/heap       | O(1)/O(log n) | Background jobs            |
| Adaptive                   | Metrics + heap   | Depends       | Cloud-scale systems        |

---

# The deeper abstraction

Nearly every load balancer is solving:

[
\text{Choose } s = \arg\min_{server} \text{Cost}(server)
]

where cost might be:

* Number of connections
* CPU utilization
* Latency
* Queue length
* Memory pressure
* Network distance
* Failure probability

The **data structure exists to answer one question efficiently**:

> “Which resource has the minimum cost under some changing metric?”

This is why heaps, balanced trees, hash rings, queues, and arrays appear repeatedly.

For frontier systems engineering, the next concepts to study are:

* **Distributed schedulers** (Kubernetes scheduler, Mesos)
* **Queueing theory** (M/M/1, M/M/k, Little’s Law)
* **Fair scheduling** (weighted fair queueing, deficit round robin)
* **Flow control and backpressure**
* **Resource allocation algorithms**
* **Congestion control**
* **Online algorithms and competitive analysis**

These are the theoretical foundations behind large-scale AI infrastructure, GPU cluster schedulers, and cloud load balancers.
