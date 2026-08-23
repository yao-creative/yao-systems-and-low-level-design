**Intent — formal machine-interface specification:** you want each component of the syscall/trap path defined as a mathematical object, its state space, and the transformations between them.

The cleanest decomposition is to separate **machine state**, **protocol data**, **control transfer**, and **kernel state**.

## 1. Global machine model

Let:

$$
\mathcal{R}
===========

\text{set of CPU registers}
$$

$$
\mathcal{V}
===========

\text{set of machine words}
$$

$$
\mathcal{A}
===========

\text{set of machine addresses}
$$

$$
\mathcal{P}
===========

{\mathsf{User},\mathsf{Kernel}}
$$

and define the CPU state as:

$$
\boxed{
\mathsf{CPUState}
=================

\mathcal{R}
\times
\mathcal{A}
\times
\mathcal{A}
\times
\mathcal{F}
\times
\mathcal{P}
}
$$

where:

* registers
* program counter
* stack pointer
* flags/status
* privilege level

are components of the state.

Set-theoretically:

$$
C
=

R \times PC \times SP \times F \times P
$$

is the set of all possible CPU states.

---

# 2. User program

A program is a state-transition system:

$$
\mathsf{Program}
================

(\mathsf{State},\delta)
$$

where:

$$
\delta :
\mathsf{State}
\rightarrow
\mathsf{State}
$$

describes ordinary user-mode execution.

The important restriction is:

$$
\forall s\in\mathsf{State}_{U},
\qquad
\operatorname{priv}(s)=\mathsf{User}
$$

The program cannot arbitrarily construct a kernel state.

---

# 3. Procedure call

A normal procedure call is fundamentally a **control-flow transformation**.

Let:

$$
\mathsf{Proc}
=============

\mathsf{Args}\to\mathsf{Result}
$$

and let:

$$
pc : \mathsf{Code}\to\mathsf{Address}
$$

A procedure call changes the continuation:

$$
\boxed{
\mathsf{Call}_f :
C_U \rightarrow C_U
}
$$

It remains inside the same privilege domain:

$$
\mathsf{User}
\rightarrow
\mathsf{User}
$$

This is the key distinction from a syscall.

---

# 4. Syscall ABI

A syscall is not fundamentally an object. It is a **protocol for interpreting CPU state**.

Define:

$$
\mathsf{SyscallId}
$$

as the set of syscall identifiers.

For example:

$$
\mathsf{SyscallId}
==================

{
\mathsf{read},
\mathsf{write},
\mathsf{open},
\mathsf{mmap},
\ldots
}
$$

For each syscall $s$, define its argument type:

$$
\mathsf{Args}(s)
$$

Then the syscall request space is the dependent sum:

$$
\boxed{
\mathsf{SyscallRequest}
=======================

\sum_{s\in\mathsf{SyscallId}}
\mathsf{Args}(s)
}
$$

This says:

> A request consists of a syscall identifier together with arguments whose type depends on that identifier.

For example:

$$
\mathsf{write}
:
(\mathsf{Fd},\mathsf{Buffer},\mathsf{Length})
$$

---

# 5. Register-level syscall encoding

The ABI provides an encoding:

$$
\boxed{
E :
\mathsf{SyscallRequest}
\rightarrow
\mathsf{Registers}
}
$$

and a corresponding decoding:

$$
\boxed{
D :
\mathsf{Registers}
\rightharpoonup
\mathsf{SyscallRequest}
}
$$

where $\rightharpoonup$ means a **partial function**, because arbitrary register contents need not represent a valid syscall.

Ideally:

$$
D(E(x))=x
$$

for every valid syscall request $x$.

Thus:

$$
\boxed{
\text{SyscallRequest}
\overset{E}{\longrightarrow}
\text{Register State}
}
$$

is the ABI.

---

# 6. `syscall` instruction

The instruction itself is a transition operator:

$$
\boxed{
\tau :
\mathsf{CPUState}_U
\rightarrow
\mathsf{CPUState}_K
\times
\mathsf{TrapCause}
}
$$

The important thing is that $\tau$ is **hardware-defined**.

It performs something conceptually equivalent to:

$$
\begin{aligned}
PC_K &\leftarrow \mathsf{TrapVector}[\mathsf{Syscall}]\
P_K &\leftarrow \mathsf{Kernel}\
T &\leftarrow \mathsf{Save}(CPU_U)
\end{aligned}
$$

So:

$$
\boxed{
\texttt{syscall}
\neq
\text{kernel function call}
}
$$

It is a machine instruction whose semantic effect is:

$$
\text{user execution}
\rightarrow
\text{architecturally controlled trap entry}
$$

---

# 7. Trap cause

The CPU needs to tell the kernel *why* control was transferred.

Define:

$$
\boxed{
\mathsf{TrapCause}
==================

\mathsf{Syscall}
+
\mathsf{Exception}
+
\mathsf{Interrupt}
}
$$

In algebraic-data-type notation:

$$
\mathsf{TrapCause}
==================

\begin{cases}
\mathsf{Syscall}(n)\
\mathsf{PageFault}(a,r)\
\mathsf{IllegalInstruction}(i)\
\mathsf{TimerInterrupt}\
\mathsf{DeviceInterrupt}(d)\
\vdots
\end{cases}
$$

This is essentially a **sum type**.

The kernel first performs case analysis over this sum.

---

# 8. Trap vector

The trap vector is a mapping from causes to privileged entry points:

$$
\boxed{
V :
\mathsf{TrapCause}
\rightarrow
\mathsf{KernelAddress}
}
$$

Conceptually:

$$
V(\mathsf{Syscall}) = pc_{\mathsf{syscall}}
$$

$$
V(\mathsf{PageFault}) = pc_{\mathsf{pagefault}}
$$

etc.

Hardware performs:

$$
PC_K := V(c)
$$

where $c$ is the trap cause.

So the vector is essentially a **dispatch table in the hardware-defined control-transfer mechanism**.

---

# 9. Trap frame

Now we reach the central data structure.

Let:

$$
\mathsf{MachineContext}
=======================

\mathsf{Registers}
\times
\mathsf{PC}
\times
\mathsf{SP}
\times
\mathsf{Flags}
\times
\mathsf{PrivilegeState}
$$

A trap frame is a representation of such a context:

$$
\boxed{
\mathsf{TrapFrame}
\subseteq
\mathsf{MachineContext}
\times
\mathsf{TrapMetadata}
}
$$

More explicitly:

$$
T =
(R,PC,SP,F,P,C,E)
$$

where:

* $R$ = registers
* $PC$ = saved instruction pointer
* $SP$ = saved stack pointer
* $F$ = saved flags
* $P$ = saved privilege information
* $C$ = trap cause
* $E$ = architecture-specific error information

The exact structure is ISA-specific.

---

# 10. Hardware trap-entry function

Define:

$$
\boxed{
\mathsf{EnterTrap}
:
\mathsf{CPUState}_U
\times
\mathsf{TrapCause}
\rightarrow
\mathsf{TrapFrame}
\times
\mathsf{CPUState}_K
}
$$

Conceptually:

$$
\mathsf{EnterTrap}(c,u)
=======================

(t,k)
$$

where:

$$
t=\mathsf{Save}(u,c)
$$

and:

$$
k.P=\mathsf{Kernel}
$$

$$
k.PC=V(c)
$$

Thus:

$$
\boxed{
u
\xrightarrow{\mathsf{EnterTrap}}
(t,k)
}
$$

This is the formal boundary between hardware and the kernel.

---

# 11. Trap handler

The trap handler is a kernel function:

$$
\boxed{
H :
\mathsf{TrapFrame}
\times
\mathsf{KernelState}
\rightarrow
\mathsf{TrapFrame}
\times
\mathsf{KernelState}
}
$$

It interprets the trap:

$$
H(T,K)
======

(T',K')
$$

For a syscall:

$$
H(T,K)
======

\mathsf{SyscallDispatch}(T,K)
$$

For a page fault:

$$
H(T,K)
======

\mathsf{PageFaultHandler}(T,K)
$$

For an interrupt:

$$
H(T,K)
======

\mathsf{InterruptHandler}(T,K)
$$

So the handler is a **state transformer**.

---

# 12. Syscall dispatcher

The dispatcher extracts the syscall request:

$$
D :
\mathsf{TrapFrame}
\rightharpoonup
\mathsf{SyscallRequest}
$$

Then dispatches:

$$
\boxed{
\mathsf{Dispatch}
:
\mathsf{SyscallRequest}
\times K
\rightarrow
\mathsf{Result}
\times K
}
$$

For example:

$$
\mathsf{Dispatch}
(\mathsf{write}(fd,b,n),K)
\rightarrow
(r,K')
$$

The return value is then placed back into the trap frame:

$$
T'.R_{\mathsf{return}}
======================

r
$$

Therefore the trap frame becomes the **return-value transport mechanism** as well.

---

# 13. Kernel state

Let:

$$
\boxed{
K
=

K_{\text{memory}}
\times
K_{\text{process}}
\times
K_{\text{scheduler}}
\times
K_{\text{io}}
\times
K_{\text{security}}
\times\cdots
}
$$

The syscall causes:

$$
K\rightarrow K'
$$

For example, an I/O syscall may transform:

$$
K_{\text{io}}
\rightarrow
K_{\text{io}}'
$$

while a memory syscall may transform:

$$
K_{\text{memory}}
\rightarrow
K_{\text{memory}}'
$$

The trap mechanism itself does **not** perform the I/O or allocation.

It merely establishes the privileged execution context in which the kernel can perform them.

---

# 14. I/O request

An I/O operation can be modeled as a request algebra:

$$
\mathsf{IORequest}
==================

\sum_{d\in\mathsf{Device}}
\mathsf{Operation}(d)
$$

For example:

$$
\mathsf{Read}(d,b,n)
$$

or:

$$
\mathsf{Write}(d,b,n)
$$

The kernel transforms:

$$
(K,\mathsf{IORequest})
\rightarrow
(K',\mathsf{IOResult})
$$

The hardware/device subsystem may then independently produce an interrupt:

$$
\mathsf{Device}
\rightarrow
\mathsf{Interrupt}
\rightarrow
\mathsf{TrapHandler}
$$

So an I/O syscall can actually involve **two control transfers**:

$$
\text{User}
\rightarrow
\text{Kernel}
\rightarrow
\text{Device}
$$

and later:

$$
\text{Device}
\rightarrow
\text{Kernel}
$$

---

# 15. Memory-management operation

Similarly:

$$
\mathsf{MemoryRequest}
======================

\mathsf{Allocate}
+
\mathsf{Map}
+
\mathsf{Unmap}
+
\mathsf{Protect}
+
\mathsf{Reclaim}
+\cdots
$$

For example:

$$
\mathsf{Allocate}(n,p)
$$

The memory manager is another kernel state transformer:

$$
\boxed{
M :
\mathsf{MemoryRequest}
\times
K_{\text{memory}}
\rightarrow
K_{\text{memory}}
\times
\mathsf{Result}
}
$$

---

# 16. Return value

Suppose the syscall produces:

$$
r\in\mathsf{ReturnValue}
$$

The kernel transforms the trap frame:

$$
\boxed{
\mathsf{SetReturn}
:
\mathsf{TrapFrame}
\times
\mathsf{ReturnValue}
\rightarrow
\mathsf{TrapFrame}
}
$$

so that:

$$
T'.R_{\text{return}}=r
$$

Now the trap frame represents exactly the machine state the user program should observe when execution resumes.

---

# 17. Return-from-trap

The inverse hardware operation is:

$$
\boxed{
\mathsf{ExitTrap}
:
\mathsf{TrapFrame}
\rightarrow
\mathsf{CPUState}_U
}
$$

Conceptually:

$$
\begin{aligned}
PC &\leftarrow T.PC\
SP &\leftarrow T.SP\
R &\leftarrow T.R\
F &\leftarrow T.F\
P &\leftarrow \mathsf{User}
\end{aligned}
$$

Thus:

$$
\boxed{
\mathsf{ExitTrap}(\mathsf{EnterTrap}(U))
\approx U
}
$$

up to intentional modifications such as the syscall return value.

More precisely:

$$
\mathsf{ExitTrap}(T')
=====================

U'
$$

where:

$$
U'.R_{\text{return}}=r
$$

---

# 18. The complete composition

Now everything can be composed:

$$
\boxed{
U
\xrightarrow{\mathsf{Encode}}
\mathsf{SyscallRegisters}
\xrightarrow{\mathsf{syscall}}
\mathsf{EnterTrap}
\xrightarrow{}
T
\xrightarrow{\mathsf{TrapHandler}}
T'
\xrightarrow{\mathsf{return\mbox{-}from\mbox{-}trap}}
U'
}
$$

More formally:

$$
\begin{aligned}
U
&\xrightarrow{E}
R_{\text{syscall}}
\
&\xrightarrow{\tau}
(T,K)
\
&\xrightarrow{H}
(T',K')
\
&\xrightarrow{\mathsf{ExitTrap}}
U'
\end{aligned}
$$

with the central invariant:

$$
\boxed{
U.P=\mathsf{User}
\qquad
K.P=\mathsf{Kernel}
}
$$

and the security property:

$$
\boxed{
\text{User code cannot directly construct an arbitrary valid }K.
}
$$

It must pass through the hardware-enforced transition.

---

## 19. Category-theoretic view

There is a particularly clean way to see the architecture.

Treat each state space as an object in **Set**:

$$
U,K,T,R,Q
\in \operatorname{Ob}(\mathbf{Set})
$$

and each deterministic transition as a morphism:

$$
U
\xrightarrow{E}
R
\xrightarrow{\tau}
T
\xrightarrow{H}
T'
\xrightarrow{\rho}
U'
$$

where:

* $E$ = ABI encoding
* $\tau$ = hardware trap entry
* $H$ = kernel handler
* $\rho$ = hardware trap return

The important conceptual split is:

$$
\boxed{
\underbrace{E,H}*{\text{software semantics}}
\qquad
\underbrace{\tau,\rho}*{\text{hardware semantics}}
}
$$

And the trap frame is the **interface object** through which the two regimes compose.

In other words, the architecture is not fundamentally:

$$
\text{application}\rightarrow\text{kernel function}
$$

but:

$$
\boxed{
\text{user state}
\rightarrow
\text{encoded protocol}
\rightarrow
\text{hardware state transition}
\rightarrow
\text{machine-state representation}
\rightarrow
\text{kernel state transition}
\rightarrow
\text{machine-state representation}
\rightarrow
\text{hardware state transition}
\rightarrow
\text{user state}
}
$$

That distinction is the key to understanding why a syscall is fundamentally different from an ordinary procedure call.
