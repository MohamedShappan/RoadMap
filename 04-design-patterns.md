# 04 — Design Patterns: The Ten That Pay

> You do not need 23 patterns. You need ~10, plus the skill of *recognizing* which shape a problem has. Memorizing definitions is worthless; recognition is the whole thing.

---

## First: the recognition skill

Patterns are not things you apply. They are **names for solutions you will arrive at anyway** when you feel a specific pain. So learn the pains, not the diagrams.

| The pain you feel | The pattern it's asking for |
|---|---|
| "This `if`/`switch` on a type keeps growing, and every new case means editing the same function" | **Strategy** |
| "I can't test this because it builds its own database client inside" | **Dependency Injection** |
| "SQL is smeared through my business logic" | **Repository** |
| "Choosing which implementation to build is duplicated in five places" | **Factory** |
| "This third-party SDK's interface is ugly / I might replace this vendor" | **Adapter** |
| "I want logging/retry/caching/auth on many things without editing each one" | **Decorator / Middleware** |
| "When X happens, five unrelated things must react — and the list keeps growing" | **Observer / Pub-Sub** |
| "This constructor takes 11 arguments, most optional" | **Builder** |
| "I need to queue / retry / undo / audit an operation as a thing" | **Command** |
| "Five classes do the same 6 steps, differing in 2 of them" | **Template Method** |

**The recognition drill:** whenever you're about to write an `if` on a *type*, a `new SomethingConcrete()` inside business logic, or a `try/catch` you're about to copy-paste, stop and ask which row of that table you're in.

### The counter-rule (equally important)

> **A pattern is justified when it removes duplication or enables a change you can concretely name. It is not justified by "this feels more professional."**

Two implementations do not need a Strategy. One database does not need a Repository *interface* (it might still want a repository *class*). Applying a pattern costs indirection, and indirection costs every future reader a hop. The rule of thumb: **apply the pattern on the second or third variation, not the first.** Write the `if`. When it gets a third branch, refactor.

---

## 1. Dependency Injection 🔥 (the highest-value one)

**Problem it solves.** A class that constructs its own dependencies is welded to them. You can't test it without a real database, can't swap implementations, and can't see what it actually needs.

**When to use.** Essentially always, for anything with I/O — DB, HTTP client, clock, random, logger, config.

**When NOT to use.** Pure functions and value objects need no injection. Also: you do **not** need a DI *framework*. Passing constructor arguments is dependency injection.

**Simple example**
```python
# Before: untestable
class OrderService:
    def __init__(self):
        self.db = PostgresClient(os.environ["DB_URL"])   # welded
        self.mailer = SendGridClient(API_KEY)            # welded

# After: dependencies handed in
class OrderService:
    def __init__(self, orders: OrderRepository, mailer: Mailer, clock: Clock):
        self.orders, self.mailer, self.clock = orders, mailer, clock
```

**Real backend example.** Your HTTP handler receives a service; the service receives a repository and a payment gateway; `main()` wires the real ones, tests wire fakes. Testing "order placed on a Sunday" becomes a fake clock instead of a mocking framework and a prayer.

**Common mistake.** Injecting a service *locator* or a giant container object — `__init__(self, container)` — which hides dependencies again and defeats the entire point. Also: treating DI as "use a framework," then generating an unreadable wiring graph.

**In production systems.** Every mature backend has a composition root (`main.go`, `Program.cs`, `app_factory.py`) where the real world is assembled once and injected downward. This is also what lets you run the same code against Postgres in prod and an in-memory fake in CI.

---

## 2. Strategy 🔥

**Problem it solves.** One operation, several interchangeable algorithms, selected at runtime. Without it you get an ever-growing `switch` that violates "open for extension, closed for modification."

**When to use.** Multiple (3+) implementations of the same conceptual operation: payment providers, shipping cost calculators, pricing/discount rules, export formats, auth strategies, retry policies.

**When NOT to use.** Two branches that will never grow. `if is_premium` is not a Strategy — it's an `if`.

**Simple example**
```python
class ShippingStrategy(Protocol):
    def cost(self, order: Order) -> int: ...

class StandardShipping:  
    def cost(self, o): return 500
class ExpressShipping:   
    def cost(self, o): return 1500
class FreeOverThreshold: 
    def cost(self, o): return 0 if o.total >= 5000 else 500

def checkout(order, shipping: ShippingStrategy):
    return order.total + shipping.cost(order)
```

**Real backend example.** `PaymentGateway` with Stripe/PayPal/bank-transfer implementations. Adding a new provider = adding a class + a registry entry. No existing file changes; nothing existing can regress.

**Common mistake.** Creating a Strategy interface with exactly one implementation "for the future." That future usually doesn't arrive, and when it does, the real requirement never fits the interface you guessed.

**In production systems.** Strategy + Factory is the standard plugin architecture: `PROVIDERS = {"stripe": StripeGateway, "paypal": PayPalGateway}` and the config picks one. It's also how feature flags often work under the hood.

---

## 3. Repository 🔥

**Problem it solves.** Data access logic spread through business logic. Business rules become untestable without a database, and swapping or optimizing storage means touching everything.

**When to use.** Any non-trivial application with persistence. Keep all SQL/ORM calls behind repository methods named in *domain* language: `find_active_by_email`, not `execute(sql)`.

**When NOT to use.** Tiny CRUD scripts. Also beware the anemic version: a repository that just forwards the ORM one-to-one adds a layer and buys nothing.

**Simple example**
```python
class OrderRepository(Protocol):
    def get(self, id: int) -> Order | None: ...
    def save(self, order: Order) -> None: ...
    def find_pending_older_than(self, cutoff: datetime) -> list[Order]: ...

class PostgresOrderRepository:   # the real one
class InMemoryOrderRepository:   # the test one
```

**Real backend example.** `OrderService.cancel()` calls `repo.get()` and `repo.save()`. Unit tests use the in-memory repo and run in microseconds; a smaller set of integration tests exercises the Postgres implementation to prove the SQL is right.

**Common mistake.** Leaking persistence concerns through the interface — returning ORM query builders, exposing `session`, or having `find_all()` and filtering in memory (an instant scalability bug). Another: making the repository do business logic.

**In production systems.** It's what allows "we moved this table to a separate service / added a caching layer / switched to a read replica" to be a one-file change. The caching decorator in §6 wraps a repository precisely because of this boundary.

---

## 4. Factory 🔥

**Problem it solves.** Object creation that involves conditionals, configuration, or multi-step setup, duplicated across call sites.

**When to use.** When "which implementation?" or "how do I build this correctly?" is a real decision — and especially when that decision appears in more than one place.

**When NOT to use.** When a constructor is enough. `UserFactory.create(name)` that just calls `User(name)` is noise.

**Simple example**
```python
def payment_gateway(provider: str, cfg: Config) -> PaymentGateway:
    match provider:
        case "stripe": return StripeGateway(cfg.stripe_key, timeout=5)
        case "paypal": return PayPalGateway(cfg.paypal_id, cfg.paypal_secret)
        case _: raise UnsupportedProvider(provider)
```

**Real backend example.** Building a storage client from config (`s3://` vs `file://` vs in-memory for tests) — one factory, and the rest of the app only knows the interface.

**Common mistake.** Abstract-factory-of-factories astronautics. In 95% of backend code, a **function that returns an interface** is all the factory you need.

**Factory vs Builder:** Factory answers *"which object?"*; Builder answers *"how do I assemble this complicated object step by step?"*

**In production systems.** The composition root is essentially one big factory. Every `NewX(cfg)` constructor in Go, every `create_app()` in Python, is this pattern.

---

## 5. Adapter 🔥

**Problem it solves.** An external interface doesn't match what your code wants — or you don't want your core coupled to a vendor.

**When to use.** Wrapping third-party SDKs, legacy APIs, or anything you might replace. Also for making two subsystems with different vocabularies talk.

**When NOT to use.** Wrapping a stable standard library for no reason. And don't build an adapter so generic it becomes a lowest-common-denominator API that hides the one feature you actually needed.

**Simple example**
```python
class SmsSender(Protocol):
    def send(self, to: str, body: str) -> None: ...

class TwilioAdapter:                    # adapts Twilio's API to our interface
    def __init__(self, client): self._c = client
    def send(self, to, body):
        self._c.messages.create(to=to, from_=self.number, body=body)
```

**Real backend example.** You start on Twilio and later move to SNS for cost. Because everything depends on `SmsSender`, the migration is one new adapter class plus a config change — not a search-and-replace across 40 files.

**Common mistake.** Letting vendor types leak through the adapter (returning a `TwilioMessage` object), which re-couples everything you were trying to decouple.

**Adapter vs Decorator:** Adapter **changes the interface**, same behavior. Decorator **keeps the interface**, adds behavior. If the wrapper implements the *same* interface it wraps, it's a decorator.

**In production systems.** This is the "ports and adapters" idea in its useful, non-dogmatic form: your domain defines the port (interface), infrastructure provides adapters.

---

## 6. Decorator / Middleware 🔥

**Problem it solves.** Cross-cutting concerns — logging, metrics, auth, retry, caching, rate limiting — that you'd otherwise copy into every function.

**When to use.** When behavior should apply to many operations uniformly and is orthogonal to what those operations do.

**When NOT to use.** When the added behavior needs to know the specifics of what it wraps. Also, deep decorator stacks make stack traces and debugging miserable — keep them shallow and ordered deliberately.

**Simple example**
```python
class CachedUserRepository:          # same interface as UserRepository
    def __init__(self, inner, cache): self.inner, self.cache = inner, cache
    def get(self, id):
        if (hit := self.cache.get(f"user:{id}")) is not None: return hit
        user = self.inner.get(id)
        self.cache.set(f"user:{id}", user, ttl=60)
        return user
```

**Real backend example.** Every HTTP framework's middleware chain *is* this pattern: `recover → request_id → logging → metrics → auth → rate_limit → handler`. Order matters enormously — request ID must be outermost so every later log line has it; auth must precede anything that reads the user.

**Common mistake.** Order-dependent bugs (rate limiting before auth means unauthenticated users consume authenticated users' quota), and swallowing errors inside a decorator so the caller never learns something failed.

**In production systems.** Retry wrappers, circuit-breaker wrappers, tracing spans, and cache layers are almost always decorators. This is how you add resilience to an existing client without touching it.

---

## 7. Observer / Pub-Sub 🔥

**Problem it solves.** One thing happens; N unrelated things must react; and the list of reactions changes over time. Without it, `place_order()` grows to 300 lines of unrelated concerns.

**When to use.** Domain events with multiple independent consumers: `OrderPlaced` → send email, update inventory, notify analytics, award loyalty points.

**When NOT to use.** When you need the result back, need it to happen transactionally, or need a guaranteed order. In-process observers also make control flow hard to trace — don't use events for things that are really just a function call.

**Simple example**
```python
bus.subscribe("order.placed", send_confirmation_email)
bus.subscribe("order.placed", decrement_inventory)
bus.subscribe("order.placed", track_analytics)

def place_order(order):
    repo.save(order)
    bus.publish("order.placed", OrderPlaced(order.id))
```

**Real backend example.** The same shape scales from an in-process event bus to Kafka/RabbitMQ/SNS+SQS. The pattern is identical; the delivery guarantees are not.

**Common mistake — and it's a big one.** Publishing the event *inside* the database transaction: if the transaction rolls back, you've already sent the email; if the broker is down, you've committed but lost the event. **The fix is the outbox pattern:** write the event into an `outbox` table in the same transaction as the business change, and have a separate worker publish it and mark it sent. Learn this — it comes up constantly in system design interviews.

**In production systems.** Event-driven architecture, CDC pipelines, webhooks, and audit logs all rest on this. Also note: consumers must be **idempotent**, because at-least-once delivery means duplicates are normal.

---

## 8. Builder 🟡

**Problem it solves.** Constructing an object with many optional parameters, or requiring multi-step assembly with validation at the end.

**When to use.** Complex configuration objects, query builders, test data factories.

**When NOT to use.** When your language has named/default arguments (Python, Kotlin, C#) — those solve the "telescoping constructor" problem natively and more readably.

**Simple example**
```java
HttpClient client = HttpClient.builder()
    .timeout(Duration.ofSeconds(5))
    .retries(3)
    .baseUrl("https://api.example.com")
    .build();               // validation happens here
```

**Real backend example.** **Test data builders** are the highest-value use: `anOrder().withStatus(PAID).withItems(3).build()` makes tests readable and removes the "update 40 tests because the constructor changed" problem.

**Common mistake.** A builder that allows an invalid object to be built — the `build()` method must validate required fields.

---

## 9. Command 🟡

**Problem it solves.** You need an operation to be a *value*: queued, retried, logged, audited, scheduled, or undone.

**When to use.** Background jobs, task queues, undo/redo, audit trails, batching.

**Simple example**
```python
@dataclass
class SendInvoiceEmail:
    order_id: int
    attempt: int = 0
    def execute(self, deps): ...
```

**Real backend example.** Every job queue is this pattern: the job payload is a serialized command, the worker deserializes and executes it. Because it's data, it can be persisted, retried with backoff, and inspected in a dead-letter queue.

**Common mistake.** Putting non-serializable things (open connections, live objects) into the command. Commands must be pure data plus an ID; dependencies get injected at execution time. Second mistake: commands that aren't idempotent, so a retry double-charges someone.

---

## 10. Template Method 🟡

**Problem it solves.** Several processes share the same skeleton and differ in a couple of steps.

**When to use.** ETL/import pipelines: `validate → parse → transform → persist → report`, where only parsing differs per file format.

**When NOT to use.** When inheritance is the only way you can express it and the hierarchy is getting deep. **Prefer the composition version**: pass the varying steps in as functions/strategies. Template Method is the inheritance-flavored sibling of Strategy, and Strategy usually ages better.

**Simple example**
```python
class Importer(ABC):
    def run(self, f):                     # the fixed skeleton
        rows = self.parse(f)              # the varying step
        valid = [r for r in rows if self.validate(r)]
        self.persist(valid)
        return Report(len(rows), len(valid))
    @abstractmethod
    def parse(self, f): ...
```

**Common mistake.** The fragile base class: base-class changes silently break subclasses, and subclasses start overriding steps they weren't meant to.

---

## Software engineering principles that matter more than patterns 🔥

- **Single Responsibility** — a module should have one reason to change. The practical test: *can you describe what it does without saying "and"?*
- **Dependency Inversion** — depend on abstractions, and let the *high-level* policy define the interface, not the low-level detail. This is why the `Mailer` interface belongs next to your domain code, not in the SendGrid package.
- **Composition over inheritance** — inheritance couples you to a parent's implementation forever; composition lets you rearrange. Use inheritance for genuine "is-a" with stable behavior; use composition for everything else.
- **Separation of concerns / layering** — handler (HTTP) → service (business rules) → repository (persistence). HTTP types must not reach the service; SQL must not reach the handler.
- **Make the change easy, then make the easy change.** (Kent Beck.) Refactor first, then add the feature — two commits, not one tangle.
- **YAGNI** — the abstraction you build for a requirement you imagined will not fit the requirement you get.
- **Explicit over implicit** — magic saves typing today and costs debugging forever.

---

## Practice: recognition drills

For each scenario, name the pattern(s) and say what you'd give up. Answers in [14 — Mastery Checkpoints](./14-mastery-checkpoints.md).

1. Your `export()` function has a 6-branch `if format ==` and product wants two more formats.
2. You want to log and time every repository call without editing any repository.
3. The notification service should send email, push, and SMS — and marketing wants to add Slack next month.
4. `OrderService` can't be unit tested because it opens its own DB connection.
5. You're moving from Stripe to Adyen and want the blast radius to be one package.
6. `createUser()` takes 9 parameters, 6 optional, and every test passes `None` six times.
7. Failed webhook deliveries must be retried with backoff for 24 hours, then dead-lettered.
8. Three report generators share the same 5 steps but differ in the data source.
9. When a user upgrades their plan, six subsystems need to know, and the list grows monthly.
10. Your service layer has raw SQL strings in it, and the team wants to add a caching layer.
