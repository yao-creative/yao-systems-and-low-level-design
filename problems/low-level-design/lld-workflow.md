# LLD Interview Workflow

This file owns the **practical interview sequence**.
The conceptual explanation for why this sequence works lives in `lld-why.md`.

---

# 1. Compressed Workflow

```text
Requirements
    ↓
Invariants
    ↓
State Machine
    ↓
Entities
    ↓
Responsibilities
    ↓
Interfaces
    ↓
Data Structures
    ↓
Concurrency / Failure
    ↓
Extensibility
    ↓
Trace One Happy Path + One Failure Path
```

```
RISERIDCE
```

---

# 2. Fast Interview Order

Use this order in a live interview.

## 1. Clarify requirements

Ask for:

- core capabilities
- assumptions
- scope boundaries
- edge cases

Output:

- a short agreed spec

## 2. Define invariants

Ask:

- what must never be violated?
- what states are illegal?

Examples:

- one active ticket per vehicle
- no double booking
- completed order cannot be cancelled

Output:

- 3 to 5 always-true statements

## 3. Draw the state machine

Ask:

- what changes over time?
- what transitions are legal?
- what transitions must be rejected?

Output:

- lifecycle states
- allowed transitions
- terminal or failure states if relevant

## 4. Identify core entities

Only list major entities.

Filter:

- if it owns changing state or enforces rules, it may be an entity
- if it is just attached data, make it a field

Output:

- major state carriers only

## 5. Assign responsibilities and ownership

Ask:

- who owns each piece of mutable state?
- who is allowed to perform each transition?
- where should each rule live?

This is the highest-value structural step.

Output:

- responsibility split
- mutation ownership
- orchestration boundary

## 6. Define interfaces only where variation exists

Ask:

- what is likely to vary?
- do I need a contract here, or just a concrete class?

Typical variation points:

- pricing
- allocation strategy
- notification channel
- payment provider

Output:

- minimal abstractions

## 7. Choose data structures

Ask:

- what operations dominate?
- what lookup, ordering, or eviction behavior matters?

Output:

- DS choices justified by operations

## 8. Add concurrency and failure handling

Ask:

- what races are possible?
- what must be atomic?
- what invalid interleavings break invariants?

Output:

- concurrency boundaries
- locking or serialization points if needed
- failure assumptions

## 9. Discuss extensibility

Ask:

- what requirement is most likely to change next?
- where would that change land?

Output:

- one or two concrete future changes
- explanation of why the current structure absorbs them

## 10. Trace flows

Always walk:

- one happy path
- one failure path

This validates the design under motion instead of only in snapshot form.

---

# 3. What To Avoid

- jumping into classes before scope is clear
- creating entities for every noun
- using interfaces without a real variation point
- discussing patterns before the underlying problem exists
- choosing data structures before operations are known
- writing code before ownership and transitions are stable

---

# 4. Short Interview Script

Use this mental script:

1. What exactly are we building?
2. What must never break?
3. What states and transitions exist?
4. What are the major entities?
5. Who owns mutation and rules?
6. Where do I actually need abstraction?
7. What operations drive DS and concurrency?
8. How does one happy path and one failure path run?
9. What future change should this design survive?
