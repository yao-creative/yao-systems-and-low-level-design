---
name: lld-session-critique
description: Critique low-level design practice notes or session markdown files by reading the full attempt, evaluating the learner's reasoning across the LLD funnel, compressing strengths and weaknesses into a concise matrix, and appending the critique back into the session note by default. Use when Codex is asked to review an LLD attempt, summarize thinking patterns, compare strengths vs weaknesses, or write a compact critique section into an `attempt.md`, `session.md`, or similar markdown artifact.
---

# LLD Session Critique

Read the full practice note before judging it. Focus on the learner's reasoning quality, not on rewriting the whole design.

## Workflow

1. Read the entire session note end to end.
2. Infer the learner's progression through the fixed LLD funnel:
   1. requirements
   2. entities
   3. relationships
   4. behavior
   5. invariants
   6. extensibility
   7. execution tracing
3. Critique the thinking, not just the final answer.
4. Compress the critique into a matrix with columns:
   1. `Dimension`
   2. `Strength`
   3. `Weakness`
5. Add a short closing assessment with 2-4 lines on the learner's strongest pattern, main failure mode, and next improvement target.
6. If the user points to a markdown note or asks to update the note, append the critique to the end of that file.
7. If the user asks for chat-only output, do not edit the file.

## Evaluation Lens

Judge patterns such as:

1. ambiguity reduction
2. entity discovery
3. ownership reasoning
4. abstraction discipline
5. session or identity modeling
6. formal reasoning quality
7. invariant thinking
8. extensibility thinking
9. learning behavior under critique

Do not force every dimension into every critique. Use only the dimensions that are visible in the note.

## Compression Rules

Keep the matrix compact. Prefer one strong sentence per cell.

Do not turn the output into a full tutorial. The point is to preserve signal:

1. what the learner consistently does well
2. where their reasoning slips categories
3. what improvement would raise their level fastest

Prefer patterns like:

- "detects the right pressure but phrases it one level too operationally"
- "good aggregate instinct, weak invariant formulation"
- "adapts quickly once the boundary is clarified"

## Editing Rules

When appending to the note:

1. preserve all existing content
2. append at the end under a short heading such as `## Critique Matrix`
3. follow with `## Compressed Assessment`
4. keep the addition compact enough that the note remains usable as practice material

## Output Style

Use a rigorous reviewer voice. Keep it concise, technical, and specific.

Prefer:

1. short paragraphs
2. a single flat markdown table
3. a brief closing assessment

Avoid:

1. full redesigns
2. class-by-class rewrites
3. generic praise without evidence
