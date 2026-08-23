**Intent: cross-language translation of the same policy–mechanism abstraction from Python into idiomatic Rust.**

The clean Rust version is particularly nice because the type system can make the policy/mechanism composition explicit.

## 1. Start with the functional version

```rust
#[derive(Debug, Clone)]
struct MemoryState {
    available_pages: usize,
    processes: Vec<Process>,
}

#[derive(Debug, Clone)]
struct Process {
    id: String,
    pages: usize,
}

#[derive(Debug)]
enum Action {
    Reclaim { process: String },
    Kill { process: String },
}
```

The **policy**:

```rust
fn choose_action(state: &MemoryState) -> Action {
    let process = state
        .processes
        .iter()
        .max_by_key(|p| p.pages)
        .expect("at least one process");

    Action::Reclaim {
        process: process.id.clone(),
    }
}
```

The **mechanism**:

```rust
fn execute_action(
    mut state: MemoryState,
    action: Action,
) -> MemoryState {
    match action {
        Action::Reclaim { .. } => {
            state.available_pages += 50;
            state
        }

        Action::Kill { process } => {
            if let Some(index) = state
                .processes
                .iter()
                .position(|p| p.id == process)
            {
                let process = state.processes.remove(index);
                state.available_pages += process.pages;
            }

            state
        }
    }
}
```

And the coordinator:

```rust
fn step(state: MemoryState) -> MemoryState {
    let action = choose_action(&state);
    execute_action(state, action)
}
```

The architecture is exactly:

$$
S
\xrightarrow{\pi}
A
\xrightarrow{M}
S'
$$

or:

$$
step(s)=M(s,\pi(s))
$$

---

# 2. Notice the ownership boundary

This is where Rust becomes particularly interesting.

The policy:

```rust
fn choose_action(state: &MemoryState) -> Action
```

only **borrows** state.

So:

$$
Policy : &S \to A
$$

It cannot mutate the runtime state.

The mechanism:

```rust
fn execute_action(
    state: MemoryState,
    action: Action,
) -> MemoryState
```

owns the transition:

$$
Mechanism : S\times A\to S
$$

But we could instead make the runtime own the state and let the mechanism borrow it mutably:

```rust
fn execute_action(
    state: &mut MemoryState,
    action: Action,
) {
    // mutate state
}
```

Then:

$$
Mechanism : &mut S\times A\to ()
$$

This is very close to the functional-core/imperative-shell pattern you have been exploring.

---

# 3. Parameterize the policy

Now suppose we have two policies:

```rust
fn reclaim_first(state: &MemoryState) -> Action {
    let process = state
        .processes
        .iter()
        .max_by_key(|p| p.pages)
        .expect("at least one process");

    Action::Reclaim {
        process: process.id.clone(),
    }
}

fn kill_first(state: &MemoryState) -> Action {
    let process = state
        .processes
        .iter()
        .max_by_key(|p| p.pages)
        .expect("at least one process");

    Action::Kill {
        process: process.id.clone(),
    }
}
```

Then:

```rust
fn step(
    state: &mut MemoryState,
    policy: impl Fn(&MemoryState) -> Action,
) {
    let action = policy(state);

    execute_action(state, action);
}
```

Now:

```rust
step(&mut state, reclaim_first);
```

or:

```rust
step(&mut state, kill_first);
```

The mechanism remains unchanged.

This is probably the **first version I would actually write** unless you have a reason to introduce traits.

---

# 4. Trait version

If policy is a meaningful abstraction boundary, define:

```rust
trait Policy {
    type Action;

    fn choose(&self, state: &MemoryState) -> Self::Action;
}
```

and:

```rust
trait Mechanism {
    type Action;

    fn execute(
        &self,
        state: &mut MemoryState,
        action: Self::Action,
    );
}
```

Implementation:

```rust
struct ReclaimFirst;

impl Policy for ReclaimFirst {
    type Action = Action;

    fn choose(&self, state: &MemoryState) -> Self::Action {
        let process = state
            .processes
            .iter()
            .max_by_key(|p| p.pages)
            .expect("at least one process");

        Action::Reclaim {
            process: process.id.clone(),
        }
    }
}
```

Mechanism:

```rust
struct MemoryMechanism;

impl Mechanism for MemoryMechanism {
    type Action = Action;

    fn execute(
        &self,
        state: &mut MemoryState,
        action: Self::Action,
    ) {
        match action {
            Action::Reclaim { .. } => {
                state.available_pages += 50;
            }

            Action::Kill { process } => {
                if let Some(index) = state
                    .processes
                    .iter()
                    .position(|p| p.id == process)
                {
                    let process = state.processes.remove(index);
                    state.available_pages += process.pages;
                }
            }
        }
    }
}
```

Now the coordinator:

```rust
fn step<P, M>(
    state: &mut MemoryState,
    policy: &P,
    mechanism: &M,
)
where
    P: Policy,
    M: Mechanism<Action = P::Action>,
{
    let action = policy.choose(state);

    mechanism.execute(state, action);
}
```

The important constraint is:

```rust
M: Mechanism<Action = P::Action>
```

which says:

$$
Action_P = Action_M
$$

The mechanism and policy must agree on the action algebra.

---

# 5. Why the associated type is useful

Instead of:

```rust
trait Policy {
    fn choose(&self, state: &MemoryState) -> Action;
}
```

we wrote:

```rust
trait Policy {
    type Action;

    fn choose(&self, state: &MemoryState) -> Self::Action;
}
```

This means a policy defines its own action space.

Mathematically:

$$
P : S\to A_P
$$

while the mechanism defines:

$$
M : S\times A_M\to S
$$

and the bound:

```rust
M: Mechanism<Action = P::Action>
```

requires:

$$
A_P=A_M
$$

This becomes very useful when you have different domains.

---

# 6. A more realistic runtime

You can make the **runtime the sole owner of state**:

```rust
struct Runtime<P, M> {
    state: MemoryState,
    policy: P,
    mechanism: M,
}
```

Then:

```rust
impl<P, M> Runtime<P, M>
where
    P: Policy,
    M: Mechanism<Action = P::Action>,
{
    fn step(&mut self) {
        let action = self.policy.choose(&self.state);

        self.mechanism.execute(
            &mut self.state,
            action,
        );
    }
}
```

Construction:

```rust
let mut runtime = Runtime {
    state,
    policy: ReclaimFirst,
    mechanism: MemoryMechanism,
};

runtime.step();
```

The ownership graph is now extremely clean:

$$
Runtime \owns State
$$

while:

$$
Policy : &State\to Action
$$

and:

$$
Mechanism : &mut State\times Action\to ()
$$

Neither policy nor mechanism owns the state.

---

# 7. This is the abstraction I would recommend for your style

I'd distinguish the layers like this:

$$
\boxed{
\begin{aligned}
&\text{Data}\
&\quad MemoryState,\ Action[4pt]
&\text{Policy}\
&\quad &State\rightarrow Action[4pt]
&\text{Mechanism}\
&\quad &mut State\times Action\rightarrow ()[4pt]
&\text{Runtime}\
&\quad owns\ State,\ coordinates\ Policy+Mechanism
\end{aligned}
}
$$

And importantly:

**Don't make `Policy` own `MemoryState`.**

Don't do:

```rust
struct Policy {
    state: MemoryState,
}
```

if policy is conceptually just a decision procedure.

Prefer:

```rust
trait Policy {
    type Action;

    fn choose(&self, state: &MemoryState) -> Self::Action;
}
```

That gives you the functional component you prefer:

$$
Policy : &S\to A
$$

while the runtime retains the authoritative state:

$$
Runtime\owns S
$$

This is a very strong pattern for the kernel/runtime designs you've been working toward: **state ownership, decision logic, and effectful realization become three separate axes rather than one giant stateful object.**
