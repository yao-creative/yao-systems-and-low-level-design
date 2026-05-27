I’ll ground each concept in the same small domain so the differences become structural rather than verbal.

We’ll use a **notification system** because it naturally exposes all four ideas.

---

# 1. Abstraction — define the contract

### Intent: *“What must a notifier do?”*

You define a contract without specifying how it works.

```python
from abc import ABC, abstractmethod

class NotificationSender(ABC):
    @abstractmethod
    def send(self, message: str) -> None:
        pass
```

### What’s happening structurally

* You’ve removed implementation detail
* You’ve created a **reasoning boundary**
* Anything that implements this is “valid”

This is not behavior yet—it’s a **shape constraint**

---

# 2. Polymorphism — multiple behaviors under same contract

### Intent: *“Different systems can send notifications differently, but I treat them uniformly.”*

```python
class EmailSender(NotificationSender):
    def send(self, message: str) -> None:
        print(f"Email: {message}")

class SMSSender(NotificationSender):
    def send(self, message: str) -> None:
        print(f"SMS: {message}")

class PushSender(NotificationSender):
    def send(self, message: str) -> None:
        print(f"Push: {message}")
```

### Usage (key point)

```python
def notify_all(sender: NotificationSender):
    sender.send("System down!")
```

Now:

```python
notify_all(EmailSender())
notify_all(SMSSender())
notify_all(PushSender())
```

### Causal meaning

* Same *interface*
* Different *execution paths*
* Caller is isolated from implementation choice

This is **runtime substitution under abstraction**

---

# 3. Inheritance — reuse + specialization via hierarchy

### Intent: *“I want to build a refined version of an existing class.”*

Example: logging behavior added to all senders.

```python
class LoggingEmailSender(EmailSender):
    def send(self, message: str) -> None:
        print("LOG: sending email")
        super().send(message)
```

### Structural meaning

* You are extending a concrete implementation
* You inherit both structure + behavior
* You override parts of it

### Key implication

This creates a **vertical dependency chain**:

```
NotificationSender
   ↓
EmailSender
   ↓
LoggingEmailSender
```

### Risk signal (important in design)

* Tight coupling to parent implementation
* Fragile base class problem (changes upstream ripple down)

---

# 4. Composition — build behavior by combining components

### Intent: *“Instead of extending, I assemble behavior from parts.”*

We separate concerns:

```python
class Logger:
    def log(self, msg):
        print(f"LOG: {msg}")
```

Now compose it:

```python
class EmailSender(NotificationSender):
    def __init__(self, logger: Logger):
        self.logger = logger

    def send(self, message: str) -> None:
        self.logger.log("sending email")
        print(f"Email: {message}")
```

### Usage

```python
sender = EmailSender(Logger())
sender.send("Hello")
```

### Structural meaning

* No hierarchy needed
* Behavior is assembled horizontally
* Dependencies are explicit

---

# 5. The real contrast (causal view)

## Inheritance path

You are saying:

> “My identity is a specialized version of something else.”

This creates:

* Type hierarchy
* Implicit coupling
* Extension via override

---

## Composition path

You are saying:

> “My behavior is built from independent capabilities.”

This creates:

* Modular system
* Explicit dependencies
* Flexible runtime assembly

---

## Abstraction + Polymorphism (separate axis)

These are orthogonal:

* Abstraction = defines the rule
* Polymorphism = multiple ways to satisfy the rule

They do **not require inheritance** (composition can still use interfaces).

---

# 6. One unified mental model

Think of system design like this pipeline:

### Step 1: Abstraction

Define *what the system must do*

### Step 2: Polymorphism

Allow *multiple valid implementations*

### Step 3: Composition vs Inheritance

Choose how to structure implementations:

* Inheritance → vertical reuse
* Composition → horizontal assembly

---

If you want, I can map this directly onto **LLD interview problems (LRU cache, parking lot, elevator system)** and show exactly where candidates usually misuse inheritance when composition is expected.
