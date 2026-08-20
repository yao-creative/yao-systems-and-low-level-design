Your **intent** is a **type-theoretic/categorical formalization of effectful computation**: specifically, you want to understand why `Result` has the algebraic form `A + E`, how monadic composition operates on it, and how a capability-dependent computation differs from the effect/result it produces.

The cleanest way is to start from sets and functions.

---

# 1. What does `A + E` mean?

Here `+` is **not ordinary numerical addition**.

It denotes the **disjoint union / coproduct of sets**.

If:

$$
A = {\text{values of type } A}
$$

and

$$
E = {\text{errors of type } E}
$$

then:

$$
A + E
$$

means:

$$
A + E
=====

({0}\times A)
\cup
({1}\times E)
$$

So every element is tagged as being either an `A` or an `E`.

Rust represents this with:

```rust
enum Result<A, E> {
    Ok(A),
    Err(E),
}
```

Thus:

$$
\operatorname{Result}(A,E) \cong A + E
$$

For example:

```rust
type UserId = String;

enum AuthError {
    InvalidCredentials,
    NetworkFailure,
}
```

Then:

$$
Result(UserId, AuthError)
$$

is the set:

$$
UserId + AuthError
$$

An actual value is either:

```rust
Ok("alice")
```

or:

```rust
Err(NetworkFailure)
```

There is **no third possibility**.

---

# 2. Why does `Result` have this shape?

Suppose you have an ordinary function:

$$
f : A \to B
$$

It says:

> Every `A` produces a `B`.

But authentication isn't total in this sense.

You actually have:

$$
f : A \to B + E
$$

because an input can produce either:

$$
A \mapsto B
$$

or:

$$
A \mapsto E
$$

For example:

$$
authenticate :
Credentials
\to
User + AuthError
$$

In Rust:

```rust
fn authenticate(
    credentials: Credentials,
) -> Result<User, AuthError>
```

This is an ordinary function whose **codomain is a coproduct**.

That's the first important conceptual point:

> `Result<T,E>` isn't "a value with some error metadata attached." Algebraically, it is a value in the coproduct `T + E`.

---

# 3. Now where does

$$
(A+E)
\to
(A\to(B+E))
\to
(B+E)
$$

come from?

This is the **monadic composition problem**.

Suppose we already have:

$$
x : A+E
$$

and another computation:

$$
f : A \to B+E
$$

We want to combine them.

The problem is:

$$
x
$$

might contain an `A`:

$$
x = Ok(a)
$$

or an `E`:

$$
x = Err(e)
$$

If it contains `A`, we can apply:

$$
f(a)
$$

But if it contains `E`, there is no `A` to give to `f`.

So the desired operation is:

$$
(A+E)
\times
(A\to B+E)
\to
B+E
$$

This is exactly what `Result::and_then` does.

---

# 4. Expand `and_then` algebraically

Imagine:

```rust
fn bind<A, B, E>(
    x: Result<A, E>,
    f: impl FnOnce(A) -> Result<B, E>,
) -> Result<B, E>
```

Its definition is essentially:

```rust
match x {
    Ok(a) => f(a),
    Err(e) => Err(e),
}
```

Mathematically:

$$
bind(x,f)
=========

\begin{cases}
f(a) & x = Ok(a)\
Err(e) & x = Err(e)
\end{cases}
$$

So:

$$
bind :
(A+E)
\times
(A\to B+E)
\to
B+E
$$

Notice the crucial asymmetry:

### Success branch

$$
A
\xrightarrow{f}
B+E
$$

### Error branch

$$
E
\xrightarrow{id}
E
\subseteq B+E
$$

The error is simply propagated.

---

# 5. Concrete example

Suppose:

```rust
fn parse_id(s: String) -> Result<u64, ParseError>;

fn lookup_user(id: u64) -> Result<User, DbError>;
```

Already there's a slight issue: the errors differ.

Conceptually, though, suppose both have a common error type:

```rust
enum AppError {
    Parse(ParseError),
    Database(DbError),
}
```

Then:

```rust
fn parse_id(
    s: String,
) -> Result<u64, AppError>;

fn lookup_user(
    id: u64,
) -> Result<User, AppError>;
```

Now:

$$
parse:
String \to u64 + E
$$

and:

$$
lookup:
u64 \to User + E
$$

Composition becomes:

$$
String
\to
u64+E
\to
User+E
$$

where the second arrow is not ordinary function composition.

It is:

$$
bind :
(u64+E)
\times
(u64\to User+E)
\to
User+E
$$

Rust:

```rust
parse_id(input)
    .and_then(lookup_user)
```

Or:

```rust
let id = parse_id(input)?;
let user = lookup_user(id)?;
Ok(user)
```

The `?` syntax is essentially making the same algebraic operation ergonomic.

---

# 6. Why can't we just compose ordinary functions?

Ordinary composition expects:

$$
f:A\to B
$$

and:

$$
g:B\to C
$$

giving:

$$
g\circ f:A\to C
$$

But `Result` functions look like:

$$
f:A\to B+E
$$

and:

$$
g:B\to C+E
$$

You can't directly compose them because:

$$
B+E \neq B
$$

The intermediate result contains an additional computational branch.

The monad supplies a **composition operation for these lifted functions**.

This is one of the deepest practical meanings of a monad:

> A monad gives you a principled way to compose computations whose outputs carry some additional computational structure.

For `Result`, the structure is **failure propagation**.

---

# 7. The three operations of a monad

For a type constructor:

$$
M : Set \to Set
$$

a monad gives you essentially two fundamental operations.

## `pure` / `return`

Embed an ordinary value:

$$
\eta_A : A \to M(A)
$$

For `Result`:

$$
\eta_A(a)=Ok(a)
$$

Rust:

```rust
Ok(value)
```

---

## `bind`

Compose a computation with a function producing another computation:

$$
\bind :
M(A)
\times
(A\to M(B))
\to
M(B)
$$

For `Result`:

$$
(A+E)
\times
(A\to B+E)
\to
B+E
$$

Rust:

```rust
result.and_then(f)
```

---

# 8. Why is this called a monad?

Because `bind` satisfies the monad laws.

## Left identity

Starting with an ordinary value:

$$
\eta(a) \bind f = f(a)
$$

For `Result`:

```rust
Ok(a).and_then(f)
```

is exactly:

```rust
f(a)
```

---

## Right identity

For a computation `m`:

$$
m \bind \eta = m
$$

In Rust:

```rust
m.and_then(|x| Ok(x))
```

returns the same result.

---

## Associativity

For:

$$
m : M(A)
$$

$$
f:A\to M(B)
$$

$$
g:B\to M(C)
$$

we require:

$$
(m\bind f)\bind g
=================

m\bind(\lambda x.f(x)\bind g)
$$

This is what allows you to regroup chains without changing their meaning.

For Rust:

```rust
m.and_then(f).and_then(g)
```

is equivalent in semantics to:

```rust
m.and_then(|x| {
    f(x).and_then(g)
})
```

This is why long `Result` pipelines compose predictably.

---

# 9. Now: what is a capability-dependent operation?

This is a different dimension.

Suppose your application needs a database.

A pure function might be:

$$
f:A\to B
$$

But a database-dependent function needs some capability/environment `R`.

So its actual mathematical type is:

$$
f:R\times A\to B
$$

where:

$$
R = \text{available capabilities}
$$

For example:

$$
R =
Storage
\times
ModelClient
\times
EventSink
$$

Then:

$$
loadUser :
R\times UserId
\to
User
$$

This means:

> Given a capability environment and an input, produce a result.

---

# 10. Currying gives you the Reader representation

By currying:

$$
R\times A\to B
$$

is isomorphic to:

$$
A\to(R\to B)
$$

So we can write:

$$
loadUser :
UserId
\to
(R\to User)
$$

Or equivalently:

$$
loadUser :
R\to(UserId\to User)
$$

This is the **Reader structure**.

In Rust, you can simply represent this as:

```rust
fn load_user<'a>(
    id: UserId,
) -> impl FnOnce(&'a Env) -> User {
    move |env| {
        env.storage.load_user(id)
    }
}
```

Although in normal Rust code I'd often write the more direct form:

```rust
fn load_user(
    env: &Env,
    id: UserId,
) -> User {
    env.storage.load_user(id)
}
```

The important thing is the **type**, not whether you literally wrap it in a closure.

---

# 11. Add errors

Now combine Reader and Result.

A capability-dependent operation that can fail is:

$$
R\times A
\to
B+E
$$

or, after currying:

$$
A
\to
(R\to(B+E))
$$

So:

$$
\boxed{
A\to Reader_R(Result_B)
}
$$

or informally:

$$
\boxed{
A\to(R\to Result(B,E))
}
$$

For your architecture:

```rust
fn load_conversation<'a>(
    id: ConversationId,
) -> impl FnOnce(&'a Env) -> Result<Conversation, AppError>
```

This says precisely:

> `load_conversation` does not own the storage. It describes a computation which, when supplied with an environment containing the necessary capability, may produce either a `Conversation` or an `AppError`.

That's much more precise than simply saying "dependency injection."

---

# 12. Capability vs effect

This gives you a very useful distinction.

Suppose:

```rust
struct Env<'a> {
    storage: &'a dyn Storage,
    model: &'a dyn ModelClient,
    events: &'a dyn EventSink,
}
```

Then:

$$
R = Env
$$

is the **capability environment**.

The operation:

$$
A\to(R\to B+E)
$$

says:

$$
\text{Input}
\to
\text{requires capability}
\to
\text{produces Result}
$$

So:

### Capability

$$
R
$$

answers:

> What am I allowed/able to access?

### Effect/result

$$
B+E
$$

answers:

> What can this computation produce?

These should not be conflated.

For example:

```rust
Storage
```

is a capability.

```rust
Result<User, DatabaseError>
```

describes an outcome.

---

# 13. This is particularly relevant to your event bus

Suppose your reducer is:

$$
reduce :
State\times Input
\to
State\times Event^*
$$

It doesn't need an event bus.

It **describes events**.

```rust
fn reduce(
    state: &AgentState,
    input: AgentInput,
) -> Transition {
    ...
}

struct Transition {
    state: AgentState,
    events: Vec<AgentEvent>,
}
```

Then the runtime can have:

```rust
struct Env<'a> {
    event_sink: &'a dyn EventSink,
    storage: &'a dyn Storage,
    model: &'a dyn ModelClient,
}
```

The interpreter takes:

$$
Event^*
\to
IO
$$

Conceptually:

$$
interpret :
R\times Event^*
\to
Effect
$$

This gives you a clean separation:

$$
\boxed{
\text{Domain}
:
State\times Input
\to
State\times Events
}
$$

versus:

$$
\boxed{
\text{Runtime}
:
Capabilities\times Events
\to
Effects
}
$$

That is very close to the **functional core / imperative shell** pattern.

---

# 14. `ControlFlow` is a different kind of coproduct

Now to:

```rust
std::ops::ControlFlow
```

Rust defines essentially:

```rust
enum ControlFlow<B, C> {
    Continue(C),
    Break(B),
}
```

Algebraically:

$$
ControlFlow(B,C)
\cong
B+C
$$

Notice that this looks almost identical to:

$$
Result(C,B)
$$

Indeed:

```text
Result<T, E>
    Ok(T)
    Err(E)

ControlFlow<B, C>
    Continue(C)
    Break(B)
```

The **algebraic shape is almost the same**.

The semantic interpretation differs.

---

# 15. `Result` vs `ControlFlow`

`Result` means:

$$
Success + Failure
$$

while `ControlFlow` means:

$$
Continue + EarlyExit
$$

For example, imagine searching a collection.

```rust
use std::ops::ControlFlow;

let result = (0..100).try_for_each(|x| {
    if x == 42 {
        ControlFlow::Break(x)
    } else {
        ControlFlow::Continue(())
    }
});
```

The meaning is:

$$
Continue(())
$$

> Keep processing.

or:

$$
Break(42)
$$

> Stop immediately and return `42`.

It isn't necessarily an error.

---

# 16. Why is `ControlFlow` useful?

It generalizes algorithms that need **short-circuiting**.

For example:

$$
fold :
State\times A
\to
ControlFlow(B,State)
$$

Each iteration can say:

```text
Continue(new_state)
```

or:

```text
Break(result)
```

This is useful for:

* tree traversal
* parsers
* visitors
* searching
* graph algorithms
* recursive interpreters
* early termination
* validation that stops at the first decisive result

A particularly nice example is a tree traversal:

```rust
fn visit(node: &Node) -> ControlFlow<Found, ()> {
    if matches_target(node) {
        return ControlFlow::Break(Found(node.id));
    }

    for child in &node.children {
        visit(child)?;
    }

    ControlFlow::Continue(())
}
```

The `?` operator works with `ControlFlow` too, propagating `Break`.

---

# 17. `Result` and `ControlFlow` are both coproducts

This is worth internalizing.

$$
Result(A,E)
\cong
A+E
$$

and:

$$
ControlFlow(B,A)
\cong
B+A
$$

The difference isn't the underlying set construction.

It is the **semantic interpretation** of the two injections.

For `Result`:

$$
A \hookrightarrow A+E
$$

means success.

$$
E \hookrightarrow A+E
$$

means failure.

For `ControlFlow`:

$$
A \hookrightarrow A+B
$$

means continue.

$$
B \hookrightarrow A+B
$$

means terminate.

Same categorical shape; different algebraic meaning.

---

# 18. Capability-dependent operation: the most general form

Now we can combine everything.

An ordinary pure operation:

$$
f:A\to B
$$

A capability-dependent operation:

$$
f:R\times A\to B
$$

A fallible capability-dependent operation:

$$
f:R\times A\to B+E
$$

An asynchronous fallible capability-dependent operation:

$$
f:R\times A\to Async(B+E)
$$

A capability-dependent operation that also emits events:

$$
f:R\times A
\to
(B+E)\times Event^*
$$

or, depending on your desired semantics:

$$
f:R\times A
\to
Async((B+E)\times Event^*)
$$

This gives you a very useful **effect signature**.

For example:

```rust
async fn execute_tool<'a>(
    env: &'a Env<'a>,
    request: ToolRequest,
) -> Result<(ToolResult, Vec<AgentEvent>), ToolError>
```

Its conceptual type is:

$$
Env\times ToolRequest
\to
Async((ToolResult+ToolError)\times Event^*)
$$

Or curried:

$$
ToolRequest
\to
Env
\to
Async((ToolResult+ToolError)\times Event^*)
$$

Now you can see exactly what the function depends on and what it can do.

---

# 19. The important architectural payoff

You can classify your functions by progressively adding structure:

$$
A\to B
$$

**pure computation**

↓

$$
A\to(R\to B)
$$

**capability-dependent computation / Reader**

↓

$$
A\to(R\to(B+E))
$$

**capability-dependent + fallible**

↓

$$
A\to(R\to Async(B+E))
$$

**capability-dependent + fallible + asynchronous**

↓

$$
A\to(R\to Async((B+E)\times W))
$$

**capability-dependent + fallible + asynchronous + output accumulation**

where `W` might be:

```rust
Vec<AgentEvent>
```

This is essentially an **effect signature**.

And it gives you a principled answer to your earlier question about dependency inversion:

> **Dependency inversion says which capabilities cross the boundary; monadic/effect structure says what computational behavior happens once those capabilities are available.**

So for your runtime I'd think in terms of:

$$
\boxed{
\text{Domain}
\subset
\text{Capability-dependent application}
\subset
\text{Effect interpreter}
}
$$

with the domain preferably staying at the leftmost, most algebraically constrained end.
