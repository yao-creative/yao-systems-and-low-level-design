**Intent: process-lifecycle semantics — specifically how `fork()` and `exec()` transform the `proc` state while preserving or replacing parts of the process environment.**

The cleanest way to see it is that **`fork` creates a new process state by copying/deriving the parent's process state, while `exec` replaces the process's program/address-space state without creating a new process.**

## 1. `fork`: create a new `proc`

Suppose:

$$
P = (\text{pid},\text{state},\text{chan},\text{files},\text{memory},\text{credentials},\ldots)
$$

Calling:

$$
fork(P)
$$

creates a new process:

$$
P' = \operatorname{clone}(P)
$$

but with selected fields changed.

Conceptually:

$$
\begin{aligned}
pid(P') &\neq pid(P)\
state(P') &= RUNNABLE\
chan(P') &= \bot\
memory(P') &\approx memory(P)\
files(P') &\approx files(P)
\end{aligned}
$$

The important point for your previous question is:

### `fork()` does **not** copy the parent's sleeping state.

If the parent is:

$$
state(P)=SLEEPING
$$

on:

$$
chan(P)=c
$$

the child isn't normally created as another sleeper on $c$.

Instead:

$$
state(P')=RUNNABLE
$$

and:

$$
chan(P')=\bot
$$

because the child begins execution as a new schedulable process.

---

# 2. What happens to open files?

This is one of the most important `fork()` semantics.

Suppose:

$$
FD_P =
{0\mapsto F_0,;1\mapsto F_1,;3\mapsto F_3}
$$

After `fork()`:

$$
FD_{P'} =
{0\mapsto F_0,;1\mapsto F_1,;3\mapsto F_3}
$$

But conceptually these are **two file-descriptor tables referring to the same underlying open-file objects**.

So:

$$
FD_P[3] \rightarrow F_3
$$

and

$$
FD_{P'}[3] \rightarrow F_3
$$

The reference counts on the underlying kernel objects are increased.

This gives the classic behavior:

```text
Parent
  fd 3 ─────┐
            ├──> OpenFile F
Child       │
  fd 3 ─────┘
```

Thus `fork()` duplicates the **process's references**, not necessarily the underlying resource itself.

This is why parent and child can communicate through inherited pipes, sockets, files, etc.

---

# 3. `fork()` and `chan`

This is quite different.

Suppose:

$$
P.state=SLEEPING
$$

and:

$$
P.chan=c
$$

`fork()` doesn't mean:

$$
P' .chan = c
$$

Instead, the child starts independently:

$$
P'.state=RUNNABLE
$$

$$
P'.chan=\bot
$$

because the child isn't waiting for the parent's outstanding event.

So you can think of `fork()` as selectively transforming the process state:

$$
Fork :
Proc \rightarrow Proc\times Proc
$$

but **different components have different cloning rules**.

For example:

$$
\begin{aligned}
Identity &: \text{new}\
Scheduling &: \text{new}\
WaitState &: \text{new}\
AddressSpace &: \text{derived/shared initially}\
OpenFiles &: \text{new references to same resources}\
Credentials &: \text{usually inherited}
\end{aligned}
$$

That's much more accurate than saying simply "fork copies the process."

---

# 4. Then `exec()`

`exec()` is fundamentally different.

It does **not** create:

$$
P'
$$

Instead:

$$
P \xrightarrow{exec(program)} P'
$$

where the **identity of the process remains** but its executable image is replaced.

For example:

$$
pid(P)=42
$$

before:

```text
PID 42
  program = shell
  address space = shell code/data
  files = {stdin, stdout, ...}
```

After:

```text
PID 42
  program = /bin/ls
  address space = ls code/data
  files = {stdin, stdout, ...}
```

So:

$$
pid(P')=pid(P)
$$

but:

$$
addressSpace(P')\neq addressSpace(P)
$$

in the relevant semantic sense.

---

# 5. Why `fork + exec` is so important

The Unix process model intentionally separates:

$$
\boxed{\text{process creation}}
$$

from

$$
\boxed{\text{program replacement}}
$$

So a shell doing:

```c
fork();
exec("/bin/ls", ...);
```

is doing:

$$
ShellProcess
\xrightarrow{fork}
ShellParent + Child
$$

then:

$$
Child
\xrightarrow{exec}
LsProcess
$$

The parent remains the shell:

$$
ShellParent
$$

while the child transforms:

$$
Child_{shell-image}
\rightarrow
Child_{ls-image}
$$

The PID stays with the child.

---

# 6. The open-file relationship explains why this works

Imagine the shell has:

$$
FD =
{0\mapsto stdin,;1\mapsto stdout,;2\mapsto stderr}
$$

After `fork`:

$$
FD_{parent}\approx FD_{child}
$$

Then the child can modify its descriptor table:

```c
dup2(pipe_write, 1);
```

and then:

```c
exec(...)
```

The new program inherits the resulting descriptors.

Therefore:

$$
fork
\rightarrow
FD\ manipulation
\rightarrow
exec
$$

is the basis of Unix shell composition.

For example:

```text
shell
  |
  | fork
  v
child
  |
  | dup2(pipe, stdout)
  |
  | exec
  v
program
```

The **program changes**, but the process's inherited kernel resources can remain.

---

# 7. What survives `exec`?

This is the useful conceptual split:

| `proc` component          | `fork`                          | `exec`                                     |
| ------------------------- | ------------------------------- | ------------------------------------------ |
| PID                       | new                             | **preserved**                              |
| Parent/child relationship | new child                       | preserved                                  |
| Scheduling state          | new                             | preserved/transformed                      |
| `chan`                    | not inherited as sleeping state | normally cleared/not relevant              |
| Open FDs                  | inherited references            | **preserved unless close-on-exec**         |
| Address space             | derived                         | **replaced**                               |
| Program image             | derived                         | **replaced**                               |
| Environment               | inherited                       | supplied to new image                      |
| Current directory         | inherited                       | preserved                                  |
| Credentials               | generally inherited             | generally preserved, subject to exec rules |

So a very useful abstraction is:

$$
Proc
====

Identity
\times
ExecutionContext
\times
ResourceContext
\times
SchedulingContext
$$

Then:

### `fork`

creates a new identity and execution context while deriving resource context:

$$
Fork:
Proc
\rightarrow
Proc\times Proc
$$

with roughly:

$$
ResourceContext_{child}
\sim
ResourceContext_{parent}
$$

### `exec`

keeps identity/resource context but replaces execution context:

$$
Exec:
Proc\times Program
\rightarrow
Proc
$$

with:

$$
\begin{aligned}
Identity' &= Identity\
Resources' &\approx Resources\
ExecutionContext' &= Load(Program)
\end{aligned}
$$

That distinction is the deeper reason Unix has the seemingly strange `fork()` + `exec()` combination: **`fork` is a process-object operation; `exec` is a program-image operation.**
