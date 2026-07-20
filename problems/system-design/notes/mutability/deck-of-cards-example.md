The problem immutability solves is fundamentally **uncontrolled state change**.

A mutable object says:

$$
\text{same identity} + \text{different state over time}
$$

A frozen/immutable object says:

$$
\text{same value} \Rightarrow \text{same meaning forever}
$$

This sounds simple, but it removes several classes of problems.

---

## 1. Preventing invariant violations

Your deck example is the clearest.

Mathematically:

$$
Deck = \operatorname{Bij}(P,C)
$$

A valid deck must satisfy:

* every position has a card
* every card appears exactly once

With mutable state:

```python
deck.cards[0] = deck.cards[1]
```

Now:

$$
d(0)=d(1)
$$

which violates injectivity.

The object still exists, but it no longer belongs to the set:

$$
\operatorname{Bij}(P,C)
$$

You have created an invalid state.

With immutable design:

```python
new_deck = Deck(...)
```

the constructor validates:

$$
new\_deck\in\operatorname{Bij}(P,C)
$$

The old deck remains valid.

---

## 2. Reasoning locally

Mutable:

```python
id="a9h6gt"
deck.shuffle()
```

A function receiving `deck` must ask:

* Did someone modify it?
* Did another reference modify it?
* Is it still valid?
* Did another thread change it?

The state space becomes:

$$
Deck \times Time
$$

You need to reason about history.

---

Immutable:

```python
id="v0m6v7"
new_deck = deck.shuffle()
```

The function has a mathematical contract:

$$
shuffle:D\rightarrow D
$$

Input and output are separate.

You can reason from the value alone.

---

## 3. Aliasing bugs

This is a huge practical problem.

Example:

```python
id="zzj8sa"
a = [1,2,3]
b = a

b.append(4)
```

Now:

```python
id="w8o2bq"
a == [1,2,3,4]
```

even though you changed `b`.

Why?

Because:

$$
a\rightarrow X
$$

and

$$
b\rightarrow X
$$

Both references point to the same mutable object.

With immutable objects:

```python
id="v1z7n4"
a = (1,2,3)
b = a
```

there is no possibility that `b` changes `a`.

Sharing is safe.

---

## 4. Concurrency

Suppose two threads:

```
Thread A:
deck.shuffle()

Thread B:
deck.position_of(card)
```

With mutation:

Possible states:

$$
d_0\rightarrow d_1\rightarrow d_2
$$

Thread B might observe a partially modified object.

You need:

* locks
* synchronization
* defensive copying

With immutable values:

Thread A:

$$
d_0\rightarrow d_1
$$

Thread B:

still sees:

$$
d_0
$$

No race over the value.

---

## 5. Hashing and dictionaries

Python requires dictionary keys to have stable hashes.

Example:

```python
id="pqlvpp"
d = {
    deck: "player hand"
}
```

If the deck mutates:

$$
hash(deck\_t)\neq hash(deck_{t+1})
$$

the dictionary can no longer find it.

Immutable objects can safely be keys.

This is why:

```python
tuple
frozenset
int
str
```

can be dictionary keys.

---

## 6. Functional composition

Mathematically we like functions:

$$
f:X\rightarrow Y
$$

because they compose:

$$
g\circ f:X\rightarrow Z
$$

Mutation breaks this intuition.

Instead of:

$$
shuffle\circ cut\circ deal
$$

you have:

```
change object
change object
change object
```

with hidden effects.

Immutable objects preserve algebraic reasoning.

---

## Category theory view

A mutable object behaves like an object in a **state category**.

You have:

$$
S \xrightarrow{transition} S
$$

where the same state evolves.

An immutable object behaves like an ordinary object in a category:

$$
f:X\rightarrow Y
$$

where morphisms produce new objects.

For your deck:

Immutable:

$$
\operatorname{shuffle}:Deck\times S_{52}\rightarrow Deck
$$

Mutable:

$$
\operatorname{shuffle}:State(Deck)\rightarrow State(Deck)
$$

The first is algebraic; the second requires tracking effects.

---

## Why not make everything immutable?

Because mutation is sometimes useful.

Example:

A database buffer:

```python
buffer.append(record)
```

is naturally a changing state.

Making it immutable could require constantly allocating new buffers.

So the tradeoff:

| Mutable               | Immutable                     |
| --------------------- | ----------------------------- |
| Fast updates          | Easy reasoning                |
| Less allocation       | Safer sharing                 |
| Good for local state  | Good for values               |
| Needs synchronization | Naturally thread-safe         |
| More hidden coupling  | More explicit transformations |

A useful design rule:

* **Entities with identity and lifecycle** → often mutable (`UserSession`, cache, connection).
* **Values defined by their content** → often immutable (`Money`, `Date`, `Deck`, `Vector`, `Permutation`).

Your deck example is almost the ideal immutable object because mathematically it is a value:

$$
d\in\operatorname{Bij}(P,C)
$$

not an entity that "changes through time."

## Example of Deck using immutable deck

With a frozen `Deck`, the shuffle operation changes from a **mutation command** to a **pure transformation**.

Instead of:

```python
deck.shuffle()
```

meaning:

$$
d \leftarrow d\circ\sigma^{-1}
$$

you make it:

```python
new_deck = deck.shuffle(shuffle)
```

meaning:

$$
\operatorname{shuffle}:D\times S_n\rightarrow D
$$

The old deck is untouched; a new deck is returned.

---

## Example design

```python
from dataclasses import dataclass
from random import shuffle


@dataclass(frozen=True)
class Deck:
    cards: tuple[Card, ...]

    def __call__(self, position: int) -> Card:
        return self.cards[position]

    def shuffle_by(self, permutation: tuple[int, ...]) -> "Deck":
        """
        Apply permutation action:

        d' = d ∘ σ⁻¹
        """

        if sorted(permutation) != list(range(len(self.cards))):
            raise ValueError("Not a valid permutation")

        new_cards = tuple(
            self.cards[permutation[i]]
            for i in range(len(self.cards))
        )

        return Deck(cards=new_cards)
```

Usage:

```python
deck0 = Deck.identity()

deck1 = deck0.shuffle_by(permutation)
```

Memory:

```
deck0
 |
 v
Deck
cards ---> (A♠, K♠, Q♠, ...)



deck1
 |
 v
Deck
cards ---> (7♣, A♠, K♥, ...)
```

`deck0` still exists.

---

## A shuffle handler in an application

Usually you separate the **domain object** from the **handler/service**.

For example:

```python
class ShuffleHandler:

    def __init__(self, shuffler):
        self.shuffler = shuffler

    def handle(self, deck: Deck) -> Deck:
        permutation = self.shuffler.generate(len(deck.cards))
        return deck.shuffle_by(permutation)
```

Then:

```python
handler = ShuffleHandler(RandomShuffler())

new_deck = handler.handle(deck)
```

The handler does not modify the deck.

---

## Why is this nicer?

Compare mutable:

```python
deck.shuffle()
deal(deck)
```

You must know:

* Did `shuffle()` happen?
* Is this the old deck?
* Did someone retain a reference?
* Is another thread using it?

With immutable:

```python
shuffled = shuffle(deck)
hand = deal(shuffled)
```

The data flow is explicit:

$$
deck_0 \xrightarrow{shuffle} deck_1 \xrightarrow{deal} hand
$$

This is basically function composition.

---

## In a stateful application

You may still have a mutable owner of the current deck:

```python
@dataclass
class GameState:
    current_deck: Deck
```

Then the state transition is:

```python
def shuffle_game(state: GameState):
    state.current_deck = state.current_deck.shuffle_by(permutation)
```

Notice the distinction:

* `Deck` is immutable value.
* `GameState` is mutable state container.

This is a very common architecture.

Mathematically:

$$
GameState \xrightarrow{transition} GameState
$$

contains:

$$
currentDeck \in Deck
$$

which remains a stable value.

---

## Rust analogy

This is exactly how Rust encourages you to write it:

```rust
impl Deck {
    fn shuffle(&self, permutation: Permutation) -> Deck {
        Deck {
            cards: new_cards
        }
    }
}
```

The receiver is:

```rust
&self
```

(read-only borrow)

and the output is:

```rust
Deck
```

(new owned value).

---

So the pattern is:

$$
\boxed{
\begin{array}{c}
\text{immutable domain objects} \\
+ \\
\text{mutable state holders}
\end{array}
}
$$

The deck itself should not "change"; the **game state changes to point at a new deck**. This preserves the algebraic structure:

$$
\operatorname{shuffle}:D\times S_n\rightarrow D
$$

while still allowing an application to evolve over time.
