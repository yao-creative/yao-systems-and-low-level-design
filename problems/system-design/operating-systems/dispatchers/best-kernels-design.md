## Causal funnel: why dispatch tables exist at all

Your terminal design goal, precisely labeled, is **amortized O(1) polymorphic resolution across a protection-domain boundary** — i.e., you need to go from an untrusted numeric/opaque selector to a trusted code pointer without (a) branching cost, (b) recompiling the caller when the callee changes, or (c) leaking implementation identity across the trust boundary. Everything below is that goal decomposed:

```mermaid
graph TD
A[Goal: resolve opcode -> implementation across a trust boundary, O(1), extensibly]
A --> B[Subgoal: eliminate call-site/impl coupling]
A --> C[Subgoal: bound resolution cost, independent of |Op|]
A --> D[Subgoal: make the resolution surface auditable/attenuable]
B --> E[Mechanism: late binding via indirection]
C --> F[Mechanism: indexed array, not chained conditionals]
D --> G[Mechanism: capability-typed / unforgeable selectors]
E --> H[Linux: file_operations, net_proto_ops]
F --> I[Linux: sys_call_table]
G --> J[seL4: capability invocation dispatch]
```

## Set-theoretic formalization

Let $\mathrm{Op}$ be the finite carrier set of opcodes (syscall numbers, IPC labels, VFS method tags). Let

$$
\mathrm{Sig} \;=\; \mathrm{State} \times \mathrm{Args} \;\rightharpoonup\; \mathrm{State}' \times \mathrm{Result}
$$

be the (partial, since faults are terminal in‑domain elements) type of a kernel transition function. A dispatch table is simply an element of the dependent product

$$
D \;\in\; \prod_{o \,\in\, \mathrm{Op}} \mathrm{Sig}
$$

i.e. $D : \mathrm{Op} \to \mathrm{Sig}$, a *total* function over a *finite index set*, which is exactly what licenses array indexing instead of comparison chains — totality over a finite discrete domain is the set-theoretic precondition for $O(1)$ lookup.

Dually, opcode selection is the universal map out of a coproduct: with injections $\iota_o : \{*\} \to \mathrm{Op}$,

$$
\mathrm{Op} \;\cong\; \coprod_{o \in \mathrm{Op}} \{*\}, \qquad D = [\,f_o\,]_{o\in \mathrm{Op}} : \coprod_{o} \{*\} \to \mathrm{Sig}
$$

so the dispatch table *is* the case-elimination principle of the sum type — a Church encoding of $\mathrm{Op}$ realized as a jump table instead of nested `match`.

## Category-theoretic formalization

Treat effectful kernel transitions as the Kleisli category $\mathrm{Kl}(\mathcal{E})$ of an effect monad $\mathcal{E}$ (capturing faults, non-determinism from interrupts, or async completion): objects are state-types, morphisms $\mathrm{State} \to \mathcal{E}(\mathrm{State}')$.

- If $\mathrm{Op}$ is treated as a **discrete category** (objects only, identities only), a dispatch table is a functor
$$F : \mathrm{Op} \to \mathrm{Kl}(\mathcal{E})$$
picking one Kleisli morphism per object. This is the *flat* case — Linux's `sys_call_table`: no structure among opcodes, so functoriality is vacuous, which correctly predicts why this layer needs no inheritance/subtyping machinery.

- If instead $\mathrm{Op}$ carries a **preorder** (capability derivation: "read-only" ⊑ "read-write"), the dispatch table becomes a functor from that preorder-as-category into $\mathrm{Kl}(\mathcal{E})$, and *derivation* (attenuating a capability) is required to be a **monomorphism** — composition can only monotonically restrict the reachable morphism set, never enlarge it. This is exactly seL4's capability model: security isn't bolted on, it's the functor being forced to respect the preorder's arrows.

- VFS-style `file_operations` structs are **records of morphisms**, i.e. an object of the product category $\prod_{m \in \mathrm{Methods}} \mathrm{Sig}_m$ — structural typing standing in for a class hierarchy, with no shared supertype required: this is *ad hoc polymorphism by explicit product*, not subtype polymorphism.

## Survey of best-designed kernels, by which node of the funnel they optimize

| Kernel | Dispatch mechanism | Which subgoal dominates |
|---|---|---|
| Linux | flat `sys_call_table[nr]`; per-object `file_operations`/`net_proto_ops` structs | (F) raw index speed at syscall layer, (E) late binding at VFS layer |
| seL4 | capability invocation label → method inside the capability's object type | (G) auditable attenuation; dispatch *is* the access-control check |
| Barrelfish (multikernel) | per-core message-type → handler table, no shared mutable dispatch state | (F)+(E) but under a no-shared-memory constraint, so the table itself is Kleisli over an async/message monad |
| Redox (Rust microkernel) | scheme (opcode) → handler via trait objects behind `Arc<dyn Scheme>` | (E), traded some (F) raw speed for Rust-level safety of the indirection |

The general principle your funnel predicts: **the more Op needs internal structure (ordering, rights, types), the more the dispatch table degenerates from a flat array (F) into a functor over a nontrivial category (G)** — that's the real design axis, not "array vs switch."

## Rust: dispatch table as pure function value, never owning state

Per your constraint — the table and its handlers borrow or mutate `KernelState`, never own it, and nothing is initialized with embedded state:

```rust
type Args = [usize; 6];
type Errno = u32;
type SysFn = fn(&mut KernelState, Args) -> Result<usize, Errno>;

// A `const` array of bare fn pointers holds no state at all —
// it is the reification of D : Op -> Sig, nothing more.
const SYSCALL_TABLE: [SysFn; 4] = [
    sys_read,
    sys_write,
    sys_open,
    sys_close,
];

fn dispatch(state: &mut KernelState, nr: usize, args: Args) -> Result<usize, Errno> {
    SYSCALL_TABLE
        .get(nr)
        .ok_or(ENOSYS)
        .and_then(|f| f(state, args))
}

fn sys_read(state: &mut KernelState, args: Args) -> Result<usize, Errno> {
    let fd = state.fd_table.get_mut(args[0])?; // mutate borrowed state, never own it
    fd.read(args[1] as *mut u8, args[2])
}
```

`SYSCALL_TABLE` is a compile-time constant of function *values* (the $D \in \mathrm{Sig}^{\mathrm{Op}}$ element itself) — there is no struct instantiated with a state field, `dispatch` never owns `KernelState`, it only ever borrows it mutably for the duration of the call, matching the Kleisli-morphism reading above (`State ⊸ Result<State', E>` realized as `&mut State -> Result<usize, Errno>` under Rust's borrow discipline standing in for the monadic threading).