Good instinct — let's isolate which relations across this conversation are actually **posets** (reflexive, antisymmetric, transitive), since not everything I called "ordered" technically qualifies. I'll define each formally, then draw its Hasse diagram.

## 1. Lifecycle phase order — a total order (chain)

$$
P_1 = (\{\text{Uninitialized}, \text{Ready}, \text{AfterWindowOpen}, \text{Eventually}\},\; \leq)
$$

Total order: any two phases are comparable. `run(t)` is only permitted when $\text{phase}_{\text{current}} \geq \text{phase}_{\text{required}}(t)$ — this poset gates *when* code is allowed to execute.A note on scope, since it matters for the labeling: the first two (lifecycle phase, trust chain) are **total orders** — every pair is comparable. The third (DI inclusion) is a **two-element sub-poset** of the Boolean lattice $\mathcal{P}(S)$ under $\subseteq$; it only has content because registration domains are disjoint, so $S_0 \subseteq S_0 \cup S_1$ strictly. The fourth is the genuine **product order** $\text{Phase} \times \text{Idle}$ — this is the only one that isn't a chain: $(\text{Ready}, \text{idle})$ and $(\text{Eventually}, \text{busy})$ are *incomparable*, which is exactly why `runWhenGlobalIdle` needs both coordinates before it fires — phase alone doesn't decide it.

One thing worth flagging: **leader election is not a poset at all.** "Holds the lock" vs. "doesn't" is a two-valued exclusion predicate, not an order relation — there's no transitive chain of who's "more the leader" than whom. I mentioned it loosely as gating logic earlier, but it doesn't belong on this list; good that you're pressing on which relations are actually order-theoretic rather than just "sequential-sounding."