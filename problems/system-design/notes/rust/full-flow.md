The full Rust development flow can be understood as a pipeline from **source code** to **executing process**. Each stage has a distinct artifact and responsibility.

```text
Developer
    ↓
Rust Source
    ↓
Cargo
    ↓
rustc
    ↓
LLVM
    ↓
Assembler
    ↓
Linker
    ↓
Executable
    ↓
Operating System Loader
    ↓
Process
    ↓
Main Thread
    ↓
Rust Startup
    ↓
main()
    ↓
Application Runtime
```

Let's go through each stage.

---

# 1. Write Rust

You write

```rust
fn main() {
    println!("Hello");
}
```

This is just text.

Artifact:

$$
S = \text{Rust source files}.
$$

---

# 2. Cargo

Cargo is the **project manager**.

It

* resolves dependencies
* downloads crates
* runs build scripts
* invokes `rustc`
* manages workspaces
* caches builds

It does **not** compile directly.

Think of Cargo as

```text
Project Manager
```

rather than

```text
Compiler
```

---

# 3. rustc

The Rust compiler

* parses
* type checks
* borrow checks
* performs trait resolution
* monomorphizes generics
* optimizes MIR

Conceptually

```text
Source
 ↓
AST
 ↓
HIR
 ↓
MIR
 ↓
LLVM IR
```

where

* AST = syntax
* HIR = semantic representation
* MIR = optimization representation

---

# 4. LLVM

Rust hands LLVM

```text
LLVM IR
```

LLVM then performs

* register allocation
* instruction scheduling
* vectorization
* machine-specific optimizations

Output

```text
Machine instructions
```

---

# 5. Assembler

Turns instructions into

```text
Object files (.o)
```

These contain machine code but are incomplete.

---

# 6. Linker

The linker combines

* your object files
* Rust standard library
* C runtime (when applicable)
* external libraries

into

```text
my_program
```

or

```text
my_program.exe
```

---

# 7. Executable

Now you have a native executable.

On Linux

```
ELF
```

On macOS

```
Mach-O
```

On Windows

```
PE
```

These formats describe

* code
* data
* symbols
* libraries
* entry point

---

# 8. OS Loader

When you execute

```bash
./my_program
```

the operating system

* creates a process
* creates virtual memory
* maps executable pages
* maps shared libraries
* creates the main thread

---

# 9. Rust startup

The CPU does **not** jump directly into `main()`.

Instead

```text
OS
 ↓
_start
 ↓
Rust startup
 ↓
main()
```

Startup code

* prepares argc/argv
* initializes the standard library
* handles panic setup
* eventually calls

```rust
fn main()
```

---

# 10. Your runtime

Now your application starts.

For example

```text
main
 ↓
Bootstrap
 ↓
Host
 ↓
ApplicationRuntime
 ↓
Controllers
 ↓
Services
 ↓
Domain
```

Everything from here onward is your architecture.

---

# 11. Runtime execution

Maybe

```rust
std::thread::spawn(...)
```

creates

```
Main Thread
Worker Thread
Worker Thread
Worker Thread
```

or

```rust
#[tokio::main]
```

creates

```
Tokio Runtime
 ↓
Executor
 ↓
Async Tasks
```

---

# 12. Shutdown

Eventually

```text
main returns
 ↓
Drop values
 ↓
Destroy runtime
 ↓
Terminate threads
 ↓
OS destroys process
```

---

# Where Docker fits

Docker is **outside** this pipeline.

```
Rust source
 ↓
Cargo
 ↓
Executable
 ↓
Docker image
 ↓
Deployment
 ↓
Server
```

Docker packages your executable together with its execution environment (filesystem, libraries, configuration), but it is not part of compilation or execution on the target machine.

---

# The complete hierarchy

From the broadest systems perspective, the flow is

```text
Hardware
    ↓
Firmware (UEFI/BIOS)
    ↓
Operating System Kernel
    ↓
Executable Loader
    ↓
Process
    ↓
Main Thread
    ↓
Rust Startup (_start)
    ↓
main()
    ↓
Bootstrap
    ↓
Host
    ↓
Application Runtime
    ↓
Executor / Scheduler
    ↓
Controllers
    ↓
Services
    ↓
Domain Logic
    ↓
Functions
```

This hierarchy is useful because each layer has a single responsibility: hardware executes instructions; the OS manages processes and threads; Rust provides compile-time safety and minimal startup; your application host manages lifecycle; and your domain logic implements the behavior of your program.
