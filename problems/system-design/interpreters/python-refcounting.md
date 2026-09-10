**Intent — runtime/memory-model formalization:** you’re asking whether Python’s object model can be understood as **heap-allocated objects + references + reference counting**, and you want the rigorous abstraction rather than just the implementation slogan.

Yes—with an important qualification: **in CPython, essentially every ordinary Python object is heap-allocated**, but “everything is on the heap” is not literally a statement about every piece of runtime data or every Python implementation.

## 1. The core mathematical model

Start with two sets:

$$
O = \{\text{Python objects}\}
$$

$$
R = \{\text{references to objects}\}
$$

A Python variable/name does **not** conceptually contain the object. It participates in a binding:

$$
B : N \rightharpoonup O
$$

where:

* $N$ = set of names
* $O$ = set of objects
* $\rightharpoonup$ means a partial function

So:

```python
x = [1, 2, 3]
y = x
```

gives approximately:

$$
B(x) = o
$$

$$
B(y) = o
$$

with one object $o$ and two bindings pointing to it.

This is why:

```python
x is y
```

is `True`.

---

# 2. Where reference counting enters

For CPython, define the reference-count function:

$$
RC : O \rightarrow \mathbb{N}
$$

For an object $o$:

$$
RC(o) = |\{r \mid r \text{ currently refers to } o\}|
$$

Very roughly:

```python
x = []
```

creates an object

$$
o = []
$$

and establishes a reference:

$$
x \mapsto o
$$

so:

$$
RC(o) \geq 1
$$

Then:

```python
y = x
```

adds another reference:

$$
x \mapsto o
$$

$$
y \mapsto o
$$

and therefore:

$$
RC(o) \gets RC(o)+1
$$

Conceptually:

```text
        ┌──────────┐
x ─────►│          │
        │ list     │
y ─────►│ object o │
        │          │
        └──────────┘
```

The arrows are **references**, not copies of the object.

---

# 3. The fundamental invariant

The useful formal invariant is:

$$
RC(o) =
\left|
\operatorname{Refs}(o)
\right|
$$

where

$$
\operatorname{Refs}(o)
=
\{r \mid r \rightarrow o\}
$$

So reference counting is essentially maintaining the cardinality of the incoming-reference set.

When a reference is created:

$$
RC(o) \leftarrow RC(o)+1
$$

When a reference disappears:

$$
RC(o) \leftarrow RC(o)-1
$$

And when:

$$
RC(o)=0
$$

CPython can immediately deallocate the object.

That's the key difference from tracing GC: **reference counting determines liveness locally from incoming references.**

---

# 4. But Python variables aren't necessarily "pointers"

This distinction is important.

At the language level:

```python
x = obj
```

means approximately:

$$
x \mapsto obj
$$

It does **not** specify that `x` is a C pointer.

In CPython's implementation, however, a variable/reference is implemented using a pointer-like representation.

For example, conceptually:

```c
PyObject *x;
```

points to a `PyObject`.

So there are three different levels:

| Level                  | Abstraction          |
| ---------------------- | -------------------- |
| Python language        | name → object        |
| CPython implementation | `PyObject*` → object |
| Machine                | address → bytes      |

Don't collapse these.

---

# 5. Are Python objects actually on the heap?

For **CPython**, ordinary Python objects are dynamically allocated.

For example:

```python
x = [1, 2, 3]
```

the list object exists in dynamically managed memory.

Conceptually:

$$
\text{Python object}
\in
\text{Heap}
$$

and the local variable/reference lives in an execution-frame structure.

So a simplified CPython picture is:

$$
\boxed{
\text{Stack/frame state}
\rightarrow
\text{heap objects}
}
$$

For example:

```python
def f():
    x = [1, 2, 3]
```

the conceptual relationship is:

$$
\text{frame} \ni x \rightarrow o_{\text{list}}
$$

where $o_{\text{list}}$ is heap allocated.

---

# 6. But "all Python objects are on the heap" has caveats

This is where the slogan becomes dangerous.

### CPython

Ordinary objects are heap objects, and CPython's C API explicitly has the concept of heap-allocated Python objects.

But the runtime also has:

* stack/local execution state
* interpreter/thread state
* static/global runtime structures
* immortal/singleton objects
* allocator arenas/pools
* temporary C values that aren't Python objects

So:

> **All Python objects are heap allocated**

is a reasonable **CPython object-model approximation**.

But:

> **Everything involved in Python execution is on the heap**

is false.

And Python the language doesn't mandate CPython's particular memory architecture.

---

# 7. The really interesting part: objects contain references too

Suppose:

```python
x = [a, b]
```

There isn't merely:

$$
x \rightarrow \text{list}
$$

The list itself contains references:

$$
x \rightarrow L
$$

and

$$
L \rightarrow a
$$

$$
L \rightarrow b
$$

Therefore reference counting forms a directed graph.

Let:

$$
G = (O,E)
$$

where:

$$
E \subseteq O \times O
$$

and

$$
(o_1,o_2)\in E
$$

means:

> object $o_1$ contains a reference to object $o_2$.

Then Python's runtime object graph is literally a **directed reference graph**.

---

# 8. This explains the famous reference-cycle problem

Consider:

```python
a = []
b = []

a.append(b)
b.append(a)

del a
del b
```

Initially:

$$
a \rightarrow A
$$

$$
b \rightarrow B
$$

and:

$$
A \rightarrow B
$$

$$
B \rightarrow A
$$

After:

```python
del a
del b
```

the external references disappear.

But:

$$
A \rightarrow B \rightarrow A
$$

still exists.

Therefore:

$$
RC(A) = 1
$$

$$
RC(B) = 1
$$

even though neither object is reachable from the program's roots.

This is the crucial limitation:

$$
RC(o)=0
\not\Longleftrightarrow
o\text{ is unreachable}
$$

Reference counting detects:

$$
RC(o)=0
\Rightarrow
o\text{ has no references}
$$

but the converse notion of **semantic garbage** requires graph reachability.

---

# 9. Tracing GC therefore solves a different problem

Define the root set:

$$
Roots \subseteq O
$$

Then define reachability:

$$
Reachable =
\{o \in O \mid
\exists r\in Roots:
r \leadsto o
\}
$$

An object is garbage when:

$$
o \notin Reachable
$$

This can happen even when:

$$
RC(o)>0
$$

because the references can exist entirely inside an unreachable cycle.

So CPython effectively combines two mechanisms:

$$
\boxed{
\text{Reference Counting}
+
\text{Cyclic Garbage Collector}
}
$$

Reference counting handles the easy/local case immediately.

The cyclic GC detects unreachable reference cycles.

---

# 10. Why this is such a nice algebraic model

You can view the runtime as:

$$
\boxed{
\text{Name bindings}
\rightarrow
\text{Object graph}
\rightarrow
\text{Liveness}
\rightarrow
\text{Deallocation}
}
$$

More formally:

$$
B:N\rightharpoonup O
$$

and

$$
G=(O,E)
$$

with:

$$
RC:O\rightarrow\mathbb N
$$

and a root-reachability relation:

$$
Reachable\subseteq O
$$

The two notions of liveness are therefore:

**Reference-count liveness**

$$
RC(o)>0
$$

versus **graph-reachability liveness**

$$
o\in Reachable
$$

They are not equivalent because:

$$
\text{cycles}
\Rightarrow
RC(o)>0
\land
o\notin Reachable
$$

That distinction is the conceptual heart of Python memory management.

### One final correction to the mental model

Don't think:

> "Python puts variables on the heap and objects on the heap."

A better model is:

$$
\boxed{
\text{bindings/references}
\longrightarrow
\text{heap-resident Python objects}
}
$$

The **object is the thing with identity, type, state, and reference count**. A Python variable/name is primarily a **binding to that object**.

That distinction will make your later understanding of `*`, `**`, ownership, borrowing, Rust `Box`, `Rc`, `Arc`, and Python reference counting much cleaner.
