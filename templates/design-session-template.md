# Problem

- Name:
- Category: system design | low-level design
- Source:
- Date:
- Timebox:

# Objective

What exactly am I optimizing for in this round?

# Clarifying Questions

- 

# Requirements

## Functional

- 

## Non-Functional

- scale:
- latency:
- consistency:
- availability:
- cost:
- security/privacy:

# Back-of-the-Envelope

- QPS:
- storage:
- bandwidth:
- fanout:
- peak assumptions:

# Core Entities / Data Model

- 

# API / Interface

```text
// sketch endpoints, RPCs, class interfaces, or contracts here
```

# Attempt 1

## High-Level Design

- 

## Read Path

1. 

## Write Path

1. 

## Scaling Strategy

- partitioning:
- caching:
- async work:
- indexing:
- replication:

## Failure Modes

- 

## Tradeoffs

- 

# Socratic Critique

Do not answer immediately. Use these to attack the design.

## Correctness

- What invariant can fail first?
- Where can duplicates happen?
- Where can ordering break?
- What stale-read behavior is acceptable?

## Scale

- What component melts first at 10x traffic?
- What is accidentally centralized?
- Where is fanout hidden?
- Which query becomes impossible without precomputation?

## Reliability

- What happens during regional failure?
- What queue can grow without bound?
- What data loss is currently possible?
- How does recovery work after partial failure?

## Operability

- What metric would warn me first?
- What would be hardest to debug in prod?
- Which dependency has unclear ownership?

## Product / UX

- What user-visible degradation is acceptable?
- What edge case is ignored but common?

## Security / Abuse

- What is the easiest abuse vector?
- Where is rate limiting required?
- What data needs stricter isolation?

# Attempt 2

Revise the design after the critique.

# Final Critique

## What Improved

- 

## Remaining Weaknesses

- 

## Biggest Bottleneck

- 

## Biggest Risky Assumption

- 

## What I Would Build Next

- 

# Frontier AI Applications

How does this problem appear inside frontier AI systems or AI products?

- agent memory / retrieval angle:
- online inference / ranking angle:
- safety / abuse angle:
- data pipeline angle:
- eval / observability angle:

# Variants

- How would this change for 100x write-heavy traffic?
- How would this change for strict multi-region requirements?
- How would this change if cost had to drop by 70%?
- How would this change if personalization became core?

# Challenge Questions

- What assumption did I borrow from a big-tech architecture without justifying it?
- If I had only one database, what would I change?
- If I could not use caching, what would break?
- What exact thing would I page on-call for?

# Next Round

- One thing to study:
- One thing to simulate or prototype:
- One related problem to attempt next:
