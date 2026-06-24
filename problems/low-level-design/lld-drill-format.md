# LLD Drill Format

Use this file as the repeatable practice format for interview drills.

Goal:

> produce the same artifacts in the same order until the flow becomes automatic

Time target:

- 15 to 20 minutes for design only
- 25 to 35 minutes if adding code skeletons

---

# 1. Drill Output Checklist

For each problem, produce all 10 artifacts:

1. Requirements
2. Core entities
3. Attributes and methods
4. Relationships
5. State machine
6. Responsibilities and ownership
7. Happy path and failure path
8. Apply design patterns
9. Concurrency notes
10. Extensibility

---

# 2. Drill Template

## Problem

Write the prompt in one sentence.

## 1. Requirements

Fill in:

- core capabilities
- assumptions
- out of scope
- edge cases

Limit:

- 4 to 7 bullets total

## 2. Core Entities

List only major state carriers.

For each entity, write:

- what state it owns
- why it exists

Rule:

- attached data is a field, not an entity

## 3. Attributes and Methods

For each entity, write:

- fields with types
- method signatures or responsibilities

Rule:

- do not jump to implementation before the entity surface is stable

## 4. Relationships

Write how entities relate:

- composition
- aggregation
- association
- dependency

Template:

```text
EntityA -> EntityB -> relationship type
```

Rule:

- if the relationship is vague, rewrite it until ownership direction is clear

## 5. State Machine

Write:

- states
- legal transitions
- illegal transitions

Rule:

- if the problem has lifecycle, draw this before class design

## 6. Responsibilities and Ownership

For each important rule or transition, answer:

- which object owns the state?
- which object performs the mutation?
- which object enforces the rule?

Template:

```text
Rule / Transition -> Owner -> Mutator -> Enforcement point
```

This is the highest-value structural step.

## 7. Happy Path and Failure Path

Trace one:

- happy path
- failure path

Rule:

- if the trace exposes missing ownership or a missing transition, go back to section 5

## 8. Apply Design Patterns

Only add patterns where variation is real.

For each pattern, write:

- what varies
- why a contract is needed

## 9. Concurrency Notes

Write:

- dominant operations
- DS choice
- shared-state or interleaving risks

Template:

```text
Operation -> DS -> Why
```

```text
Shared state -> race risk -> control point
```

## 10. Extensibility

Write:

- one realistic change axis
- what stays stable
- what changes
- where the change lands

Rule:

- if the change forces broad rewiring, the earlier decomposition is too loose

---

# 3. Scoring Rubric

After each drill, grade yourself on 5 questions:

1. Did I clarify scope before modeling?
2. Did I make invariants explicit?
3. Did I assign ownership for every important mutation?
4. Did I avoid fake abstractions?
5. Could I explain one change request without restructuring everything?

If any answer is `no`, that is the next thing to practice.

---

# 4. Practice Sequence

Start with workflow-heavy problems:

1. Parking lot
2. Elevator
3. ATM
4. Vending machine
5. Food order or ride trip

Then move to structure-heavy problems:

1. Library
2. Splitwise
3. Chess
4. Logger
5. File system

---

# 5. Final Compression

In every drill, force yourself to answer:

1. What are we building?
2. What must never break?
3. What changes over time?
4. Who owns the state?
5. Who is allowed to mutate it?
6. Where is variation real?
7. How does one success path run?
8. How does one failure path break safely?
9. What future change should this survive?
