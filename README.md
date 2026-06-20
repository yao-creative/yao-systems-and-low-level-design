# System and Low-Level Design Practice

## Motivation:

I've learnt LLD design before, however, never really practiced it in a structured way. I realized that my productivity and my usage of them is plateau-ing. Take the analogy of a directed/ dependency graph, if one doesn't regularly revise the fundamentals, there are missing dependencies that some further nodes are unreachable and time is spent wandering around the primatives. 

In this attempt, I want to understand with light applied category theory of design and formalize the state machines and other abstraction concepts within.


## Organization

Use folders plus Markdown as the default format.

Why:
- Design answers are mostly structured thinking, tradeoffs, diagrams, and critique.
- Markdown stays diffable, searchable, and easy to review over time.
- Notebooks are useful only when you need simulation, quick benchmarking, or code experiments.

## Recommended Format

Use this shape:

```text
problems/
  system-design/
    bitly/
      session.md
    dropbox/
      session.md
  low-level-design/
    rate-limiter/
      session.md
templates/
  design-session-template.md
```

## Default Workflow

For each problem:

1. Copy `templates/design-session-template.md` into a new problem folder as `session.md`.
2. Fill `Attempt 1` without looking at references.
3. Pause and do the `Socratic critique` section only with questions, not answers.
4. Revise into `Attempt 2`.
5. Write the `Final critique`.
6. Add `Variants` and `Frontier AI applications`.
7. If needed, add a notebook only for concrete experiments.

## When To Use Notebooks

Use notebooks for:
- feed ranking toy simulations
- rate limiter traffic simulation
- top-k streaming sketches
- crawler frontier scheduling experiments
- storage or sharding back-of-the-envelope calculations

Do not use notebooks as the primary format for design reasoning.

## Solving Standard

Every session should force you to answer:
- What is the API or interface?
- What are the core entities and invariants?
- What is the read path?
- What is the write path?
- What breaks at scale?
- What gets cached, sharded, or precomputed?
- What consistency is actually required?
- What metrics and failure modes matter?
- What tradeoff did I choose and why?

## Critique Standard

End every session with:
- strongest part of the design
- weakest assumption
- hidden bottleneck
- production risk
- testability gap
- simplification I made that would break in reality
- one frontier-AI adaptation
- three follow-up variants

## Suggested Cadence

- `45-60 min`: first-principles solve
- `15-20 min`: Socratic critique
- `15-20 min`: revision
- `10 min`: final critique and variants

This repo is set up to optimize for iteration speed, not polished notes.
