Below is a **formal FP / category-theoretic encoding of each pattern as LLD primitives**. No prose—just structure.

---

# 0. Base formal system (shared)

Let:

* ( $\mathcal{S}$ ) = set of states
* ( $\mathcal{T} \subseteq \mathcal{S} \times \mathcal{S}$ ) = valid transitions
* ( $\mathcal{M} : \mathcal{S} \to \mathcal{S}$ ) or ( $\mathcal{S} \to \mathcal{S}'$ ) = state transformer

Category view:

* Objects: ( $S \in \mathcal{S}$ )
* Morphisms: ( $f: S_i \to S_j \in \mathcal{T}$ )

Encapsulation = restriction:

$$
\mathcal{T}_{exposed} \subseteq \mathcal{T}
$$

---

# 1. Abstract Data Type (Encapsulated Object)

### Signature

$$
ADT := (\Sigma, C, O)
$$

* ( $\Sigma$ ): hidden representation
* ( $C: \alpha \to \Sigma$ ): constructors
* ( $O: \Sigma \to \beta$ ): observers

### Constraint

$$
\forall s \in \Sigma,\quad s \notin \mathsf{Expose}
$$

### Morphism form

$$
O_i \circ C_j : \alpha_j \to \beta_i
$$

### Encapsulation invariant

$$
\neg \exists f : \Sigma \to \Sigma \text{ exposed}
$$

---

# 2. Module / Opaque Type System

### Definition

$$
Module := (\Sigma_{hidden}, {f_k}_{exposed})
$$

Visibility function:

$$
vis : \Sigma \to {0,1}, \quad vis(\Sigma_{hidden}) = 0
$$

### Access constraint

$$
f_k \in \mathsf{Hom}(\Sigma_{hidden}, \Sigma_{hidden} \cup \beta)
$$

but:

$$
\forall g \notin Module,\quad g \not\subseteq \mathsf{Hom}(\Sigma_{hidden}, -)
$$

---

# 3. Strategy Pattern (Function Selection Encapsulation)

### Type

$$
Strategy := A \to B
$$

### Family of strategies

$$
\mathcal{F} = { f_i : A \to B }_{i \in I}
$$

### Selector

$$
select : C \to \mathcal{F}
$$

### Execution

$$
exec(c,a) = select(c)(a)
$$

### Encapsulation constraint

$$
\mathcal{F} \text{ hidden}, \quad select \text{ exposed}
$$

---

# 4. Command Pattern

### Type

$$
Cmd := \Sigma \to \Sigma
$$

or parameterized:

$$
Cmd_x : \Sigma \to \Sigma
$$

### Command set

$$
\mathcal{C} = { Cmd_i }
$$

### Execution monoid

$$
(Cmd, \circ, id)
$$

### Composition

$$
Cmd_j \circ Cmd_i : \Sigma \to \Sigma
$$

### Encapsulation constraint

$$
\mathcal{C}_{exposed} \subseteq \mathcal{C}
$$

---

# 5. State Pattern (Algebraic State Machine)

### State space

$$
\Sigma = \bigsqcup_{i \in I} S_i
$$

(sum type decomposition)

### Transition function

$$
\delta : \Sigma \to \Sigma
$$

refined:

$$
\delta_{ij} : S_i \to S_j
$$

### Valid transition graph

$$
G = (I, E), \quad E \subseteq I \times I
$$

### Constraint

$$
\delta_{ij} = \varnothing \quad \text{if } (i,j) \notin E
$$

---

# 6. Factory Pattern (Controlled Construction)

### Unsafe constructor

$$
raw : \alpha \to \Sigma
$$

### Validating constructor

$$
safe : \alpha \to \Sigma \cup {\bot}
$$

or

$$
safe : \alpha \to \mathsf{Either}(\mathsf{Error}, \Sigma)
$$

### Constraint predicate

$$
P : \alpha \to {0,1}
$$

$$
safe(x) =
\begin{cases}
raw(x) & P(x)=1 \
\bot & P(x)=0
\end{cases}
$$

---

# 7. Observer Pattern (Event Coalgebra)

### Event space

$$
E
$$

### System state

$$
\Sigma
$$

### Transition with emission

$$
f : \Sigma \to (\Sigma \times E)
$$

### Observer set

$$
Obs = { o_i : E \to U_i }
$$

### Lifted execution

$$
o_i(f(s)) = o_i(e)
$$

### Encapsulation constraint

$$
Obs \not\subset \Sigma,\quad Obs \text{ external coalgebra}
$$

---

# 8. Unified encapsulation principle (all patterns)

### General form

Encapsulation = restriction of accessible morphisms:

$$
Encap = (\mathcal{S}, \mathcal{T}_{allowed} \subseteq \mathcal{S}\times \mathcal{S})
$$

or functionally:

$$
Encap = (\Sigma, \mathsf{Hom}_{exposed}(\Sigma,\Sigma))
$$

### Enforcement operator

$$
\Pi : \mathsf{AllMorphisms} \to \mathsf{AllowedMorphisms}
$$

---

If you want next step, I can compress this into a **single unified algebra (FP-LDD calculus)** where all patterns become just different factorizations of:

$$
\Sigma \xrightarrow{;f;} \Sigma \times E
$$

or map it directly to elevator / scheduling systems as a full categorical state machine.
