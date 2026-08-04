**Intent: build a high-ROI taxonomy of idiomatic Rust patterns, especially the ones that exploit Rust’s type system and ownership model.**

The most useful way to understand idiomatic Rust is not as a list of design patterns like GoF. It is more like:

$$
\text{make invalid states unrepresentable}
\quad+\quad
\text{make ownership explicit}
\quad+\quad
\text{compose transformations}
$$

Here are the patterns I would prioritize.

---

# 1. Newtype pattern — semantic types

Instead of:

```rust
type UserId = u64;
type OrderId = u64;
```

use:

```rust
struct UserId(u64);
struct OrderId(u64);
```

Now:

$$
u64 \to UserId
$$

is a semantic lift.

```rust
fn get_user(id: UserId) {}
fn get_order(id: OrderId) {}
```

You cannot accidentally pass an `OrderId` where a `UserId` is required.

**Use for:**

* IDs
* units
* validated values
* security capabilities
* domain concepts
* preventing argument swaps

Example:

```rust
struct Meters(f64);
struct Seconds(f64);
```

Now the type system prevents:

```rust
speed = meters / seconds;
```

from accidentally becoming:

```rust
speed = seconds / meters;
```

---

# 2. Make invalid states unrepresentable

Instead of:

```rust
struct User {
    verified: bool,
    email: String,
}
```

you might have:

```rust
struct UnverifiedUser {
    email: String,
}

struct VerifiedUser {
    email: String,
}
```

Then:

```rust
fn send_sensitive_email(user: VerifiedUser) {}
```

The function cannot even receive an unverified user.

The state space changes from:

$$
\text{User}
===========

\text{Email}
\times
{true,false}
$$

to:

$$
\text{User}
===========

\text{UnverifiedUser}
+
\text{VerifiedUser}
$$

This is a **sum-type encoding of state**.

This pattern is everywhere in high-quality Rust.

---

# 3. Typestate

The more general form:

```rust
struct Connection<S> {
    socket: TcpStream,
    _state: PhantomData<S>,
}
```

Then:

```rust
struct Disconnected;
struct Connected;
```

```rust
impl Connection<Disconnected> {
    fn connect(self) -> Connection<Connected> {
        // ...
    }
}
```

```rust
impl Connection<Connected> {
    fn send(&self, data: &[u8]) {
        // ...
    }
}
```

The transition is:

$$
Connection(Disconnected)
\to
Connection(Connected)
$$

The type system encodes the protocol.

**Use for:**

* database transactions
* network protocols
* authenticated sessions
* parsers
* workflow engines
* agents

---

# 4. RAII / Drop guards

Rust's most important systems pattern:

```rust
struct LockGuard<'a> {
    lock: &'a Lock,
}

impl Drop for LockGuard<'_> {
    fn drop(&mut self) {
        self.lock.unlock();
    }
}
```

Then:

```rust
let guard = lock.acquire();
// critical section
// automatically unlocks here
```

The lifetime of a resource becomes:

$$
\text{Acquire}
\to
\text{Guard exists}
\to
\text{Guard dropped}
\to
\text{Release}
$$

This generalizes to:

* locks
* transactions
* tracing spans
* temporary directories
* file handles
* database connections
* resource cleanup

The powerful principle is:

> **Resource lifetime = lexical scope.**

---

# 5. Borrowing as capability passing

A borrow is not merely “a reference.”

Conceptually:

```rust
fn read(x: &T)
```

means:

$$
\text{caller gives temporary read capability}
$$

while:

```rust
fn mutate(x: &mut T)
```

means:

$$
\text{caller gives exclusive mutation capability}
$$

The core ownership rule is:

$$
\text{many readers}
\oplus
\text{one writer}
$$

but never both simultaneously.

This is essentially a static capability discipline.

---

# 6. `Option` as absence, not sentinel values

Bad:

```rust
fn find_user(id: Id) -> User {
    User {
        id: 0, // means "not found"
    }
}
```

Idiomatic:

```rust
fn find_user(id: Id) -> Option<User>
```

The semantic domain becomes:

$$
User + 1
$$

That is, either:

$$
Some(User)
$$

or:

$$
None
$$

This is much better than encoding absence inside the data domain.

---

# 7. `Result` as typed control flow

Instead of:

```rust
fn process() -> bool
```

where `false` could mean anything:

```rust
fn process() -> Result<Output, Error>
```

Then:

```rust
let output = load()?
    .validate()?
    .transform()?;
```

The `?` operator is essentially a short-circuiting composition:

$$
A
\to
Result(B,E)
\to
Result(C,E)
\to
Result(D,E)
$$

The computation stops at the first failure.

This is probably the most important idiom in application Rust.

---

# 8. Iterator pipelines

Instead of:

```rust
let mut output = Vec::new();

for x in values {
    if x > 10 {
        output.push(x * 2);
    }
}
```

use:

```rust
let output: Vec<_> = values
    .into_iter()
    .filter(|x| *x > 10)
    .map(|x| x * 2)
    .collect();
```

The abstraction is:

$$
Iterator(A)
\xrightarrow{filter}
Iterator(A)
\xrightarrow{map}
Iterator(B)
\xrightarrow{collect}
Collection(B)
$$

This is one of Rust's most idiomatic examples of compositional programming.

---

# 9. `From` / `Into`: canonical conversion graphs

Instead of manually constructing conversions everywhere:

```rust
impl From<RawConfig> for ValidatedConfig {
    fn from(raw: RawConfig) -> Self {
        // ...
    }
}
```

Then:

```rust
let config: ValidatedConfig = raw.into();
```

The conceptual graph is:

$$
RawConfig
\to
ValidatedConfig
\to
RuntimeConfig
$$

You can model your program as a pipeline of representations:

```text
Raw
  ↓
Parsed
  ↓
Validated
  ↓
Normalized
  ↓
Executable
```

This is especially powerful for:

* compilers
* parsers
* configuration
* API boundaries
* protocol handling

Each type represents a different semantic phase.

---

# 10. Parse, don't validate

This is one of the highest-value Rust patterns.

Instead of:

```rust
struct Email(String);

fn is_valid(email: &Email) -> bool {
    ...
}
```

you make invalid values impossible:

```rust
struct ValidEmail(String);

impl ValidEmail {
    fn parse(s: String) -> Result<Self, EmailError> {
        if valid(&s) {
            Ok(Self(s))
        } else {
            Err(EmailError)
        }
    }
}
```

Then:

```rust
fn send_email(email: ValidEmail) {}
```

The function does not need to validate again.

Formally:

$$
Raw
\xrightarrow{\text{parse}}
Result(Valid,E)
$$

After construction:

$$
Valid
\Rightarrow
\text{invariant holds}
$$

This is a form of **proof-carrying data**.

---

# 11. Builder pattern

For complex construction:

```rust
let app = AppBuilder::new()
    .port(8080)
    .workers(4)
    .database(db)
    .build()?;
```

But the Rust version is often stronger than the classic Builder pattern.

You can use typestate:

```rust
Builder<MissingPort>
    .port(8080)
    .build()
```

where `build()` only exists after required fields are supplied.

So:

$$
Builder_{incomplete}
\to
Builder_{complete}
\to
Product
$$

---

# 12. Trait-based static polymorphism

Instead of:

```rust
Box<dyn Storage>
```

you can write:

```rust
fn save<S: Storage>(storage: S) {
    ...
}
```

The distinction is:

### Dynamic dispatch

$$
\exists S.\ S : Storage
$$

represented by:

```rust
dyn Storage
```

### Static dispatch

$$
\forall S.\ S : Storage
$$

represented by:

```rust
S: Storage
```

Use generics when you want:

* zero-cost abstraction
* compile-time specialization
* monomorphization

Use `dyn Trait` when you want:

* runtime heterogeneity
* plugin architecture
* stable object identity

---

# 13. Trait objects as existential types

This:

```rust
Box<dyn Storage>
```

can be thought of as:

$$
\exists S.\ Storage(S)
$$

You know there exists some concrete implementation, but you intentionally hide which one.

This is useful for:

```rust
Vec<Box<dyn Handler>>
```

where:

```rust
HandlerA
HandlerB
HandlerC
```

can all coexist.

The tradeoff is:

$$
\text{runtime flexibility}
\leftrightarrow
\text{compile-time specialization}
$$

---

# 14. `Arc<Mutex<T>>` — but only when the ownership model demands it

The classic:

```rust
Arc<Mutex<T>>
```

means:

$$
\text{shared ownership}
+
\text{exclusive mutation}
$$

But a very important idiom is:

> Do not start with `Arc<Mutex<T>>`. Start by asking who owns the data.

Often better:

```rust
struct Actor {
    state: State,
    rx: Receiver<Message>,
}
```

and only the actor owns the state.

Then other tasks send messages:

$$
\text{Message}
\to
\text{Owner}
\to
\text{State transition}
$$

This is the **actor pattern**.

For many concurrent systems, this is easier to reason about than shared mutable state.

---

# 15. Message passing / actor pattern

Instead of:

$$
T
\to
Arc<Mutex<T>>
$$

use:

$$
\text{Actor}
+
\text{Mailbox}
$$

```rust
enum Command {
    Deposit(u64),
    Withdraw(u64),
}
```

```rust
struct AccountActor {
    balance: u64,
    rx: Receiver<Command>,
}
```

All mutation is serialized through the actor.

This gives a transition function:

$$
State \times Command
\to
State
$$

or:

$$
\delta : S \times E \to S
$$

This is an extremely natural fit for Rust.

---

# 16. Interior mutability

Rust separates:

```rust
&T
```

from:

```rust
&mut T
```

But sometimes an object needs mutation behind a shared reference.

Then:

```rust
Cell<T>
RefCell<T>
Mutex<T>
RwLock<T>
AtomicT
```

are different implementations of:

$$
\text{shared reference}
\to
\text{controlled mutation}
$$

The important question is **where the invariant is checked**:

| Type         | Invariant checked                    |
| ------------ | ------------------------------------ |
| `Cell<T>`    | compile-time-ish API restrictions    |
| `RefCell<T>` | runtime                              |
| `Mutex<T>`   | runtime + thread synchronization     |
| `RwLock<T>`  | runtime + read/write synchronization |
| `Atomic<T>`  | hardware atomicity                   |

This is a very useful hierarchy.

---

# 17. `enum` as a state machine

Rust's enums are algebraic data types.

```rust
enum Connection {
    Disconnected,
    Connecting,
    Connected(Socket),
    Failed(Error),
}
```

Mathematically:

$$
Connection
==========

1
+
1
+
Socket
+
Error
$$

Then:

```rust
match connection {
    Connection::Disconnected => ...
    Connection::Connecting => ...
    Connection::Connected(socket) => ...
    Connection::Failed(error) => ...
}
```

The compiler checks exhaustiveness.

This is perhaps the most powerful Rust pattern for modeling domain state.

---

# 18. Exhaustive matching as proof elimination

When you write:

```rust
match state {
    State::A => ...
    State::B => ...
}
```

you are effectively proving:

$$
State = A \lor B
$$

The compiler ensures you handled both cases.

This is one reason Rust code can be much more robust than code using:

```python
if state == "A":
```

because the set of possible states is explicit.

---

# 19. Phantom types

Sometimes a type parameter carries compile-time information but no runtime data:

```rust
struct Id<T> {
    value: u64,
    _marker: PhantomData<T>,
}
```

Then:

```rust
struct User;
struct Order;

type UserId = Id<User>;
type OrderId = Id<Order>;
```

Runtime representation:

$$
UserId \cong u64
$$

but type-level semantics:

$$
UserId \not\cong OrderId
$$

This is extremely powerful for:

* units
* state machines
* security
* protocol phases
* ownership markers

---

# 20. Zero-sized types as compile-time witnesses

```rust
struct Authenticated;
struct Unauthenticated;
```

These types contain no data.

But:

```rust
Session<Authenticated>
```

is different from:

```rust
Session<Unauthenticated>
```

The type itself is a **witness** that some property holds.

This gives:

$$
\text{Runtime operation}
\to
\text{Compile-time proof obligation}
$$

Very powerful.

---

# 21. The "sealed trait" pattern

You may want users to use a trait but not implement it themselves:

```rust
mod private {
    pub trait Sealed {}
}

pub trait MyTrait: private::Sealed {
    ...
}
```

This lets you control the closed world of implementations.

Useful when:

* exhaustiveness matters
* you need future compatibility
* only your types should implement the trait

This is a form of **controlled algebraic extensibility**.

---

# 22. `Drop` as linear-resource management

Rust's ownership system is close to linear logic:

$$
\text{resource}
\to
\text{owned exactly once}
$$

A value is consumed:

```rust
fn consume(x: Resource) {}
```

After:

```rust
consume(resource);
```

the original resource cannot be used.

This makes APIs express resource transitions:

$$
Resource
\to
Consumed
$$

You cannot represent the consumed state as the same usable value.

This is especially powerful for:

* transactions
* file handles
* sockets
* locks
* cryptographic keys
* capabilities

---

# The patterns I would prioritize for you

Given your interest in **category theory, algebra, state machines, distributed systems, and AI infrastructure**, I would study Rust in this order:

### Tier 1 — essential

$$
\boxed{
\begin{aligned}
&enum + match\
&Option / Result\
&ownership + borrowing\
&newtypes\
&From / Into\
&iterators\
&Drop / RAII
\end{aligned}
}
$$

### Tier 2 — advanced systems design

$$
\boxed{
\begin{aligned}
&\text{typestate}\
&\text{phantom types}\
&\text{capability types}\
&\text{actor model}\
&\text{interior mutability}\
&\text{trait objects vs generics}\
&\text{zero-copy design}
\end{aligned}
}
$$

### Tier 3 — frontier

$$
\boxed{
\begin{aligned}
&\text{proof-carrying data}\
&\text{typed workflows}\
&\text{capability-secure agents}\
&\text{compile-time protocol verification}\
&\text{effect-like abstractions}\
&\text{distributed state machines}
\end{aligned}
}
$$

The central Rust design question is:

> **What states, capabilities, ownership relationships, and invariants can I move from runtime into the type structure?**

That question is the common thread connecting `Result`, `enum`, newtypes, typestate, lifetimes, traits, `Drop`, and Rust's ownership model.
