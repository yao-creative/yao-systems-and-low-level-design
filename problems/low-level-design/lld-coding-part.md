You *do* put implementation in, but only after a very specific “commit threshold.” The mistake most people make in LLD interviews is treating implementation as a parallel track instead of a **downstream consequence of stabilized design decisions**.

A precise way to frame it:

> Implementation is not a phase you enter early; it is what you do after the system’s transition structure is no longer under revision pressure.

---

# 1. Where implementation sits in the flow

Think of your LLD process as a funnel with two regimes:

## Regime 1 — Model formation (high uncertainty)

You are still deciding:

* what the objects are
* what the relationships are
* what the invariants are
* what the state transitions look like

At this stage:

* DO NOT write full method bodies
* DO NOT commit to data structure internals prematurely

You are operating in:

> “design space exploration”

---

## Regime 2 — Model stabilization (low uncertainty)

You have now fixed:

* class boundaries
* responsibility allocation
* key invariants
* interaction graph

At this point:

> implementation becomes safe because the abstraction no longer changes frequently

This is where you start coding.

---

# 2. The exact “commit threshold” (important mental pivot)

You switch into implementation when you can answer YES to:

### A. Object completeness

Can I list all core entities without hesitation?

### B. Transition completeness

Can I describe every important state change as a method call?

### C. Ownership clarity

For every state variable, do I know exactly which object owns it?

### D. Interaction stability

Would adding a new feature mostly mean *adding methods*, not restructuring everything?

If all are true:

> you are no longer designing the model — you are instantiating it

---

# 3. What “implementation” actually means in LLD interviews

Implementation is not “write everything fully.”

It is usually 3 layers:

## Layer 1 — Skeleton code (always expected)

You write:

* class definitions
* method signatures
* fields

This shows:

> structural correctness

---

## Layer 2 — Key logic only (selective depth)

You implement:

* critical state transitions
* invariants enforcement
* core algorithms

You skip:

* trivial getters/setters
* obvious boilerplate

This shows:

> correctness under constraints

---

## Layer 3 — Edge-case completion (time permitting)

You fill:

* failure handling
* concurrency guards
* validation rules

This shows:

> robustness awareness

---

# 4. The deeper principle (why this ordering exists)

LLD interviews are implicitly testing:

> whether you can separate *structural correctness* from *procedural completeness*

So the correct causal ordering is:

1. Define state space (what exists)
2. Define transitions (what can happen)
3. Define invariants (what must hold)
4. THEN implement transitions

Because implementation without stable state space leads to:

> local code correctness, global design failure

---

# 5. A useful mental rule

When unsure:

> “If changing this decision would force me to rewrite multiple classes, I am not ready to implement yet.”

---

# 6. Practical compression (what you actually do in interview)

You usually alternate like this:

* Sketch class structure (light code)
* Refine interactions (no full code)
* Lock design
* Implement core methods
* Fill gaps if time remains

---

If you want, I can give you a **real LLD trace (like LRU cache or elevator system)** showing exactly where implementation starts line-by-line, so you can see the transition boundary in practice.
