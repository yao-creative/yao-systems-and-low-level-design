# LLD Interview Workflow

This file owns the **practical interview sequence**.
The conceptual explanation for why this sequence works lives in `lld-why.md`.

---

# 1. Compressed Workflow

```text
Requirements
    ↓
Core Entities
    ↓
Relationships
    ↓
API / Interfaces
    ↓
State Machine
    ↓
Responsibilities / Ownership
    ↓
Happy Path + Failure Path
    ↓
Patterns
    ↓
Concurrency Notes
    ↓
Extensibility
```

```
RISERIDCE
```

---

# 2. Fast Interview Order

Use this order in a live interview.

## 1. Clarify requirements

Ask for:

- high-level design constraints
- functional requirements
- core capabilities
- assumptions
- scope boundaries
- edge cases

Output:

- a short agreed spec
- functional scope and constraints from the HLD view

## 2. Identify core entities

Ask:

- what are the major state carriers?
- what should remain a field instead of becoming its own entity?

Output:

- major entities only
- one-sentence reason each

## 3. Model relationships

Ask:

- how do entities relate?
- where is composition, aggregation, association, or dependency?

Output:

- relationship map or class diagram notes

## 4. Define APIs and interfaces

Ask:

- what APIs does this design expose?
- where do we need an interface instead of a concrete class?
- what is the request / response contract?
- what are the preconditions and postconditions?

Output:

- API surface
- interface boundaries
- contract notes

## 5. Draw the state machine

Ask:

- what states exist?
- what transitions are legal?
- what transitions are illegal?

Output:

- lifecycle states
- allowed transitions
- terminal or failure states if relevant

## 6. Assign responsibilities and ownership

Ask:

- who owns each piece of mutable state?
- who is allowed to perform each transition?
- where should each rule live?

Output:

- responsibility split
- mutation ownership
- enforcement point

## 7. Trace happy path and failure path

Ask:

- how does the main use case run?
- how does one failure path break safely?

Output:

- end-to-end trace of one success path
- end-to-end trace of one failure path

## 8. Apply design patterns

Ask:

- what is likely to vary?
- do I need a contract here, or just a concrete class?

Output:

- minimal abstractions
- patterns only where justified by variation

## 9. Concurrency notes

Ask:

- what operations dominate?
- what shared state can race?
- what must be atomic?

Output:

- DS choices justified by operations
- race risks and control points

## 10. Discuss extensibility

Ask:

- what requirement is most likely to change next?
- where would that change land?

Output:

- one or two concrete future changes
- explanation of why the current structure absorbs them

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
