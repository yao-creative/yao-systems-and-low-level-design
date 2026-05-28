# LLD Drill Format

Use this file as the repeatable practice format for interview drills.

Goal:

> produce the same artifacts in the same order until the flow becomes automatic

Time target:

- 15 to 20 minutes for design only
- 25 to 35 minutes if adding code skeletons

---

# 1. Drill Output Checklist

For each problem, produce all 9 artifacts:

1. Requirements
2. Invariants
3. State machine
4. Core entities
5. Responsibilities and ownership
6. Interfaces
7. Data structures and concurrency notes
8. One happy path and one failure path
9. One requirement change and impact analysis

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

## 2. Invariants

Write 3 to 5 always-true statements.

Examples:

- only one active trip per driver
- a booked seat cannot be sold twice
- an order in `DELIVERED` cannot return to `PAID`

Rule:

- if an invariant is vague, rewrite it until you can point to the owner that enforces it

## 3. State Machine

Write:

- states
- legal transitions
- illegal transitions

Template:

```text
STATE_A -> STATE_B
STATE_B -> STATE_C
STATE_B -> FAILED
```

Rule:

- if the problem has lifecycle, draw this before class design

## 4. Core Entities

List only major state carriers.

For each entity, write:

- what state it owns
- why it exists

Rule:

- attached data is a field, not an entity

## 5. Responsibilities and Ownership

For each important rule or transition, answer:

- which object owns the state?
- which object performs the mutation?
- which object enforces the invariant?

Template:

```text
Rule / Transition -> Owner -> Mutator -> Enforcement point
```

This is the most important section.

## 6. Interfaces

Only add interfaces where variation is real.

For each interface, write:

- what varies
- why a contract is needed

Template:

```text
PricingStrategy -> pricing rules vary
AllocationStrategy -> assignment policy varies
```

## 7. Data Structures and Concurrency

Write:

- dominant operations
- DS choice
- atomicity or race concerns

Template:

```text
Operation -> DS -> Why
```

```text
Shared state -> race risk -> control point
```

## 8. Trace Two Flows

Write one:

- happy path
- failure path

Template:

```text
request -> validate -> transition -> side effect -> new state
```

Rule:

- if the trace exposes missing ownership, go back to section 5

## 9. Requirement Change

Add one realistic change:

- new pricing rule
- new role
- retry support
- cancellation window
- concurrency increase

Then answer:

- what stays stable?
- what changes?
- where does the change land?

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
