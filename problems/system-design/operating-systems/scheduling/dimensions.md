Your intent is **OS scheduling-policy decomposition**: you’re asking how a kernel selects a scheduling policy, and what the fundamental dimensions of that decision are.

The key idea is:

> A kernel generally does **not choose one scheduling tactic once and for all**. It maintains a scheduling policy whose decision function evaluates the current runnable set and system state whenever a scheduling decision is needed.

## 1. The basic abstraction

At a high level, suppose the runnable tasks are a set

$$
R = \{p_1,p_2,\ldots,p_n\}.
$$

The scheduler computes something like

$$
S : (R,\,\sigma,\,e) \to (p,c)
$$

where:

* $R$ = runnable tasks
* $\sigma$ = scheduler/system state
* $e$ = event triggering reconsideration
* $p$ = task selected
* $c$ = CPU/core on which it should execute

So scheduling actually contains **two coupled decisions**:

$$
\text{Scheduling}
=
\text{Task selection}
+
\text{CPU placement}.
$$

This distinction becomes very important on multicore systems.

---

# 2. What makes the kernel choose a particular tactic?

Usually the kernel doesn't reason:

> "This workload looks like X, therefore I'll switch to algorithm Y."

Instead, the policy is encoded into the scheduler.

For example, a scheduler may define a priority/ordering function

$$
f(p,\sigma)
$$

and select

$$
p^* = \operatorname{argmax}_{p\in R} f(p,\sigma).
$$

The interesting question becomes:

**What dimensions does $f$ depend on?**

There are several major ones.

---

## 3. Dimension 1 — Priority

The simplest dimension is:

$$
\text{Who should run first?}
$$

Tasks may have:

* static priority
* dynamic priority
* real-time priority
* user/process priority
* deadline
* niceness
* interactive priority

A simple priority scheduler might have

$$
f(p)=\operatorname{priority}(p).
$$

Then:

$$
p^*=\arg\max_{p\in R}\operatorname{priority}(p).
$$

But priority alone creates starvation.

So real schedulers generally add other dimensions.

---

# 4. Dimension 2 — Time fairness

Another fundamental dimension is:

$$
\text{How much CPU time has this task already received?}
$$

Suppose two tasks have equal priority:

$$
p_1,p_2.
$$

If

$$
T(p_1) \gg T(p_2),
$$

the scheduler may favor $p_2$.

This produces **fairness**.

A scheduler can therefore think in terms of a virtual accumulated CPU time:

$$
V(p).
$$

Then a rough policy might be:

$$
p^*=\arg\min_{p\in R} V(p).
$$

This is the conceptual foundation behind fair-share scheduling.

---

# 5. Dimension 3 — Responsiveness

Interactive workloads introduce another dimension:

$$
\text{How urgently does this task need CPU time?}
$$

For example:

* a GUI event handler
* audio processing
* keyboard input
* network packet processing
* an interactive shell

may need CPU immediately even if another process has been running.

So the scheduler considers **latency**:

$$
L(p)=\text{time since }p\text{ became runnable}.
$$

A policy can therefore reward waiting:

$$
f(p)=\alpha P(p)+\beta L(p)-\gamma T(p).
$$

This is one reason scheduling isn't simply FIFO or priority ordering.

---

# 6. Dimension 4 — Deadlines

Real-time systems add an entirely different objective:

$$
\text{Will this task meet its deadline?}
$$

For task $p$:

$$
D(p)=\text{deadline}.
$$

A deadline scheduler may prioritize the smallest deadline:

$$
p^*=\arg\min_{p\in R}D(p).
$$

This gives **Earliest Deadline First (EDF)**.

Notice how different this objective is from fairness.

A normal desktop system might care about:

$$
\text{fairness}+\text{responsiveness}.
$$

A real-time system might care about:

$$
\text{deadline satisfaction}.
$$

Hence there isn't one universally optimal scheduler.

---

# 7. Dimension 5 — CPU affinity / topology

Once you have multiple cores, task selection isn't enough.

Suppose:

$$
C=\{c_1,c_2,\ldots,c_m\}.
$$

Now the scheduler needs:

$$
(p^*,c^*)
$$

rather than merely $p^*$.

The question becomes:

> **Where should this task execute?**

The scheduler may consider:

* CPU affinity
* NUMA locality
* cache locality
* CPU utilization
* SMT sibling relationships
* energy efficiency
* heterogeneous cores

For example:

$$
\operatorname{cost}(p,c)
=
\text{migration cost}
+
\text{cache cost}
+
\text{NUMA cost}.
$$

Then placement can seek

$$
c^*=\arg\min_{c\in C}\operatorname{cost}(p^*,c).
$$

This is why multicore scheduling is substantially harder than single-core scheduling.

---

# 8. Dimension 6 — Preemption

Another fundamental policy dimension is:

$$
\text{When am I allowed to interrupt the current task?}
$$

Two broad choices:

### Cooperative

The running task continues until it yields or blocks.

$$
p_{\text{current}}\rightarrow p_{\text{next}}
$$

only at explicit scheduling points.

### Preemptive

The kernel can interrupt it.

For example:

$$
\text{timer interrupt}
\rightarrow
\text{scheduler}
\rightarrow
p_{\text{new}}.
$$

Modern general-purpose OSes predominantly use preemptive scheduling because it provides much better control over latency and fairness.

---

# 9. Dimension 7 — Work conservation

Another important property:

> If there is runnable work and an available CPU, should the CPU run it?

A **work-conserving** scheduler tries to maintain:

$$
R\neq\varnothing
\implies
\text{CPU executes some }p\in R.
$$

This interacts with load balancing.

If:

$$
|R_{c_1}| \gg |R_{c_2}|,
$$

the scheduler may migrate work from $c_1$ to $c_2$.

But migration itself has a cost.

Hence:

$$
\text{load balancing benefit}
>
\text{migration cost}
$$

is roughly the condition under which migration makes sense.

---

# 10. Dimension 8 — Resource constraints

Scheduling can also incorporate constraints such as:

* CPU quotas
* cgroups
* containers
* user limits
* power budgets
* thermal throttling
* CPU affinity masks

So a task isn't simply:

$$
p\in R.
$$

It may have a feasible CPU set:

$$
A(p)\subseteq C.
$$

The scheduler must choose:

$$
c\in A(p).
$$

This gives you a useful mathematical decomposition:

$$
\boxed{
\text{Scheduling}
=
\text{Eligibility}
+
\text{Ordering}
+
\text{Placement}
+
\text{Preemption}
}
$$

---

# 11. So how does Linux actually work?

Linux is a useful concrete example.

Linux doesn't have one universal "scheduler algorithm." It has **scheduling classes**.

Conceptually:

$$
\text{Runnable tasks}
\rightarrow
\text{scheduling classes}
\rightarrow
\text{highest applicable class}
\rightarrow
\text{task selection}.
$$

Historically and conceptually you can think of classes such as:

* real-time scheduling
* deadline scheduling
* normal/fair scheduling
* idle scheduling

The important architectural point is that **the scheduler first establishes policy class, then the class determines how candidates are ordered**.

So the hierarchy is approximately:

$$
\text{Global scheduling policy}
\supset
\text{Scheduling class}
\supset
\text{Runnable queue}
\supset
\text{Selection rule}.
$$

This is much closer to how you should think about kernel scheduling than "the kernel picks an algorithm."

---

# 12. The deepest decomposition

If you're trying to model this as a state machine, I'd use these dimensions:

$$
\boxed{
\begin{aligned}
\text{Eligibility} &: \quad \text{Can }p\text{ run on }c?\\
\text{Priority} &: \quad \text{Which runnable }p\text{ dominates?}\\
\text{Fairness} &: \quad \text{Who has received insufficient service?}\\
\text{Latency} &: \quad \text{Who needs service soon?}\\
\text{Deadline} &: \quad \text{Who has a temporal constraint?}\\
\text{Affinity} &: \quad \text{Where should }p\text{ run?}\\
\text{Preemption} &: \quad \text{Should current execution be interrupted?}\\
\text{Load balancing} &: \quad \text{Should work move between CPUs?}\\
\text{Resource policy} &: \quad \text{What quotas/budgets constrain execution?}
\end{aligned}
}
$$

And these aren't independent.

For example:

$$
\text{priority}
\rightarrow
\text{selection}
\rightarrow
\text{CPU placement}
\rightarrow
\text{migration}
\rightarrow
\text{cache locality}
\rightarrow
\text{future selection}.
$$

So scheduler state is **feedback-controlled** rather than a simple static ordering.

That is probably the most useful conceptual connection to your recent state-machine work: **the scheduling policy is a transition function over a dynamically changing runnable-state space**, not merely a data structure containing a queue.
