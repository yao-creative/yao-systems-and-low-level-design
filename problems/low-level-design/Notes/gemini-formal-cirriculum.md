To build a working knowledge of formal methods for software design, you need to sequence your learning so that foundational math supports the higher-level abstractions. Jumping straight into verification without understanding basic logic will cause unnecessary frustration.

Here is a structured, four-phase curriculum isolating the exact concepts you need to learn, in the order you should learn them.

1. **First-Order Logic and Set Theory:** The prerequisite math for all formal modeling.
Before you can formally define a system, you need the vocabulary to express absolute truths and constraints.

* **Propositional Logic:** Truth tables, boolean satisfiability, and logical operators (AND, OR, NOT, IMPLIES).
* **First-Order (Predicate) Logic:** Universal quantifiers ("for all") and existential quantifiers ("there exists"). You need this to state rules like "For all users, if a user is active, they must have a valid email."
* **Set Operations:** Unions, intersections, and subsets.
* **Relations and Functions:** Understanding injectivity, surjectivity, and bijectivity. This is how you formally model database schemas and API mappings.


2. **Automata Theory:** Modeling behavior and lifecycles.
Once you can describe static constraints, you need to describe how systems change state over time.

* **Deterministic Finite Automata (DFA):** The mathematical definition of a state machine (states, alphabet, transition function, start state, accept states).
* **Statecharts (Harel Statecharts):** Hierarchical states, orthogonal (parallel) regions, and history states. This translates pure DFAs into usable software components.
* **Mealy and Moore Machines:** Understanding the difference between outputs determined by state alone versus outputs determined by state *and* input.


3. **Practical Type Theory:** Modeling data and interfaces.
This bridges the gap between your logic foundations and actual code, allowing the compiler to enforce your design.

* **Algebraic Data Types (ADTs):** Mastering Product Types (structs/records—"A *and* B") and Sum Types (tagged unions/enums—"A *or* B").
* **Exhaustive Pattern Matching:** Using the compiler to prove that all possible states of a Sum Type have been handled.
* **Parsing vs. Validating:** The concept of converting weak types (like strings) into strong ADTs at the system boundary so internal business logic never has to deal with invalid data.
* **Curry-Howard Correspondence:** The conceptual realization that a type is a mathematical proposition, and a program is the proof of that proposition.


4. **Axiomatic Semantics:** Proving algorithmic correctness.
The final phase is applying formal proofs to individual functions, memory, and algorithms.

* **Hoare Triples:** Writing contracts using preconditions, execution commands, and postconditions (e.g., `{P} C {Q}`).
* **Loop Invariants:** Defining properties that remain true before, during, and after a loop executes. This is critical for proving that an algorithm actually terminates and produces the right result.
* **Design by Contract:** Applying Hoare logic in practice by writing explicit `assert`, `require`, and `ensure` statements in your low-level functions.