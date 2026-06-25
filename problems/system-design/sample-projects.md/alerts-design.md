This is one of the central problems of realtime systems:

> **How do clients converge to the correct state if push delivery is unreliable and clients disconnect?**

The answer is almost always:

```text
Push for freshness
Pull for correctness
```

or equivalently:

```text
WebSocket = notification channel
Database/API = source of truth
```

---

# Naive Push Model (Incorrect)

```mermaid
sequenceDiagram

participant Backend
participant Client

Backend->>Client: Alert #17

Note over Client: Client goes offline

Note over Client: Alert lost forever
```

This only works for ephemeral data like:

* typing indicators
* cursor position
* "user is online"
* live mouse movement

---

# Production Model

```mermaid
sequenceDiagram

participant Backend
participant DB
participant Client

Backend->>DB: Store Alert #17
Backend->>Client: Push Alert #17

Note over Client: Client goes offline

Note over Client: Client reconnects
Client->>Backend: Fetch alerts since last_seen
Backend->>DB: Query missed alerts
DB-->>Backend: [17]
Backend-->>Client: Replay Alert #17
```

Push merely reduces latency.

The database guarantees correctness.

---

# Formal Model

Client maintains:

```text
last_seen_event_id = 1042
```

Server stores:

```text
1040
1041
1042
1043
1044
1045
```

Client reconnects:

```http
GET /alerts?after=1042
```

Server returns:

```json
[
    { "id": 1043 },
    { "id": 1044 },
    { "id": 1045 }
]
```

Client catches up.

---

# Sequence Numbers

This is the simplest approach.

```text
event_id
-------
1
2
3
4
5
6
```

Reconnect:

```text
give me everything after 5
```

Used by:

* chat systems
* notification systems
* feeds
* event sourcing systems

---

# Timestamp Based

Alternative:

```text
last_timestamp = 10:05:31
```

Reconnect:

```http
GET /alerts?since=10:05:31
```

Problems:

* clock skew
* duplicate timestamps
* ordering ambiguity

Sequence IDs are usually better.

---

# Snapshot + Delta Pattern

Large systems often do:

```text
load snapshot
+
apply deltas
```

Example:

```mermaid
sequenceDiagram

participant Client
participant Server

Client->>Server: GET current state
Server-->>Client: cpu=80,mem=70

Server->>Client: cpu=81
Server->>Client: cpu=82
Server->>Client: cpu=84
```

If websocket disconnects:

```text
ignore missed deltas
reload snapshot
continue receiving deltas
```

This is extremely common.

---

# Monitoring Dashboard Example

Suppose dashboard shows:

```text
CPU: 93%
Memory: 78%
Disk: 40%
```

If client disconnects for 2 minutes:

Do you need:

```text
CPU:
91
92
93
94
95
96
95
94
93
```

Usually no.

Instead:

```mermaid
sequenceDiagram

participant React
participant API

React->>API: GET current metrics
API-->>React: CPU=93 MEM=78

React->>API: Open websocket

API->>React: CPU=94
API->>React: CPU=95

Note over React: Disconnect

Note over React: Reconnect

React->>API: GET current metrics
API-->>React: CPU=92 MEM=75
```

This is called:

> **snapshot + streaming updates**

and is probably what you want for monitoring.

---

# Chat Example

Chat is different.

Missing:

```text
hello
how are you
want lunch?
```

is unacceptable.

Therefore:

```mermaid
sequenceDiagram

participant Client
participant Server
participant DB

Server->>DB: Persist Message #100

Note over Client: Client goes offline

Note over Client: Client reconnects
Client->>Server: Last message id = 95
Server->>DB: Fetch 96-100
Server-->>Client: Replay messages
```

---

# Event Classification

| Event Type       | Store? | Replay? |
| ---------------- | ------ | ------- |
| CPU metric       | No     | No      |
| Typing indicator | No     | No      |
| Presence update  | No     | No      |
| Alert created    | Yes    | Yes     |
| Chat message     | Yes    | Yes     |
| Notification     | Yes    | Yes     |
| Trade execution  | Yes    | Yes     |
| Payment event    | Yes    | Yes     |

---

# Monitoring System Recommendation

For your FastAPI + React monitor:

## Backend

```text
metrics table
alerts table
```

Only alerts are durable.

Metrics can expire after:

```text
1 hour
24 hours
7 days
```

depending on needs.

---

## Frontend Startup

```mermaid
sequenceDiagram

participant React
participant API
participant WS

React->>API: GET active alerts
API-->>React: Current alerts

React->>WS: Connect websocket

WS->>React: New alert
WS->>React: Alert resolved
```

---

## Reconnection

```mermaid
sequenceDiagram

participant React
participant API

Note over React: Disconnect

Note over React: Reconnect

React->>API: GET active alerts
API-->>React: Full current state

React->>API: Reopen websocket
```

No replay logic required.

---

# General Rule

There are three canonical strategies:

| Strategy         | Example                       |
| ---------------- | ----------------------------- |
| Snapshot only    | dashboard refresh             |
| Snapshot + delta | monitoring systems            |
| Event log replay | chat, finance, event sourcing |

Most realtime dashboards use **snapshot + delta**, while most messaging systems use **event log replay**. The distinction is whether your application cares about the **current state** or the **history of transitions that produced that state**.
