Your intent is **control-flow architecture**: understanding why systems introduce an explicit routing mechanism instead of encoding control flow directly as function composition.

The key distinction is:

> **Function chaining answers “what happens next?”**
> **Dispatch answers “which computation should happen next?”**

## 1. Function chaining has a fixed control-flow graph

Suppose you write:

$$
f \rightarrow g \rightarrow h
$$

Then the control flow is already encoded in the program:

$$
x \xrightarrow{f} y \xrightarrow{g} z \xrightarrow{h} w
$$

In code:

```rust
let x = f(input);
let y = g(x);
let z = h(y);
```

This is excellent when the computation really is a fixed pipeline.

But imagine an HTTP server.

You don't know beforehand whether the request is:

$$
/users
$$

or

$$
/orders
$$

or

$$
/payments
$$

So you need:

$$
\text{request} \rightarrow ?
$$

The dispatcher introduces a **runtime routing function**:

$$
D : I \rightarrow H
$$

and therefore:

$$
D(i)(i)
$$

The control flow becomes **data-dependent**.

---

# 2. A dispatcher turns control flow into data

This is the deepest reason dispatchers exist.

Without dispatch:

$$
\text{program structure}
=
\text{control flow}
$$

With dispatch:

$$
\text{program structure}
+
\text{routing data}
=
\text{control flow}
$$

For example:

$$
R =
\{
\texttt{GET /users} \mapsto h_1,
\texttt{GET /orders} \mapsto h_2,
\texttt{POST /users} \mapsto h_3
\}
$$

Then:

$$
D(request)=R(\operatorname{key}(request))
$$

This means you can change routing **without rewriting the entire control-flow graph**.

That's enormously valuable in extensible systems.

---

# 3. It separates selection from execution

Without a dispatcher, you tend to write:

```rust
if request.path == "/users" {
    get_users(request)
} else if request.path == "/orders" {
    get_orders(request)
} else if request.path == "/payments" {
    get_payments(request)
}
```

The function doing routing now knows:

* what requests exist
* how they're identified
* which handler implements each request
* how to select them.

A dispatcher extracts this responsibility:

$$
\boxed{
\text{Selection}
\neq
\text{Execution}
}
$$

So:

$$
\text{Dispatcher}
:
I \rightarrow H
$$

while:

$$
H
:
I \rightarrow O
$$

The dispatcher determines **who** acts.

The handler determines **what** happens.

---

# 4. This becomes extremely important at scale

Imagine 10,000 handlers.

A giant chain:

$$
\text{if }i=i_1 \rightarrow h_1
$$

$$
\text{else if }i=i_2 \rightarrow h_2
$$

$$
\ldots
$$

is basically a huge centralized control-flow structure.

Instead you can construct:

$$
R : I \rightarrow H
$$

and perform lookup.

For exact keys:

$$
R : I \rightarrow H
$$

might be implemented as a hash map.

For HTTP paths, it might be a trie/radix tree.

For pattern matching, it might be a decision tree.

For commands, it might be a registry.

The **mathematical routing abstraction stays the same while the implementation changes.**

---

# 5. Dispatchers enable dynamic extensibility

This is one of their biggest benefits.

Suppose your system has:

$$
R_0 =
\{
A \mapsto h_A,
B \mapsto h_B
\}
$$

A plugin can extend it:

$$
R_1 =
R_0
\cup
\{C \mapsto h_C\}
$$

You don't need to modify the central execution pipeline.

This is why registries are common in:

* plugin systems
* RPC frameworks
* CLI frameworks
* web frameworks
* event systems
* dependency injection
* serialization
* database drivers

The dispatcher becomes a **stable extension point**.

---

# 6. Dispatchers enable indirection

Suppose:

$$
\texttt{GenerateReport}
\rightarrow
\texttt{ReportHandler}
$$

Today:

$$
D(\texttt{GenerateReport}) = H_1
$$

Tomorrow:

$$
D(\texttt{GenerateReport}) = H_2
$$

The caller doesn't care.

That lets you change:

* implementation
* location
* worker
* version
* resource pool
* execution policy

without changing the producer.

In distributed systems this becomes:

$$
\text{logical operation}
\rightarrow
\text{routing}
\rightarrow
\text{physical execution}
$$

That indirection is extremely powerful.

---

# 7. Dispatch can centralize cross-cutting policy

This is where industrial dispatchers become much more than lookup tables.

Suppose:

$$
D(i)=h
$$

The dispatcher can implement:

$$
\text{authenticate}
\rightarrow
\text{authorize}
\rightarrow
\text{rate limit}
\rightarrow
\text{trace}
\rightarrow
\text{select handler}
\rightarrow
\text{execute}
\rightarrow
\text{metrics}
$$

So instead of every handler implementing these independently:

$$
h_1,h_2,\ldots,h_n
$$

the runtime establishes common policy around them.

This gives you:

$$
\boxed{
\text{centralized mechanism}
+
\text{decentralized behavior}
}
$$

That's one of the most important architectural patterns here.

---

# 8. Dispatch also enables different routing policies

Suppose:

$$
I=\{\text{tasks}\}
$$

You could dispatch according to:

$$
D_1(i)=\text{round-robin worker}
$$

or:

$$
D_2(i)=\text{least-loaded worker}
$$

or:

$$
D_3(i)=\text{highest-priority worker}
$$

or:

$$
D_4(i)=\text{worker owning resource }r
$$

The producer still says:

> "Here is task $i$."

It doesn't need to know how execution is allocated.

This is where dispatch begins merging conceptually with **scheduling and resource arbitration**.

---

# 9. But don't overuse dispatchers

This is equally important.

If you have:

$$
f \rightarrow g \rightarrow h
$$

and the sequence is intrinsically fixed, introducing:

$$
D(f)=g
$$

is usually pointless abstraction.

You've transformed:

```text
f → g
```

into:

```text
f → dispatcher → lookup → g
```

without solving a real problem.

So:

### Fixed computation

Use composition:

$$
f \circ g \circ h
$$

### Dynamic selection

Use dispatch:

$$
D : I \rightarrow H
$$

### Many consumers

Use pub/sub:

$$
D : I \rightarrow \mathcal{P}(H)
$$

### Resource-dependent selection

Use scheduling:

$$
D : (T,S) \rightarrow T
$$

where $S$ is system state.

---

# 10. This gives you a useful hierarchy

You can think of increasingly dynamic control flow as:

$$
\boxed{
\text{Composition}
\;<\;
\text{Dispatch}
\;<\;
\text{Scheduling}
}
$$

**Composition:**

> I already know what happens next.

$$
f \rightarrow g
$$

**Dispatch:**

> I know the input, but I need to determine which computation handles it.

$$
i \rightarrow D(i) \rightarrow h
$$

**Scheduling:**

> I know the set of runnable computations, but I need to determine which one gets resources now.

$$
(T,S) \rightarrow t
$$

That is why the OS analogy is so useful.

A trap handler is not merely "the next function." The kernel receives an **externally determined stimulus**, maps it to the appropriate handler, and then executes it.

Likewise, a large application receives requests, commands, events, and jobs whose destinations aren't encoded as a single fixed function chain.

**Dispatchers solve the problem of making control flow a runtime-selectable relation rather than a statically fixed sequence.**
