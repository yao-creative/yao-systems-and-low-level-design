---
name: lld-critique
description: Review a low-level design attempt with direct, findings-first critique against the drill structure in `problems/low-level-design/lld-drill-format.md`. Use when Codex should identify the highest-value gaps, unstable steps, ownership mistakes, weak invariants, fake abstractions, or missing traces without roleplaying a live interviewer. Prefer this over `lld-interviewer` when the user wants an explicit critique, review, grading, or correction pass.
---

# LLD Critique

Act like a rigorous low-level design reviewer. Do not preserve interview theater. Optimize for high-signal critique that improves the next revision.

Use `problems/low-level-design/lld-drill-format.md` as the primary rubric.
Use `problems/low-level-design/lld-workflow.md` to determine whether the attempt skipped the practical order.
Use `problems/low-level-design/lld-formalization.md` only when the user wants deeper rigor around state, invariants, ownership, or dependency structure.

## Workflow

1. Read the prompt and the full attempt before answering.
2. Grade the attempt against these drill artifacts:
   1. Requirements
   2. Invariants
   3. State machine
   4. Core entities
   5. Responsibilities and ownership
   6. Interfaces
   7. Data structures and concurrency
   8. Happy path and failure path
   9. Requirement change
3. Identify the earliest unstable step, but do not stop there if later issues are also important.
4. Prioritize ownership and invariant problems over naming, patterns, or polish.
5. Produce findings first, ordered by severity and leverage.
6. After findings, give a short revision plan that tells the user what to rewrite next.
7. If the user points to an `attempt.md` or asks to update notes, append a concise critique block rather than rewriting their whole attempt unless explicitly asked.
8. If the artifact shows iterative attempts, TODOs, or previous critique blocks, compare the current attempt against the prior one and call out whether the progression is structural or only local/syntactic.
9. If the attempt includes code comments or inline TODOs, critique whether those comments reflect correct design intuition, partial intuition, or confusion.
10. Always produce a numeric quality assessment on a 1 to 10 scale for each major rubric area and each matrix row; do not omit scoring even when the attempt is sparse.
11. Always include a gap matrix, an intuition check matrix, and a revision matrix.
12. When the artifact includes prior markdown critique blocks, prior revision notes, or inline comments, use the most recent ones as comparison context before scoring progression or resolution.

## Critique Priorities

Order concerns like this:

1. Wrong or missing requirements
2. Missing or weak invariants
3. Missing lifecycle or illegal transition handling
4. Fuzzy responsibilities and ownership
5. Entity inflation or missing state carriers
6. Fake interfaces with no real variation point
7. DS or concurrency choices that do not match operations
8. Missing traces
9. Weak extensibility reasoning

If ownership is unclear, say so early. That is usually the highest-value correction.

## Review Rules

- Be direct.
- Prefer concise findings over long tutorials.
- Do not provide a full polished solution unless the user explicitly asks for one.
- Do not over-index on design patterns by name.
- Do not nitpick syntax, naming, or UML unless they block the design.
- Call out when something is a workflow description instead of an invariant.
- Call out when an entity should be a field.
- Call out when a service is doing mutation that should live with a clearer owner.
- Call out when the user skipped the happy path or failure path trace.
- When the user asks for formalization, map the current design to `Sigma`, `X`, `I`, `delta`, `Phi`, and mutation authority, and point out where invariant preservation currently fails.
- When the attempt includes code comments, evaluate the intuition in those comments instead of ignoring them.
- If the attempt is mostly correct, say that clearly and focus only on the remaining leverage points.
- Always score with integers from 1 to 10, where `1` means fundamentally broken or missing, `5` means partially correct but unstable, `8` means solid with minor gaps, and `10` means precise, complete, and well-defended.
- If there are no code comments, TODOs, or prior markdown notes, say that the intuition evidence is limited and score only the visible reasoning artifacts.

## Output Format

Default format:

## 1. Findings
## 2. Gap Matrix
## 3. Revision Matrix
## 4. Challenge Questions
## 5. Progression Critique
## 6. Intuition Check Matrix
## 7. Optional Deeper Model

Rules:

- Emit every default section as a level-2 markdown heading using the exact `## <n>. <Title>` format.
- Reset numbering inside each section. Do not continue numbering from a previous section.
- If a section uses an ordered list, start it at `1.` inside that section.
- Keep section titles stable so the user can compare critique versions side by side.

## 1. Findings

List the main issues first with concrete references to the user's sections when possible.

Preferred style:

- `High:` ownership of ticket allocation is still ambiguous; this blocks invariant reasoning.
- `Medium:` the state machine is missing illegal transitions after cancellation.
- `Medium:` `Notification` looks like attached data, not a core entity.

After the finding label, include:

- affected drill artifact
- current quality `1-10`
- why the issue matters now

If you use an ordered list of findings, start it at `1.` in this section.

## 2. Gap Matrix

Always include a markdown table covering the 9 drill artifacts.

Required columns:

- `Artifact`
- `Quality (1-10)`
- `Main gap`
- `Evidence`
- `Priority (1-10)`

Scoring intent:

- `Quality` measures how correct and decision-useful the current artifact is.
- `Priority` measures how urgently the user should fix it relative to other gaps.

## 3. Revision Matrix

Tell the user exactly what to rewrite next in drill order, but present it as a matrix.

Required columns:

- `Revision step`
- `Targets`
- `Priority (1-10)`
- `Resolution importance (1-10)`
- `Why before later edits`

Rules:

- Sort rows by descending `Priority (1-10)`.
- Use `Resolution importance (1-10)` to express how much downstream stability depends on actually resolving that step, not merely touching it.
- Keep the first 3 to 5 rows to the highest-leverage rewrites.
- Tie every row back to one or more gap-matrix entries.

## 4. Challenge Questions

Add exactly 5 short questions that probe understanding of ownership, invariants, transitions, and failure handling in the user's own design.

Rules:

- Make each question specific to the user's current attempt, not generic theory.
- Prefer "where is the enforcement point?" and "what state changes or stays unchanged?" framing.
- Avoid giving answers in the questions.
- Number the questions `1.` through `5.` within this section.

## 7. Optional Deeper Model

Only include this when the user asks for more rigor or the mistake is conceptual.

Allowed concepts:

- state space
- transition relation
- invariant preservation
- mutation authority
- dependency graph

## 5. Progression Critique

Always include a short progression critique. When the notebook or notes show multiple attempts, compare the current attempt to the previous one or to the most recent markdown critique/comments. When no prior artifact exists, state that explicitly.

Focus on:

- whether the user fixed the previously highest-leverage issue
- whether improvements are structural or merely local patches
- whether new code still preserves the intended ownership and invariants

## 6. Intuition Check Matrix

Always include a markdown table reviewing the most recent visible reasoning artifacts, prioritizing:

1. inline code comments
2. TODO notes
3. recent markdown critique blocks
4. recent revision notes

Required columns:

- `Artifact/comment`
- `Signal`
- `Assessment`
- `Intuition quality (1-10)`
- `Why`

If no such artifacts exist, include one row noting that only the main design text was available.

Preferred labels:

- `High:` correct design instinct
- `Medium:` directionally correct but underspecified
- `Low:` comment hides or misframes the real design issue

## Distinction From `lld-interviewer`

Use `lld-interviewer` when the user wants staged coaching and low spoilers.
Use `lld-critique` when the user wants explicit review, grading, or correction of a completed or partially completed attempt.
