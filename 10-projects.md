# 10 — The Project Sequence

> Six projects. Each one forces concepts the previous one didn't. **Resist scope creep** — an unfinished ambitious project teaches far less than a finished modest one.

**Rules for all projects:**
- Same language, PostgreSQL, Redis. Don't diversify.
- Every project: README with setup, `docker compose up` works, tests pass in CI.
- Ship it in a fixed time box. When the box ends, write down what you'd do next and move on.
- **Write a short "what I learned and what I'd do differently" doc for each.** This is what you'll talk about in interviews, and the reflection is where the learning consolidates.

---

## Project 1 — Production-Quality REST API
**After Stage 3 · ~3 weeks · Domain: a task/project manager (users, projects, tasks, comments)**

| Aspect | What to build |
|---|---|
| **Requirements** | Register/login, CRUD projects and tasks, assign tasks, comment, list with filters + pagination. Users only see their own projects. |
| **Architecture** | Single service. `handler → service → repository`. Dependency injection at a composition root. Config from environment. |
| **Database** | Postgres. Proper FKs, `NOT NULL`, `CHECK` on status enums, unique on email. Versioned migrations. Indexes justified by actual queries. |
| **APIs** | REST with correct status codes, cursor pagination, a consistent error envelope with `request_id`, request validation at the boundary. |
| **Caching** | None yet — *deliberately*. You'll add it in Project 2 and feel the difference. |
| **Networking** | Understand every status code you return. Set timeouts on any outbound call. |
| **Patterns** | Repository, DI, middleware chain (request-ID → logging → auth → error mapping). |
| **Testing** | Unit tests for service logic with a fake repository; integration tests against real Postgres in Docker; a few API-level happy-path tests. |
| **Scaling** | Stateless (sessions in Redis or JWT). Note where it would break first. |
| **Failure** | What happens if the DB is down? Return 503, not a stack trace. |
| **Observability** | Structured JSON logs with request IDs; a request-duration log line per request. |

**Done when:** someone else can clone, `docker compose up`, and add an endpoint using only the README.

---

## Project 2 — E-commerce Backend
**After Stage 4–5 · ~4 weeks · The project that teaches transactions and caching for real**

| Aspect | What to build |
|---|---|
| **Requirements** | Catalog with categories, search/filter, cart, checkout, orders, inventory, order history, admin product management. |
| **Architecture** | Still one service. Add a background worker process. Introduce Strategy for payment providers and shipping cost, Adapter for the payment SDK. |
| **Database** | The schema from [03](./03-databases.md). Critically: **`order_items` snapshots price and product name**. Inventory updated with an atomic conditional `UPDATE`. Seed with 100k+ products and 1M+ order rows so performance is real. |
| **APIs** | Faceted product listing (filter by category/price/availability), cart operations, `POST /orders` with an **idempotency key**. |
| **Caching** | Cache-aside in Redis for product detail and category listings. Invalidate on admin update. Deliberately create a staleness bug, then fix it. Measure the before/after latency. |
| **Networking** | Integrate a real payment sandbox (Stripe test mode). Timeouts, retries, and handling "we called them and never heard back." |
| **Patterns** | Strategy (payment, shipping), Factory (provider selection), Decorator (caching repository), Observer (order events). |
| **Testing** | Concurrency test: two simultaneous checkouts for the last item — exactly one must win. This test is the heart of the project. |
| **Scaling** | Read replica for catalog reads (Docker). Observe and handle replication lag. |
| **Failure** | Payment provider down/slow/timeout. Inventory insufficient mid-checkout. Cart abandoned with reserved stock. |
| **Observability** | Metrics: checkout success rate, payment latency, cache hit rate, DB pool usage. |

**Done when:** you can survive a load test at checkout without overselling a single item, and you can explain each index in the schema.

---

## Project 3 — Notification System
**After Stage 5 · ~2 weeks · The purest "queues and reliability" project**

| Aspect | What to build |
|---|---|
| **Requirements** | An internal API: "notify user X about event Y." Multi-channel (email, SMS, push — providers can be fakes that log). User preferences and quiet hours. Templates with variables. Delivery status tracking. |
| **Architecture** | API service + per-channel queues + channel workers. This is where you feel *why* per-channel isolation matters. |
| **Database** | `notifications`, `templates`, `user_preferences`, `delivery_attempts`. Idempotency key unique per logical notification. |
| **APIs** | `POST /notifications → 202 Accepted` with a notification ID. `GET /notifications/{id}` returns delivery status per channel. |
| **Caching** | Cache user preferences (read on every send). |
| **Networking** | Provider calls with timeouts; distinguish permanent failures (invalid address → don't retry) from transient (5xx → retry). |
| **Patterns** | Command (the job payload), Strategy (per-channel sender), Adapter (each provider SDK). |
| **Testing** | Duplicate request → one send. Provider fails 3 times then succeeds → one send. Provider always fails → lands in DLQ. |
| **Scaling** | Scale workers independently per channel. Batch where providers support it. |
| **Failure** | **The core of this project.** Retry with backoff, dead-letter queue, poison messages, per-user rate limiting so a bug can't send 10,000 emails to one person. |
| **Observability** | Queue depth, oldest-message age, per-channel success rate, DLQ size with an alert on it. |

**Done when:** you can kill a provider mid-run, restart it, and end with every notification delivered exactly once.

---

## Project 4 — URL Shortener (at scale)
**After Stage 5–6 · ~1.5 weeks · Small surface area, so you can focus entirely on performance**

| Aspect | What to build |
|---|---|
| **Requirements** | Create short URLs, redirect, custom aliases, expiry, click analytics. |
| **Architecture** | Tiny API + Redis + Postgres + an analytics worker. |
| **Database** | One table, one hot index. Base62-encoded IDs. |
| **APIs** | `POST /urls`, `GET /{code}` → 302. |
| **Caching** | Aggressive — URLs are immutable, so long TTLs. Target a >95% cache hit rate. |
| **Networking** | 301 vs 302 and the analytics consequence. Cache headers. |
| **Testing** | Collision handling; concurrent custom-alias creation (the UNIQUE constraint is the correct answer). |
| **Scaling** | **The point of the project:** load test to 5,000+ rps on your laptop. Profile. Remove every bottleneck you find. Write down the numbers at each step. |
| **Failure** | Redis down → still works from Postgres, slower. Prove it by killing Redis under load. |
| **Observability** | Cache hit rate, p50/p95/p99 redirect latency. |

**Done when:** you have a written performance report with before/after numbers for at least three optimizations.

---

## Project 5 — Distributed Order Processing System
**After Stage 6–7 · ~4 weeks · The capstone. This is the one you'll talk about in interviews.**

| Aspect | What to build |
|---|---|
| **Requirements** | An order goes through: created → payment authorized → inventory reserved → fulfilled → shipped. Any step can fail and must compensate. Everything must be observable and exactly-once from the customer's perspective. |
| **Architecture** | 3–4 small services (order, payment, inventory, notification) communicating via a queue/broker. **This is the one time you build microservices — so you learn what they actually cost.** |
| **Database** | A database per service. No shared tables — this is the constraint that forces everything interesting. |
| **APIs** | `POST /orders` → `202` + order ID; `GET /orders/{id}` shows the current state of the saga. |
| **Patterns** | **Saga** with compensating transactions. **Outbox pattern** so events are never lost or phantom-published. Command objects for jobs. |
| **Caching** | Minimal. Don't let it distract from the point. |
| **Networking** | Inter-service timeouts, retries, circuit breakers. Simulate a service being slow, then down. |
| **Testing** | Kill a service mid-saga and verify the system converges to a correct state. Replay a duplicate event and verify nothing double-processes. |
| **Scaling** | Scale one service independently and show that you can. |
| **Failure** | **The whole project.** Payment succeeds but inventory fails → refund. Service crashes after committing but before publishing → the outbox saves you. Duplicate event delivery → idempotent consumers. |
| **Observability** | Distributed tracing across all services with a propagated trace ID. Reconstruct the full lifecycle of one order from traces alone. |

**Done when:** you can kill any single service at any point and the system eventually reaches a correct, explainable state — and you can *show* that in a trace.

---

## Project 6 — Large-Scale System Design Document
**After Stage 7 · ~1 week · No code. A written design.**

Pick one of: a social feed at 50M DAU, a ride-hailing backend, a video platform, or a global chat system.

Produce a document containing:
1. Requirements (functional, non-functional) and explicit out-of-scope
2. Capacity estimates with worked arithmetic, ending in "the hard dimension is X"
3. API definitions for the core operations
4. Data model with chosen datastores and *why*
5. Architecture diagram
6. The bottleneck at 1×, 10×, and 100× scale
7. Scaling strategy: caching, replication, partitioning/sharding with a named shard key
8. Async/event flows, including where consistency becomes eventual and whether that's acceptable
9. Failure analysis: every component — what happens when it's down, slow, or partitioned
10. Observability plan: metrics, SLOs, alerts, and how you'd debug a specific reported symptom
11. **Trade-offs section: 5 decisions, the alternative, and what you gave up**
12. "What I'd do differently at 10× scale"

**Done when:** you could present it in 45 minutes and defend every box against "why not X?"

---

## What you have at the end

A portfolio that demonstrates, in order: clean API engineering → transactional correctness under concurrency → async reliability → performance work with measurements → distributed systems → architectural judgment. That progression is *exactly* what interviewers are trying to establish, and you'll have artifacts for each claim.
