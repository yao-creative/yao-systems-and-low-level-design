The general pattern is:

> Take a morphism
>
> [
> f : X \to T(Y)
> ]
>
> and construct another morphism of the same type
>
> [
> \hat f : X \to T(Y)
> ]
>
> with additional behavior.

This family of patterns is extremely common.

---

# 1. Decorator Pattern

The classical OO answer.

```text
f(x)
```

becomes:

```text
retry(f)(x)
```

or

```text
logging(f)(x)
```

or

```text
metrics(f)(x)
```

Example:

```text
PaymentService
```

wrapped as:

```text
RetryingPaymentService
LoggingPaymentService
CachingPaymentService
```

Category theoretically:

[
Decorator : Hom(X,Y)\to Hom(X,Y)
]

A morphism transformer.

Retry is almost always implemented this way.

---

# 2. Middleware / Interceptor Chain

Instead of one wrapper:

```text
Request
   |
Logging
   |
Retry
   |
CircuitBreaker
   |
Handler
```

Composition:

[
f'
==

Logging
\circ Retry
\circ CircuitBreaker
\circ f
]

Examples:

* HTTP middleware
* RPC interceptors
* gRPC interceptors

This is probably the most common modern implementation.

---

# 3. Proxy Pattern

The wrapped object presents the same interface.

```text
Client
   |
Proxy
   |
Real Service
```

Example:

```text
UserRepository
```

becomes:

```text
RetryingUserRepository
```

or:

```text
CachingUserRepository
```

The proxy controls invocation.

---

# 4. Policy Objects

Instead of embedding retry behavior:

```text
retry(f)
```

you pass strategy explicitly:

```text
execute(
    retry_policy,
    timeout_policy,
    circuit_breaker_policy,
    f
)
```

Examples:

```text
ExponentialBackoff
LinearBackoff
NoRetry
```

This separates:

* operation
* operational semantics

Very common in cloud SDKs.

---

# 5. Higher-Order Functions

Functional version of decorator.

```haskell
retry :: IO a -> IO a
```

Examples:

```text
retry(fetchUser)
timeout(fetchUser)
trace(fetchUser)
```

These compose naturally:

```text
trace(retry(timeout(fetchUser)))
```

This is mathematically very clean.

---

# 6. Monad Transformers

Functional programming's answer when effects stack.

Instead of:

```text
IO<Result<User>>
```

you might have:

```text
RetryT (ReaderT Config IO) User
```

or:

```text
StateT Cache (ExceptT Error IO)
```

Retry becomes part of the effect stack.

This is much more algebraic than decorators.

---

# 7. Aspect-Oriented Programming

The old enterprise Java solution.

Declare:

```text
@Transactional
@Retryable
@Timed
```

Runtime injects behavior.

Conceptually:

```text
retry ∘ transaction ∘ metrics ∘ f
```

without explicit composition in code.

---

# 8. Command Pattern

Represent operation as a value.

Instead of:

```text
charge()
```

store:

```text
ChargeCommand(order123)
```

Wrapper executes:

```text
RetryExecutor.execute(command)
```

Useful because commands can be:

* serialized
* queued
* retried
* persisted

Background job systems often do this.

---

# 9. Workflow/Saga Wrappers

Long-running operations are wrapped in orchestration logic.

```text
ReserveInventory
ChargeCard
ShipOrder
```

becomes:

```text
Retry(ReserveInventory)
Retry(ChargeCard)
Retry(ShipOrder)
```

plus compensation.

The workflow engine owns the wrapper.

---

# Which pattern dominates in practice?

| Layer                 | Typical pattern        |
| --------------------- | ---------------------- |
| HTTP                  | middleware             |
| Database access       | proxy/decorator        |
| Functional code       | higher-order functions |
| Background jobs       | command + executor     |
| Distributed workflows | saga/workflow engine   |
| Enterprise frameworks | AOP                    |

---

# Category theoretic view

All of these are variants of:

[
W : Hom(X,Y) \to Hom(X,Y)
]

where (W) is:

* Retry
* Cache
* Trace
* Metrics
* Authorization
* Transaction
* Circuit breaker

These are **endomorphisms on hom-sets**.

If composition is preserved:

[
W(g \circ f)
============

W(g)\circ W(f)
]

then (W) behaves like an endofunctor on a category of computations.

---

A very common stack in production looks like:

```text
Metrics(
    Trace(
        CircuitBreaker(
            Retry(
                Timeout(
                    ActualOperation
                )
            )
        )
    )
)
```

which categorically resembles composition of endofunctors:

[
Metrics
\circ
Trace
\circ
CircuitBreaker
\circ
Retry
\circ
Timeout
]

acting on the underlying computation morphism.

This perspective is one reason middleware pipelines, interceptors, and functional combinators feel so similar: they are all implementations of "morphism transformers" applied to the same underlying arrow.
