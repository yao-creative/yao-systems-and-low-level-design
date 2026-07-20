A **database lease** is a **time-bounded ownership claim** stored in a database that allows one process/client/node to temporarily become the authoritative actor for some resource.

The core problem it solves:

> "How do multiple distributed actors coordinate ownership when they can crash, disconnect, or run concurrently?"

A lease is a controlled form of **temporary lock**.

---

## 1. Intuition

Imagine a distributed scheduler with 10 workers.

You want exactly one worker to run a periodic job:

```
send_daily_report()
```

You cannot simply use:

```
if nobody is running:
    run job
```

because two workers can observe "nobody running" simultaneously.

Instead:

```
Worker A:
"I own this job until 12:00:10"

Worker B:
"Can I take ownership?"
"No, lease is still valid"

12:00:10 passes

Worker B:
"Lease expired. I acquire it."
```

The database becomes the **source of truth**.

---

# 2. Basic lease data model

Usually a lease is a row:

```sql
CREATE TABLE leases (
    resource_id TEXT PRIMARY KEY,
    owner_id TEXT,
    expires_at TIMESTAMP,
    version BIGINT
);
```

Example:

| resource | owner    | expiry   |
| -------- | -------- | -------- |
| job_123  | worker_A | 12:00:10 |

Meaning:

```
worker_A owns job_123
until timestamp 12:00:10
```

---

# 3. Acquisition algorithm

The critical operation is:

> "Acquire only if nobody owns it OR ownership expired."

SQL:

```sql
UPDATE leases
SET
    owner_id = 'worker_B',
    expires_at = NOW() + INTERVAL '10 seconds',
    version = version + 1
WHERE resource_id = 'job_123'
AND (
    expires_at < NOW()
    OR owner_id = 'worker_B'
);
```

Then check:

```
rows_updated == 1
```

Success:

```
worker_B owns lease
```

Failure:

```
someone else owns it
```

---

# 4. The important concept: lease ≠ mutex

A mutex says:

$$
\exists! x : Owner(resource)=x
$$

"there exists exactly one owner"

Forever.

A lease says:

$$
Owner(resource,t)=x
$$

only during an interval:

$$
t \in [start,end]
$$

Ownership is a function of time:

$$
L : Resource \times Time \rightarrow Owner \cup {\bot}
$$

Example:

$$
L(job,10s)=A
$$

but:

$$
L(job,20s)=B
$$

The ownership naturally expires.

---

# 5. Formal model

A lease can be modeled as:

$$
Lease =
(Resource, Owner, Expiration, Epoch)
$$

where:

* `Resource`

  * what is being owned

* `Owner`

  * identity of current holder

* `Expiration`

  * deadline

* `Epoch`

  * version preventing stale owners

---

## Lease validity predicate

A lease is valid if:

$$
Valid(L,t)=
(t < L.expiration)
$$

Ownership predicate:

$$
Owns(x,r,t)=
(L.resource=r)
\land
(L.owner=x)
\land
(t<L.expiration)
$$

---

# 6. Why expiration exists

Distributed systems have failure modes:

Example:

```
Worker A acquires lease

Worker A crashes

Nobody releases lock
```

With a mutex:

```
system dead forever
```

With a lease:

```
wait until expiry

another worker takes over
```

The lease converts:

> permanent failure

into

> temporary unavailability

---

# 7. The hard problem: stale owners

Consider:

```
t=0

Worker A:
acquire lease
expires at t=10


t=11

Worker A:
(network was partitioned)
still thinks it owns resource


Worker B:
acquires lease


Now:

A and B both operate
```

This is called a:

**split brain**

---

# 8. Epoch fencing tokens

The standard solution is a monotonically increasing version.

Lease table:

| resource | owner | epoch |
| -------- | ----- | ----- |
| job      | A     | 1     |

A gets:

```
epoch=1
```

Later:

| resource | owner | epoch |
| -------- | ----- | ----- |
| job      | B     | 2     |

B gets:

```
epoch=2
```

Every write includes the epoch:

```
UPDATE resource
SET value=x
WHERE epoch=2
```

Database rejects:

```
epoch=1
```

because it is stale.

Formalized:

$$
epoch_{new}>epoch_{old}
$$

and:

$$
Write(x,e)
\text{ is valid only if }
e=currentEpoch
$$

This is a **capability revocation mechanism**.

---

# 9. Lease renewal

Usually leases are not held once.

They are renewed:

```
Acquire:
expiration = now + 30s


Every 10s:

UPDATE leases
SET expires_at = now + 30s
WHERE owner=A
```

This is called:

**heartbeat renewal**.

Timeline:

```
0s       acquire
10s      renew
20s      renew
30s      renew
40s      crash
70s      expire
```

---

# 10. Database transaction requirements

Lease correctness depends on atomicity.

The acquire operation must behave as:

$$
Check(expired)
\rightarrow
Write(owner)
$$

as one indivisible transition.

Formally:

$$
State_{old}
\xrightarrow{Acquire}
State_{new}
$$

must be atomic.

Otherwise:

```
Worker A:
read expired=true


Worker B:
read expired=true


both acquire
```

---

# 11. Relationship to locking

You can think of the hierarchy:

```
Synchronization primitives

        |
        v

Mutual exclusion

        |
        v

Locks

        |
        v

Distributed locks

        |
        v

Leases
```

A lease is a distributed lock with:

1. timeout
2. renewal
3. fencing/versioning

---

# 12. Lease vs common alternatives

| Primitive                        | Guarantee                    | Failure handling         |
| -------------------------------- | ---------------------------- | ------------------------ |
| Mutex                            | single process ownership     | deadlock                 |
| DB row lock                      | transaction scoped ownership | connection loss releases |
| Distributed lock                 | cluster ownership            | requires coordination    |
| Lease                            | time bounded ownership       | expiry takeover          |
| Consensus lease (Raft/Zookeeper) | strong distributed ownership | higher cost              |

---

# 13. Category-theoretic view

A lease system is a **state transition system**.

Objects:

$$
S = {Free, Owned, Expired}
$$

Transitions:

$$
Acquire:
Free \rightarrow Owned
$$

$$
Renew:
Owned \rightarrow Owned
$$

$$
Expire:
Owned \rightarrow Expired
$$

$$
Release:
Owned \rightarrow Free
$$

The database provides a morphism:

$$
f:S_t \rightarrow S_{t+1}
$$

with the invariant:

$$
\forall t:
|{owner(resource,t)}| \leq 1
$$

Meaning:

> At every point in time, at most one valid owner exists.

---

# 14. Real-world uses

* leader election
* distributed cron jobs
* Kubernetes controllers
* singleton workers
* task claiming
* queue consumers
* service discovery
* cache ownership
* database migrations

Examples:

* Kubernetes uses lease objects for leader election.
* Apache ZooKeeper uses ephemeral nodes, which are conceptually related to leases.

---

The compressed mental model:

$$
\boxed{
Lease =
Lock
+
Timeout
+
Renewal
+
Fencing\ Token
}
$$

The deepest idea is that a lease is not primarily a locking primitive; it is a **distributed ownership protocol under partial failure**.
