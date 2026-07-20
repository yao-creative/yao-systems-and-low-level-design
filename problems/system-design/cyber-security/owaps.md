**Intent: functional decomposition of a security taxonomy.**

The most useful way to understand the OWASP Top 10 is **not as ten isolated vulnerabilities**, but as ten ways the function

$$
\text{untrusted external interaction}
\longrightarrow
\text{application state transition}
$$

can fail.

A web application is essentially a function:

$$
F(\text{request},\text{identity},\text{current state},\text{configuration},\text{dependencies})
\longrightarrow
(\text{response},\text{state change},\text{side effects})
$$

The OWASP Top 10 asks:

> **Where can an attacker cause this function to produce an outcome that the system's security policy should have forbidden?**

The 2021 list contains these ten categories. ([OWASP Foundation][1])

---

## 1. Broken Access Control — **the authorization function is wrong**

### Functional model

```text
request
  ↓
authenticate: "Who are you?"
  ↓
authorize: "May you perform this operation on this resource?"
  ↓
operation
```

The security invariant should be:

$$
\operatorname{allowed}(u, a, r)
\Rightarrow
\operatorname{perform}(u,a,r)
$$

But the application accidentally implements:

$$
\operatorname{perform}(u,a,r)
$$

without correctly checking:

$$
\operatorname{allowed}(u,a,r)
$$

### Example

```http
GET /users/123/profile
```

User 123 can access their profile.

But user 456 changes the URL:

```http
GET /users/123/profile
```

and receives user 123's data.

The bug is not primarily "the URL is guessable."

The bug is:

$$
\operatorname{authorize}(456,\text{read},123) = \text{true}
$$

when it should be false.

**Functional category: missing or incorrect precondition.**

This is analogous to a function whose type says:

```text
read_user(user_id) -> User
```

when its real type should be closer to:

```text
read_user(actor, user_id)
  -> User
  only if authorize(actor, READ, user_id)
```

---

# 2. Cryptographic Failures — **the confidentiality/integrity function is inadequate**

You have sensitive data:

$$
D
$$

and want:

$$
\operatorname{protect}(D)
$$

against unauthorized observers or modification.

A secure system needs properties such as:

$$
\operatorname{decrypt}(\operatorname{encrypt}(D,k), k) = D
$$

while an attacker without $k$ should not feasibly recover $D$.

Functional failures include:

* storing passwords as plaintext
* using weak hashing
* transmitting sensitive data without TLS
* using encryption incorrectly
* exposing secrets in logs

For example:

```text
password
    ↓
SHA-1(password)
    ↓
database
```

is functionally wrong for password storage because the attacker can efficiently search the input space.

A proper password storage function is more like:

$$
\operatorname{store}(p)
=======================

\operatorname{KDF}(p,\text{salt},\text{cost})
$$

where the computation is intentionally expensive.

**Functional category: the protection transformation does not provide the required security property.**

---

# 3. Injection — **data is accidentally interpreted as program**

This is one of the most important categories.

You intend:

$$
\operatorname{execute}(\text{program}, \text{data})
$$

but accidentally construct:

$$
\operatorname{execute}(\text{program} + \text{data})
$$

where the data modifies the program.

Example:

```python
query = "SELECT * FROM users WHERE name = '" + user_input + "'"
```

The intended function is:

$$
Q : \text{String} \to \text{QueryResult}
$$

But the implementation has collapsed two semantic domains:

$$
\text{Data} \cong \text{ProgramText}
$$

So the attacker supplies input that changes the query.

Parameterized queries preserve the distinction:

```text
SQL program: SELECT ... WHERE name = ?
Data:        attacker input
```

Conceptually:

$$
\operatorname{execute}(P, d)
$$

rather than:

$$
\operatorname{execute}(P \mathbin{+!!+} d)
$$

This is the same fundamental class as:

* SQL injection
* command injection
* XSS
* template injection

**Functional category: failure to preserve the boundary between code and data.**

---

# 4. Insecure Design — **the security specification is missing**

This is more fundamental than a coding bug.

Suppose you implement:

```text
transfer_money(from, to, amount)
```

The code works perfectly.

But you never designed:

```text
maximum_daily_transfer(user)
```

or:

```text
require_2fa_for_large_transfer
```

Then the implementation may be bug-free while the system is insecure.

The problem is that the desired behavior was never specified.

Formally:

$$
\text{Implementation}
\models
\text{specified requirements}
$$

but:

$$
\text{specified requirements}
\not\models
\text{actual security policy}
$$

Example:

```text
User can submit unlimited password reset requests.
```

There may be no coding mistake.

The system simply lacks a rate-limiting security requirement.

**Functional category: the desired security invariant was never encoded into the system design.**

This is analogous to a distributed-systems bug where the system has no defined consistency model.

You cannot correctly implement a requirement that does not exist.

---

# 5. Security Misconfiguration — **the implementation/configuration has an unsafe parameter**

The design might be correct:

$$
\text{only admins} \to \text{admin panel}
$$

but the deployment accidentally has:

```text
DEBUG = true
```

or:

```text
default_password = "admin"
```

or:

```text
CORS = "*"
```

or:

```text
unnecessary_service = enabled
```

The system's behavior is a function of both code and configuration:

$$
F(\text{code},\text{configuration})
$$

Security misconfiguration means:

$$
\text{code is acceptable}
$$

but:

$$
\text{configuration} \notin \text{secure configuration space}
$$

For example:

$$
C_{\text{production}}
=====================

{\text{TLS on},\text{debug off},\text{least privilege},\ldots}
$$

but the actual deployment uses:

$$
C_{\text{actual}}
\notin C_{\text{production}}
$$

**Functional category: the system is instantiated with unsafe parameters.**

---

# 6. Vulnerable and Outdated Components — **a dependency function already has a known bad behavior**

Your application is a composition:

$$
F = f_n \circ f_{n-1} \circ \cdots \circ f_2 \circ f_1
$$

Suppose:

```text
your application
    ↓
framework
    ↓
library
    ↓
parser
```

If one component has:

$$
f_i : x \mapsto \text{unsafe behavior}
$$

then the composition may be insecure even if your own code is correct.

Example:

```text
your code
  ↓
old image parser
  ↓
known remote-code-execution vulnerability
```

The security of the whole system is bounded by the security of its dependencies:

$$
\operatorname{Security}(F)
\leq
\min_i \operatorname{Security}(f_i)
$$

This is not mathematically literally a numeric minimum, but it is a useful engineering model.

**Functional category: dependency composition imports an unsafe implementation.**

---

# 7. Identification and Authentication Failures — **the identity function is incorrect**

Authentication is a function:

$$
\operatorname{authenticate} :
\text{Credentials}
\to
\text{Principal}
$$

The desired property is:

$$
\operatorname{authenticate}(c)
==============================

u
\iff
c \text{ proves identity of } u
$$

Failures include:

```text
weak passwords
```

```text
session tokens that never expire
```

```text
session fixation
```

```text
poor password reset
```

```text
credential stuffing not mitigated
```

The fundamental problem is:

$$
\operatorname{authenticate}(x)
==============================

u
$$

when $x$ should not be sufficient evidence that the actor is $u$.

Notice the distinction:

### Authentication

> Who are you?

$$
\operatorname{identify}(a) = u
$$

### Authorization

> What may you do?

$$
\operatorname{authorize}(u,a,r)
$$

A system can have:

```text
correct authentication
```

but:

```text
broken authorization
```

For example:

```text
I correctly know that you are Alice.
```

but:

```text
I incorrectly allow Alice to access Bob's account.
```

---

# 8. Software and Data Integrity Failures — **the system trusts data/code without verifying provenance**

The system receives an artifact:

$$
x
$$

and assumes:

$$
\operatorname{trusted}(x)
$$

without verifying that:

$$
x
\text{ came from an authorized source}
$$

or:

$$
x
\text{ has not been modified}
$$

Examples:

```text
download package
      ↓
execute package
```

without verifying its integrity.

A secure pipeline is:

$$
x
\overset{\text{signature/hash/provenance}}{\longrightarrow}
\operatorname{verified}(x)
\longrightarrow
\operatorname{use}(x)
$$

This is especially important for:

* software updates
* CI/CD pipelines
* packages
* serialized data
* model artifacts
* configuration
* supply chains

**Functional category: the system accepts an artifact without establishing its authenticity or integrity.**

---

# 9. Security Logging and Monitoring Failures — **the observation function is incomplete**

A production system has two kinds of behavior:

$$
\text{primary computation}
$$

and:

$$
\text{observation}
$$

For example:

```text
request
  ↓
authorization
  ↓
database mutation
```

should often also produce:

```text
audit event
```

The observation function is:

$$
O :
\text{system events}
\to
\text{logs/metrics/alerts}
$$

A security monitoring failure means:

$$
O(e) = \varnothing
$$

for an important security event $e$.

Example:

```text
attacker logs in
attacker accesses 10,000 records
attacker deletes audit logs
```

If no useful event is recorded, then:

```text
incident occurs
    ↓
no detection
    ↓
no investigation
    ↓
no response
```

This category is about **detectability**, not merely prevention.

**Functional category: the system's observability function does not expose security-relevant state transitions.**

---

# 10. SSRF — **user-controlled data reaches a privileged network function**

This is a very nice example of **capability confusion**.

Suppose the application has:

```text
fetch_url(url)
```

The server can access:

```text
Internet
internal network
cloud metadata service
localhost
```

The user controls:

```text
url
```

Therefore:

$$
\text{untrusted user}
\to
\text{server network capability}
$$

The application accidentally exposes:

$$
\operatorname{fetch}_{\text{server}} :
\text{URL}
\to
\text{NetworkResponse}
$$

to an untrusted caller.

The intended function might have been:

$$
\operatorname{fetch}_{\text{approved_external_resource}}
$$

but the actual function is:

$$
\operatorname{fetch}_{\text{anything_the_server_can_reach}}
$$

**Functional category: an untrusted input is allowed to select a resource using a more privileged capability than it should possess.**

---

# The deeper functional classification

I would compress the Top 10 into five fundamental failure modes:

| Fundamental failure               | OWASP categories                                      |
| --------------------------------- | ----------------------------------------------------- |
| **Wrong permission predicate**    | A01 Access Control                                    |
| **Bad transformation / boundary** | A02 Cryptography, A03 Injection                       |
| **Missing security invariant**    | A04 Insecure Design                                   |
| **Unsafe system composition**     | A05 Misconfiguration, A06 Dependencies, A08 Integrity |
| **Wrong identity or provenance**  | A07 Authentication, A08 Integrity                     |
| **Insufficient observation**      | A09 Logging                                           |
| **Capability leak**               | A10 SSRF                                              |

The common structure is:

$$
\boxed{
\text{Attacker controls some input}
\to
\text{input reaches a powerful operation}
\to
\text{security invariant is violated}
}
$$

For example:

### Access control

$$
\text{attacker}
\to
\text{resource identifier}
\to
\text{database read}
\to
\text{unauthorized data}
$$

### Injection

$$
\text{attacker}
\to
\text{program/data boundary}
\to
\text{interpreter}
\to
\text{attacker-controlled computation}
$$

### SSRF

$$
\text{attacker}
\to
\text{URL}
\to
\text{server network capability}
\to
\text{privileged network access}
$$

### Authentication failure

$$
\text{attacker}
\to
\text{credential/session input}
\to
\text{identity function}
\to
\text{wrong principal}
$$

### Misconfiguration

$$
\text{deployment}
\to
\text{unsafe parameter}
\to
\text{otherwise-correct system}
\to
\text{unsafe behavior}
$$

---

## The most useful mental model for SWE

When designing a function, ask:

$$
\text{Input}
\longrightarrow
\text{Parser}
\longrightarrow
\text{Authorization}
\longrightarrow
\text{Business Logic}
\longrightarrow
\text{Side Effect}
$$

Then ask four questions:

1. **Who controls the input?**
2. **What capability does the input eventually reach?**
3. **What invariant is supposed to prevent abuse?**
4. **Can I detect when the invariant is violated?**

That gives you a practical security review:

```text
Input
  ↓
Can attacker control it?
  ↓
What does it select?
  ↓
What capability executes?
  ↓
What authorization/provenance check exists?
  ↓
What happens if it fails?
  ↓
Can the failure be detected?
```

So, from a software-engineering perspective, the OWASP Top 10 is fundamentally a catalogue of **bad compositions between untrusted data, authority, state transitions, and interpreters**. The list is an awareness taxonomy rather than a complete security specification; for systematic verification, OWASP itself points developers toward more comprehensive application-security verification practices. ([OWASP Foundation][1])

[1]: https://owasp.org/Top10/2021/?utm_source=chatgpt.com "OWASP Top 10:2021"
