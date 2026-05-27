# Design Patterns Decision Matrix (Compressed)

Use design patterns when the problem shape is repeating. Reach for the simplest pattern that removes the main source of change or coupling.

ROI scoring: `5 = most interview-useful`, `3 = medium frequency or secondary`, `1 = know it, but rarely lead with it`.

| Situation | Symptom | Pattern | Use It For | Avoid When | Compression | ROI to Learn |
| --- | --- | --- | --- | --- | --- | --- |
| Algorithm varies by policy | `if` chain chooses formula | Strategy | Swap behavior cleanly | There is one stable algorithm | Behavior slot | 5 |
| Object needs stepwise validated creation | Telescoping constructors | Builder | Assemble valid objects progressively | Object is tiny and obvious | Controlled assembly | 5 |
| One change should notify many listeners | Polling or manual fan-out | Observer | Event-driven updates | Strict synchronous workflow is required | One-to-many signal | 5 |
| State-specific behavior explodes conditionals | Status checks in every method | State | Model legal transitions explicitly | States are few and static enough for one enum switch | Behavior by current state | 5 |
| Object creation is complex | Many constructor branches | Factory Method | Delegate creation choice | A plain constructor is enough | Creation indirection | 4 |
| Extra behavior wraps a core object | Logging, retry, caching layers | Decorator | Compose orthogonal features | Order of wrappers is confusing or fragile | Layered behavior | 4 |
| Existing API is wrong shape | Legacy interface mismatch | Adapter | Translate contracts | You control both sides and can change them | Interface translation | 4 |
| Related families must stay compatible | Cross-product of object variants | Abstract Factory | Create coherent families together | Only one product type exists | Family-level creation | 3 |
| One system talks to many peers | Mesh of object-to-object calls | Mediator | Centralize collaboration rules | Interaction is simple and local | Hub coordination | 3 |
| Need one shared facade over subsystem complexity | Too many entry points | Facade | Offer a simpler front door | Clients need fine-grained control anyway | Simplified surface | 3 |
| Need reversible or queueable actions | Actions need history or async dispatch | Command | Package requests as objects | A direct method call is sufficient | Explicit action object | 3 |
| Tree structure should look uniform | Leaf vs group branching everywhere | Composite | Treat part and whole consistently | Structure is not hierarchical | Uniform recursion | 3 |
| Need staged access or lazy behavior | Direct access is too expensive or unsafe | Proxy | Control, defer, or secure access | Added indirection gives little value | Controlled stand-in | 3 |
| Traversal/operation changes often | New operation edits every node class | Visitor | Add operations over stable structure | Structure changes more often than operations | Externalized operation | 2 |
| One object should control a resource | Global shared coordinator | Singleton | One logical instance with explicit access point | Global state harms testing or lifecycle control | Managed single instance | 2 |
| Sequential handlers may or may not act | Nested conditionals by responsibility | Chain of Responsibility | Pass request until one handles it | Routing is static and obvious | Ordered handoff | 2 |
| Need stable skeleton with customizable steps | Variants share one high-level flow | Template Method | Reuse algorithm skeleton | Inheritance cost is too high; prefer composition | Fixed flow, variable steps | 2 |
| Need to save snapshots for rollback | Manual field copying | Memento | Capture and restore state | State is huge or externally owned | Snapshot | 1 |
| Shared heavy objects are repeatedly recreated | Memory or init cost too high | Flyweight | Share intrinsic state | Object identity and mutation dominate | Shared intrinsic state | 1 |

---

# Pattern Selection Heuristics

| If the change axis is... | Usually start with... |
| --- | --- |
| varying algorithm | Strategy |
| varying creation logic | Factory or Builder |
| varying wrappers/cross-cutting behavior | Decorator |
| varying collaboration graph | Mediator |
| varying state-dependent behavior | State |
| varying external interfaces | Adapter or Facade |
| varying follow-up reactions | Observer |
| varying tree operations | Composite or Visitor |

## Highest-ROI Patterns First

1. Strategy
2. Builder
3. State
4. Observer
5. Factory Method
6. Adapter
7. Decorator

---

# Neutral Examples

## Strategy

A shipping service can choose `FlatRatePricing`, `DistancePricing`, or `SurgePricing` without changing the checkout flow.

## Builder

A document export request may require optional watermark, page size, locale, and compression settings; a builder keeps the setup readable and valid.

## Decorator

A base `DataFetcher` can be wrapped with caching, retry, and metrics decorators without modifying the fetcher itself.

## Adapter

A new analytics client can adapt an old vendor SDK so the rest of the code still calls `track(event)`.

## Observer

A user profile update can notify audit logging, email preferences sync, and search indexing listeners after the main change succeeds.

## State

A media player can behave differently in `Playing`, `Paused`, and `Stopped` states without scattering status checks throughout the code.

## Command

A text editor can represent `InsertText`, `DeleteText`, and `PasteClipboard` as commands to support undo and macro replay.

## Composite

A graphics editor can treat both a single shape and a grouped shape collection through the same `draw()` interface.
