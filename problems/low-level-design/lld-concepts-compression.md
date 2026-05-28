# LLD Concepts Ranked Decision Matrix

Use this as a compressed selection sheet for low-level design. Read the table from top to bottom: higher rows have higher learning ROI and show up more often in interviews and real system code.

ROI scoring: `5 = foundational and high-frequency`, `4 = very useful`, `3 = useful but more situational`, `2 = secondary`, `1 = low-frequency or advanced edge`.

| Rank | LLD Concept | When to Use It | Core Problem It Solves | Why It Works | What It Looks Like | Learning Goal Compression | ROI |
| --- | --- | --- | --- | --- | --- | --- | --- |
| 1 | Encapsulation | State has rules and should not be mutated freely | Invalid state and scattered validation | Keeps invariants next to the data they protect | `account.withdraw(amount)` instead of public balance writes | Put state and guardrails together | 5 |
| 2 | Abstraction / Interfaces | Callers should depend on capability, not implementation | Tight coupling to concrete classes | Stabilizes boundaries and allows substitution | `PaymentGateway`, `NotificationChannel` interfaces | Separate what clients need from how it is done | 5 |
| 3 | Composition | A class is growing into a god object | Monolithic classes and mixed responsibilities | Splits behavior into collaborating parts with clear ownership | `CheckoutService` composed from pricing, tax, payment collaborators | Prefer object graphs over giant inheritance trees | 5 |
| 4 | State Modeling / FSM | Behavior changes by lifecycle stage | `if/else` explosion by status | Makes legal transitions explicit and local | `Created -> Paid -> Shipped` state objects or enum transition table | Model lifecycle as a transition graph | 5 |
| 5 | Polymorphism | Different variants share one contract but behave differently | Branching by type | Pushes variation behind one call site | `channel.send()` for email/SMS/push | Replace type checks with dispatch | 5 |
| 6 | Single Responsibility / Cohesion | One class changes for many unrelated reasons | Low cohesion and ripple-effect edits | Groups logic that changes together | separate `InventoryService`, `PricingService`, `OrderService` | One owner per reason to change | 5 |
| 7 | Dependency Inversion | High-level policy is tied to infra details | Hard-to-test business logic and vendor lock-in | Reverses dependency direction toward stable contracts | domain depends on `Repository` interface, not SQL class | Depend inward on policies, not outward on details | 5 |
| 8 | Invariants / Aggregate Boundaries | Multiple fields must change consistently | Broken business rules across methods | Defines the transaction boundary for safe mutation | `Reservation.confirm()` checks seat, time, payment together | Protect consistency at the boundary that owns it | 5 |
| 9 | Strategy | One algorithm varies by policy or runtime choice | Branching over interchangeable business rules | Isolates the change axis cleanly | `PricingStrategy`, `RoutingStrategy` | One slot for varying behavior | 5 |
| 10 | Builder / Valid Construction | Object creation has many options or validation rules | Invalid half-built objects | Forces controlled assembly of valid objects | `OrderBuilder().withItems(...).withCoupon(...)` | Make invalid construction hard | 5 |
| 11 | Observer / Pub-Sub | One change should trigger multiple follow-ups | Manual fan-out and forgotten side effects | Decouples producer from many consumers | order placed event notifies billing, analytics, email | One-to-many reaction without hard wiring | 4 |
| 12 | Interface Segregation | Different users need different slices of an API | Fat interfaces and fake implementations | Narrows contracts to role-specific usage | `ReadableCatalog` vs `CatalogAdmin` | Keep APIs as small as the caller needs | 4 |
| 13 | Tell, Don't Ask | Callers pull raw state and decide behavior externally | Logic drift outside object owners | Pushes intent to the object with the knowledge | `cart.applyCoupon(code)` instead of external field poking | Put behavior where the data lives | 4 |
| 14 | Factory Method / Simple Factory | Creation choice depends on input or configuration | Constructors know too much | Centralizes creation branching | `NotificationFactory.create(type)` | Hide creation branching behind one seam | 4 |
| 15 | Decorator | Orthogonal features must stack without subclass explosion | Logging/retry/cache concerns polluting core logic | Wraps behavior additively and locally | cached -> retried -> metered data fetcher | Layer features around a stable core | 4 |
| 16 | Adapter | Existing or third-party API has the wrong shape | Integration mismatch | Translates one contract into another | vendor SDK adapted to `PaymentGateway` | Fix the boundary, not the whole system | 4 |
| 17 | Command | Actions need queueing, history, retries, or undo | Calls are too implicit and not portable | Turns behavior into an explicit object | `CancelReservationCommand` | Reify actions as first-class units | 3 |
| 18 | Repository | Domain logic should not know storage mechanics | Persistence details leaking everywhere | Creates a storage-facing boundary aligned to domain access | `OrderRepository.findById()` | Separate domain access intent from storage details | 3 |
| 19 | Composite | Part and whole should be treated uniformly | Recursive tree handling with special cases | Gives one interface over hierarchical structures | file and folder both implement `Node` | Uniform recursion over trees | 3 |
| 20 | Mediator | Many peers coordinate in tangled ways | Mesh communication and coupling | Centralizes collaboration rules | chat room, workflow coordinator, UI form mediator | Replace peer-to-peer mesh with hub coordination | 3 |
| 21 | Liskov Substitution | A subtype is proposed in an inheritance hierarchy | Surprising subtype behavior | Forces semantic compatibility between parent and child | subtype should not break parent expectations | Inheritance must preserve contract, not just compile | 3 |
| 22 | Proxy | Access must be controlled, deferred, or virtualized | Expensive or sensitive dependency access | Adds a stand-in with the same contract | lazy image loader, access-controlled service proxy | Same interface, different access policy | 2 |
| 23 | Template Method | Variants share one stable high-level flow | Duplicate workflow skeletons | Reuses fixed sequence with overridable steps | import pipeline with customizable validation hooks | Fixed algorithm, variable steps | 2 |
| 24 | Visitor | Structure is stable but operations keep changing | Every new operation edits many node classes | Externalizes operations from the object structure | AST visitor, pricing rule visitor over stable nodes | Stable structure, expanding operation set | 2 |
| 25 | Flyweight | Huge numbers of similar objects waste memory | Repeated intrinsic state duplication | Shares common immutable state | glyphs, map tiles, repeated icons | Separate shared intrinsic state from per-instance data | 1 |
| 26 | Singleton | There should be one logical instance with controlled access | Accidental multiple coordinators | Centralizes access to a unique service instance | config registry, process-wide coordinator | Use sparingly; global access trades convenience for test pain | 1 |

## Reading Heuristic

If you are unsure what to reach for:

| Problem Shape | Start With |
| --- | --- |
| Invalid state transitions | Encapsulation + invariants + state modeling |
| One class doing too much | Single responsibility + composition |
| Variant behavior keeps adding `if/else` | Abstraction + polymorphism + strategy |
| Infra details leaking into domain logic | Dependency inversion + repository + adapter |
| One action triggers many side effects | Observer or command |
| Creation logic is messy | Builder or factory |
| Integration contract mismatch | Adapter |
| Cross-cutting behavior stacks around a core | Decorator |

## Highest-ROI Learning Order

1. Encapsulation
2. Abstraction
3. Composition
4. State modeling
5. Polymorphism
6. Single responsibility
7. Dependency inversion
8. Invariants / aggregate boundaries
9. Strategy
10. Builder

## Examples For Each Concept

### 1. Encapsulation

A `BankAccount` should expose `deposit()` and `withdraw()` instead of allowing callers to set `balance` directly, so overdraft and negative-amount rules stay in one place.

```python
class BankAccount:
    def __init__(self) -> None:
        self._balance = 0

    def deposit(self, amount: int) -> None:
        if amount <= 0:
            raise ValueError("amount must be positive")
        self._balance += amount

    def withdraw(self, amount: int) -> None:
        if amount <= 0 or amount > self._balance:
            raise ValueError("invalid withdrawal amount")
        self._balance -= amount

    @property
    def balance(self) -> int:
        return self._balance
```

### 2. Abstraction / Interfaces

A checkout flow can depend on a `PaymentGateway` interface while Stripe and Razorpay adapters provide different concrete implementations behind that boundary.

```python
from abc import ABC, abstractmethod


class PaymentGateway(ABC):
    @abstractmethod
    def charge(self, amount: int) -> None:
        pass


class StripeGateway(PaymentGateway):
    def charge(self, amount: int) -> None:
        print(f"Stripe charged {amount}")


class CheckoutService:
    def __init__(self, gateway: PaymentGateway) -> None:
        self.gateway = gateway

    def checkout(self, amount: int) -> None:
        self.gateway.charge(amount)
```

### 3. Composition

A `CheckoutService` can be built from `PricingService`, `TaxService`, and `PaymentService` collaborators instead of becoming one giant class that owns every rule.

```python
class PricingService:
    def subtotal(self, item_price: int, quantity: int) -> int:
        return item_price * quantity


class TaxService:
    def tax(self, amount: int) -> int:
        return amount // 10


class PaymentService:
    def pay(self, amount: int) -> None:
        print(f"Paid {amount}")


class CheckoutService:
    def __init__(self) -> None:
        self.pricing = PricingService()
        self.tax = TaxService()
        self.payment = PaymentService()

    def checkout(self, item_price: int, quantity: int) -> None:
        subtotal = self.pricing.subtotal(item_price, quantity)
        total = subtotal + self.tax.tax(subtotal)
        self.payment.pay(total)
```

### 4. State Modeling / FSM

An `Order` can move through `CREATED`, `PAID`, `PACKED`, `SHIPPED`, and `DELIVERED`, with each transition validated explicitly so shipping cannot happen before payment.

```python
from enum import Enum, auto


class OrderState(Enum):
    CREATED = auto()
    PAID = auto()
    PACKED = auto()
    SHIPPED = auto()
    DELIVERED = auto()


class Order:
    def __init__(self) -> None:
        self.state = OrderState.CREATED

    def pay(self) -> None:
        if self.state is not OrderState.CREATED:
            raise ValueError("order must be created before payment")
        self.state = OrderState.PAID

    def pack(self) -> None:
        if self.state is not OrderState.PAID:
            raise ValueError("order must be paid before packing")
        self.state = OrderState.PACKED

    def ship(self) -> None:
        if self.state is not OrderState.PACKED:
            raise ValueError("order must be packed before shipping")
        self.state = OrderState.SHIPPED
```

### 5. Polymorphism

A notification module can call `channel.send(message)` for `EmailChannel`, `SmsChannel`, and `PushChannel` without branching on channel type.

```python
from abc import ABC, abstractmethod


class NotificationChannel(ABC):
    @abstractmethod
    def send(self, message: str) -> None:
        pass


class EmailChannel(NotificationChannel):
    def send(self, message: str) -> None:
        print(f"Email: {message}")


class SmsChannel(NotificationChannel):
    def send(self, message: str) -> None:
        print(f"SMS: {message}")


class Notifier:
    def notify(self, channel: NotificationChannel, message: str) -> None:
        channel.send(message)
```

### 6. Single Responsibility / Cohesion

Instead of one `OrderManager` doing inventory checks, payment capture, and invoice generation, split those into focused services that each change for one reason.

```python
class InventoryService:
    def in_stock(self, sku: str) -> bool:
        return True


class BillingService:
    def charge(self, user_id: str, amount: int) -> None:
        print(f"Charged {user_id}")


class InvoiceService:
    def generate(self, user_id: str, amount: int) -> None:
        print(f"Invoice for {user_id}")
```

### 7. Dependency Inversion

A fraud-check workflow should depend on a `RiskScorer` contract, not directly on one vendor SDK, so tests can swap in a fake scorer and production can swap vendors later.

```python
from abc import ABC, abstractmethod


class RiskScorer(ABC):
    @abstractmethod
    def score(self, user_id: str) -> int:
        pass


class VendorRiskScorer(RiskScorer):
    def score(self, user_id: str) -> int:
        return 42


class FraudCheckService:
    def __init__(self, scorer: RiskScorer) -> None:
        self.scorer = scorer

    def allow(self, user_id: str) -> bool:
        return self.scorer.score(user_id) < 80
```

### 8. Invariants / Aggregate Boundaries

A `Reservation` aggregate can own seat assignment, payment status, and expiry rules so callers cannot confirm two users into the same seat through separate object updates.

```python
class Reservation:
    def __init__(self, seat_id: str) -> None:
        self.seat_id = seat_id
        self.paid = False
        self.confirmed = False

    def mark_paid(self) -> None:
        self.paid = True

    def confirm(self) -> None:
        if not self.paid:
            raise ValueError("payment required")
        if self.confirmed:
            raise ValueError("already confirmed")
        self.confirmed = True
```

### 9. Strategy

A delivery system can choose `BikeRoutingStrategy`, `CarRoutingStrategy`, or `WalkingRoutingStrategy` based on distance and city zone without changing dispatch flow.

```python
from abc import ABC, abstractmethod


class RoutingStrategy(ABC):
    @abstractmethod
    def eta_minutes(self, distance_km: int) -> int:
        pass


class BikeRoutingStrategy(RoutingStrategy):
    def eta_minutes(self, distance_km: int) -> int:
        return distance_km * 4


class CarRoutingStrategy(RoutingStrategy):
    def eta_minutes(self, distance_km: int) -> int:
        return distance_km * 2


class Dispatcher:
    def estimate(self, strategy: RoutingStrategy, distance_km: int) -> int:
        return strategy.eta_minutes(distance_km)
```

### 10. Builder / Valid Construction

A `SearchQueryBuilder` can assemble optional filters like language, date range, sort order, and page size while preventing invalid combinations.

```python
from dataclasses import dataclass


@dataclass(frozen=True)
class SearchQuery:
    language: str
    sort_by: str
    page_size: int


class SearchQueryBuilder:
    def __init__(self) -> None:
        self._language = "en"
        self._sort_by = "relevance"
        self._page_size = 10

    def language(self, language: str) -> "SearchQueryBuilder":
        self._language = language
        return self

    def sort_by(self, sort_by: str) -> "SearchQueryBuilder":
        self._sort_by = sort_by
        return self

    def page_size(self, page_size: int) -> "SearchQueryBuilder":
        if page_size <= 0:
            raise ValueError("page_size must be positive")
        self._page_size = page_size
        return self

    def build(self) -> SearchQuery:
        return SearchQuery(self._language, self._sort_by, self._page_size)
```

### 11. Observer / Pub-Sub

When an order is placed, the order service can publish an event that billing, email, analytics, and warehouse listeners all react to independently.

```python
from abc import ABC, abstractmethod


class OrderListener(ABC):
    @abstractmethod
    def on_order_placed(self, order_id: str) -> None:
        pass


class EmailListener(OrderListener):
    def on_order_placed(self, order_id: str) -> None:
        print(f"Email sent for {order_id}")


class OrderPublisher:
    def __init__(self) -> None:
        self.listeners: list[OrderListener] = []

    def subscribe(self, listener: OrderListener) -> None:
        self.listeners.append(listener)

    def place_order(self, order_id: str) -> None:
        for listener in self.listeners:
            listener.on_order_placed(order_id)
```

### 12. Interface Segregation

Warehouse staff may need `reserveStock()` and `releaseStock()`, while storefront code only needs `getAvailability()`, so one large inventory API should be split into narrower interfaces.

```python
from abc import ABC, abstractmethod


class InventoryReader(ABC):
    @abstractmethod
    def get_availability(self, sku: str) -> int:
        pass


class InventoryWriter(ABC):
    @abstractmethod
    def reserve_stock(self, sku: str, count: int) -> None:
        pass

    @abstractmethod
    def release_stock(self, sku: str, count: int) -> None:
        pass


class InventoryService(InventoryReader, InventoryWriter):
    def get_availability(self, sku: str) -> int:
        return 10

    def reserve_stock(self, sku: str, count: int) -> None:
        pass

    def release_stock(self, sku: str, count: int) -> None:
        pass
```

### 13. Tell, Don't Ask

Instead of reading cart fields externally to compute discounts and mutate totals, call `cart.applyCoupon(code)` and let the cart enforce its own pricing rules.

```python
class Cart:
    def __init__(self, total: int) -> None:
        self._total = total

    def apply_coupon(self, code: str) -> None:
        if code == "SAVE10":
            self._total -= 10

    @property
    def total(self) -> int:
        return self._total
```

### 14. Factory Method / Simple Factory

A document export module can call `ExporterFactory.create("pdf")` to get the right exporter without spreading construction logic across controllers.

```python
from abc import ABC, abstractmethod


class Exporter(ABC):
    @abstractmethod
    def export(self, data: str) -> None:
        pass


class PdfExporter(Exporter):
    def export(self, data: str) -> None:
        print(f"PDF: {data}")


class CsvExporter(Exporter):
    def export(self, data: str) -> None:
        print(f"CSV: {data}")


class ExporterFactory:
    @staticmethod
    def create(export_type: str) -> Exporter:
        if export_type == "pdf":
            return PdfExporter()
        if export_type == "csv":
            return CsvExporter()
        raise ValueError("unknown type")
```

### 15. Decorator

A base `DataFetcher` can be wrapped by caching, retry, and metrics decorators so each concern is layered independently around the same fetch contract.

```python
from abc import ABC, abstractmethod


class DataFetcher(ABC):
    @abstractmethod
    def fetch(self) -> str:
        pass


class ApiFetcher(DataFetcher):
    def fetch(self) -> str:
        return "data"


class LoggingFetcher(DataFetcher):
    def __init__(self, inner: DataFetcher) -> None:
        self.inner = inner

    def fetch(self) -> str:
        print("fetching...")
        return self.inner.fetch()
```

### 16. Adapter

If a third-party shipping SDK exposes `createShipment(payload)` but your domain expects `ship(order)`, an adapter can translate between the two.

```python
from abc import ABC, abstractmethod


class ThirdPartyShippingSdk:
    def create_shipment(self, payload: str) -> None:
        print(f"SDK shipment: {payload}")


class ShippingGateway(ABC):
    @abstractmethod
    def ship(self, order_id: str) -> None:
        pass


class ShippingAdapter(ShippingGateway):
    def __init__(self) -> None:
        self.sdk = ThirdPartyShippingSdk()

    def ship(self, order_id: str) -> None:
        self.sdk.create_shipment(f"{{order_id: {order_id}}}")
```

### 17. Command

A text editor can represent `InsertText`, `DeleteText`, and `Paste` as command objects so it can support undo, redo, and macro replay.

```python
from abc import ABC, abstractmethod


class Command(ABC):
    @abstractmethod
    def execute(self) -> None:
        pass


class InsertTextCommand(Command):
    def __init__(self, doc: list[str], text: str) -> None:
        self.doc = doc
        self.text = text

    def execute(self) -> None:
        self.doc.append(self.text)
```

### 18. Repository

An `OrderRepository` can expose methods like `findById()` and `save()` so business logic does not need to know whether data lives in Postgres, Redis, or a mock store.

```python
from abc import ABC, abstractmethod
from dataclasses import dataclass


@dataclass
class Order:
    id: str


class OrderRepository(ABC):
    @abstractmethod
    def find_by_id(self, order_id: str) -> Order:
        pass

    @abstractmethod
    def save(self, order: Order) -> None:
        pass


class InMemoryOrderRepository(OrderRepository):
    def find_by_id(self, order_id: str) -> Order:
        return Order(order_id)

    def save(self, order: Order) -> None:
        print(f"saved {order.id}")
```

### 19. Composite

A design tool can treat both a single `Shape` and a grouped `ShapeGroup` through the same `draw()` interface.

```python
from abc import ABC, abstractmethod


class Shape(ABC):
    @abstractmethod
    def draw(self) -> None:
        pass


class Circle(Shape):
    def draw(self) -> None:
        print("circle")


class ShapeGroup(Shape):
    def __init__(self, shapes: list[Shape]) -> None:
        self.shapes = shapes

    def draw(self) -> None:
        for shape in self.shapes:
            shape.draw()
```

### 20. Mediator

In a UI form, changing country may need to reset state, city, and postal-code validation rules; a mediator can coordinate that instead of every field talking to every other field.

```python
class AddressFormMediator:
    def on_country_changed(
        self,
        country: str,
        state: "StateField",
        city: "CityField",
    ) -> None:
        state.reset()
        city.reset()
        print(f"rules updated for {country}")


class StateField:
    def reset(self) -> None:
        pass


class CityField:
    def reset(self) -> None:
        pass
```

### 21. Liskov Substitution

If `Bird` has `fly()`, then making `Penguin` a subtype is suspicious because callers expecting every bird to fly will break; the hierarchy should be redesigned.

```python
class Bird:
    def move(self) -> None:
        print("bird moves")


class FlyingBird(Bird):
    def fly(self) -> None:
        print("bird flies")


class Penguin(Bird):
    def swim(self) -> None:
        print("penguin swims")
```

### 22. Proxy

A `DocumentProxy` can delay loading a large PDF from remote storage until the user actually opens it, while still exposing the same `Document` interface.

```python
from abc import ABC, abstractmethod


class Document(ABC):
    @abstractmethod
    def content(self) -> str:
        pass


class RealDocument(Document):
    def content(self) -> str:
        return "large pdf bytes"


class DocumentProxy(Document):
    def __init__(self) -> None:
        self.real: RealDocument | None = None

    def content(self) -> str:
        if self.real is None:
            self.real = RealDocument()
        return self.real.content()
```

### 23. Template Method

A file import pipeline can define fixed steps like parse, validate, transform, and persist, while subclasses customize only the validation or transformation step.

```python
from abc import ABC, abstractmethod


class FileImporter(ABC):
    def run(self, file: str) -> None:
        self.parse(file)
        self.validate(file)
        self.persist(file)

    def parse(self, file: str) -> None:
        print(f"parse {file}")

    @abstractmethod
    def validate(self, file: str) -> None:
        pass

    def persist(self, file: str) -> None:
        print(f"persist {file}")


class CsvImporter(FileImporter):
    def validate(self, file: str) -> None:
        print(f"validate csv {file}")
```

### 24. Visitor

An expression tree for a compiler can stay structurally stable while visitors add interpretation, optimization, and pretty-printing as separate operations.

```python
from abc import ABC, abstractmethod
from dataclasses import dataclass


class Expr(ABC):
    @abstractmethod
    def accept(self, visitor: "ExprVisitor") -> None:
        pass


@dataclass
class NumberExpr(Expr):
    value: int

    def accept(self, visitor: "ExprVisitor") -> None:
        visitor.visit_number(self)


class ExprVisitor(ABC):
    @abstractmethod
    def visit_number(self, expr: NumberExpr) -> None:
        pass


class PrintVisitor(ExprVisitor):
    def visit_number(self, expr: NumberExpr) -> None:
        print(expr.value)
```

### 25. Flyweight

A text editor can share one glyph object per character/font combination instead of allocating a new heavy object for every visible letter on screen.

```python
from dataclasses import dataclass


@dataclass(frozen=True)
class Glyph:
    ch: str


class GlyphFactory:
    def __init__(self) -> None:
        self.cache: dict[str, Glyph] = {}

    def get(self, ch: str) -> Glyph:
        if ch not in self.cache:
            self.cache[ch] = Glyph(ch)
        return self.cache[ch]
```

### 26. Singleton

A process-wide configuration registry may have one logical instance loaded once at startup, though this should be used carefully because it increases global coupling.

```python
class ConfigRegistry:
    _instance: "ConfigRegistry | None" = None

    def __new__(cls) -> "ConfigRegistry":
        if cls._instance is None:
            cls._instance = super().__new__(cls)
        return cls._instance
```
