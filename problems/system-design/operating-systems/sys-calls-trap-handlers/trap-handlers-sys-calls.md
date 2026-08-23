**Intent — OS control-transfer model:** you want the complete causal path from a user-space system call, through the hardware privilege transition and trap handler, into kernel services such as I/O or memory management, and then back to user mode.

```mermaid
sequenceDiagram
    autonumber

    participant U as User Process<br/>User Mode
    participant CPU as CPU / Hardware
    participant IVT as Trap / Syscall Vector
    participant K as Kernel<br/>Kernel Mode
    participant IO as I/O Device / Driver
    participant MM as Memory Manager

    U->>U: Execute system call procedure
    U->>CPU: syscall / trap instruction

    Note over CPU: Hardware performs privilege transition
    CPU->>CPU: Save user PC, flags, registers
    CPU->>CPU: Save/establish privilege state
    CPU->>CPU: Raise privilege level<br/>User → Kernel
    CPU->>IVT: Lookup syscall/trap vector

    IVT->>K: Transfer control to trap handler

    Note over K: Kernel mode
    K->>K: Trap handler saves kernel-visible context
    K->>K: Decode syscall number + arguments
    K->>K: Validate user arguments / capabilities

    alt I/O request
        K->>IO: Submit I/O request
        IO-->>K: Completion / result
    else Need more memory
        K->>MM: Allocate / map / reclaim memory
        MM-->>K: Memory available / allocation result
    else Other kernel service
        K->>K: Execute kernel operation
    end

    K->>K: Prepare return value
    K->>CPU: return-from-trap instruction

    Note over CPU: Hardware reverses privilege transition
    CPU->>CPU: Restore saved registers
    CPU->>CPU: Restore user PC + flags
    CPU->>CPU: Lower privilege level<br/>Kernel → User

    CPU->>U: Resume user process
    U->>U: Continue after system call
```

The key distinction is that **the procedure call itself does not create the privilege transition**:

$$
\text{User procedure}
\rightarrow
\text{syscall instruction}
\rightarrow
\boxed{\text{hardware-controlled trap}}
\rightarrow
\text{kernel trap handler}
$$

A normal procedure call is essentially:

$$
PC \mapsto f
$$

while a system call is a **controlled change of execution domain**:

$$
(U,; PC,; S_U)
\xrightarrow{\text{syscall}}
(K,; PC_{\mathrm{handler}},; S_K)
$$

where:

* $U$ = user privilege domain
* $K$ = kernel privilege domain
* $S_U$ = saved user CPU state
* $S_K$ = kernel execution state

The return instruction performs the inverse control transfer:

$$
(K,; S_K,; S_U)
\xrightarrow{\text{return-from-trap}}
(U,; S_U)
$$

### The important causal hierarchy

```mermaid
flowchart TD
    A["User Mode<br/>Application"] --> B["System Call API / Procedure"]
    B --> C["syscall / trap instruction"]

    C --> D["CPU Hardware"]
    D --> E["Save execution context"]
    D --> F["Raise privilege<br/>User → Kernel"]
    D --> G["Select trap vector"]

    E --> H["Trap Handler"]
    F --> H
    G --> H

    H --> I["Kernel System Call Dispatcher"]

    I --> J["I/O Subsystem / Driver"]
    I --> K["Memory Manager"]
    I --> L["Other Kernel Subsystem"]

    J --> M["Result"]
    K --> M
    L --> M

    M --> N["Prepare return state"]
    N --> O["return-from-trap"]
    O --> P["CPU restores user context"]
    P --> Q["Lower privilege<br/>Kernel → User"]
    Q --> R["Resume User Process"]
```

So the **trap handler is the boundary object** between the two execution regimes:

$$
\boxed{
\text{User computation}
;\middle|;
\text{hardware-enforced control transfer}
;\middle|;
\text{Kernel computation}
}
$$

And importantly, **I/O and memory allocation happen after entering kernel mode**; they are not themselves what causes the privilege transition. The `syscall`/trap mechanism causes the transition, while the kernel decides which protected resource operation to perform.
