# 08 — Backend Engineering: The Connective Tissue

> This is where databases, networking, and system design become an actual running service. Each section ends with **how it connects** to the other domains — that cross-domain reasoning is what interviewers and senior engineers listen for.

---

## 1. REST API design 🔥

**Why it exists:** you need a predictable contract so clients, caches, proxies, and monitoring all behave sensibly without knowing your business.

**The rules that matter:**
- **Resources are nouns, plural:** `/orders`, `/orders/42/items`. Verbs live in the HTTP method, not the path.
- **Correct status codes.** `201` + `Location` on create. `202` when you queued the work. `204` on delete. `409` on a conflicting state. `422` when the body parses but is semantically invalid.
- **Pagination:** cursor/keyset for anything that can grow (`?limit=20&cursor=...`). Offset only for small, bounded lists. This is a direct consequence of [03's pagination section](./03-databases.md) — the API shape is driven by the database's cost model.
- **Filtering and sorting** via query params, with an **allow-list** of sortable fields (otherwise you've given users the ability to trigger unindexed sorts — a free denial-of-service).
- **Consistent envelopes:** decide once whether list responses are `{data: [], next_cursor}` and stick to it everywhere.
- **Nesting:** at most one level. `/users/42/orders` is fine; `/users/42/orders/7/items/3/refunds` is not — use `/refunds/3`.

**Connects to:** networking (status codes, idempotent verbs, caching headers), databases (pagination strategy, which filters are indexable), system design (statelessness makes horizontal scaling possible).

## 2. API versioning 🟡

**Why:** you cannot force mobile clients to update. Old versions live for years.
**How:** URL versioning (`/v1/`) is the pragmatic default — visible, cacheable, easy to route. Header versioning is purer and harder to debug.
**Better than versioning:** don't break things. Additive changes (new optional fields, new endpoints) never need a version bump. Reserve versions for genuine breaking changes, support N-1, and instrument usage so you know when it's safe to kill v1.

## 3. Validation 🔥

**Why:** every client input is hostile or buggy. Validation is also documentation.
- Validate at the **boundary**, once, into a typed structure. Business logic should receive validated data and never re-check.
- **Three layers, all needed:** schema validation (types, required, ranges) → business rules (in the service: "can't cancel a shipped order") → database constraints (the final guarantee).
- Reject unknown fields. Set max lengths and array sizes (an unbounded array field is a memory exhaustion attack).
- Never rely on client-side validation for anything.

**Connects to:** databases (constraints are the last line), security (injection, mass assignment).

## 4. Error handling 🔥

**Why:** errors are part of your API contract, and good error handling is most of what makes a service debuggable.

```json
{ "error": { "code": "insufficient_inventory", "message": "Only 2 units available",
             "details": {"sku": "ABC", "requested": 5, "available": 2},
             "request_id": "01HX..." } }
```
- **A stable machine-readable `code`** — clients must never parse the human message.
- **Distinguish expected from unexpected.** Expected domain failures (validation, not found, conflict) are 4xx and are *not* logged as errors. Unexpected failures are 5xx, logged with a stack trace and an alert. Conflating them creates alert fatigue, and alert fatigue creates outages.
- **Never leak internals** (stack traces, SQL, hostnames) to clients — but always attach the `request_id` so a user can report it and you can find the full detail in your logs.
- **Handle errors at one level**: one central error-mapping middleware, not `try/catch` in every function.
- **Never swallow an error silently.** The empty `catch` block is the single most expensive line of code in software.

## 5. Authentication 🔥

**AuthN = who are you. AuthZ = what may you do.** Confusing them is the root of most access-control bugs.

**Passwords:** hash with **bcrypt / argon2 / scrypt** (slow, salted by design). Never MD5/SHA — they're fast, which is exactly wrong. Never store recoverable passwords. Enforce length over character-class gymnastics, and check against known-breached lists.

**Sessions vs JWT — know this trade-off cold:**

| | Session (server-side) | JWT (stateless) |
|---|---|---|
| Where state lives | Redis/DB; cookie holds an opaque ID | In the token itself, signed |
| Revocation | **Instant** — delete the record | **Hard** — valid until expiry |
| Scaling | Needs a shared store (Redis is trivial) | No lookup needed |
| Size | Tiny cookie | Larger, sent on every request |
| Risk | Session store is a dependency | **Stale claims**: a user demoted 10 minutes ago still has admin |

**The honest guidance:** for a normal web app, **server-side sessions in Redis are simpler and safer** than people assume, and revocation alone often justifies them. Use JWTs for short-lived access tokens (5–15 min) paired with a **refresh token that is stored server-side and revocable** — that combination gets you statelessness on the hot path plus real revocation. Never put sensitive data in a JWT payload (it's base64, not encrypted). Always verify the signature *and* the algorithm (reject `alg: none`), the expiry, the issuer, and the audience.

**Cookies:** `HttpOnly` (JS can't read it), `Secure` (HTTPS only), `SameSite=Lax` (CSRF mitigation).

**OAuth2 / OIDC — conceptually only 🟡.** The authorization code flow with PKCE: your app redirects the user to the provider, the user authenticates *there*, the provider redirects back with a short-lived code, and your **backend** exchanges that code for tokens. The point: **the user's password never touches your system**, and you receive scoped, revocable access. OAuth2 is authorization; OIDC adds an identity layer (the ID token). Know why the implicit flow is deprecated (tokens in the URL). You do not need to implement this — you need to explain it.

## 6. Authorization 🔥/🟡

- **Check on every request, at the resource level.** The most common real vulnerability in web apps is **IDOR**: `GET /orders/1234` returning someone else's order because you checked "is logged in" but not "owns this order."
- Put authorization in the **service layer**, not the handler, so it can't be bypassed by a new entry point (background job, admin tool, GraphQL resolver).
- **RBAC** (roles) covers most needs; **ABAC** (attribute/ownership rules) for "can edit if owner or admin of the org."
- **Multi-tenancy:** every query must be scoped by `tenant_id`. Enforce it in the repository layer or via row-level security — never rely on each developer remembering.

## 7. Layered architecture & Dependency Injection 🔥

```
HTTP handler   → parse, validate, map to/from DTOs, set status codes.  Knows HTTP. Knows no SQL.
Service        → business rules, transaction boundaries, authorization, orchestration. Knows neither HTTP nor SQL.
Repository     → persistence. Knows SQL. Knows no business rules.
Domain model   → entities and invariants. Knows nothing external.
```
**The test:** could you expose the same service via a CLI or a queue consumer without changing it? If not, HTTP has leaked downward.

DI makes this testable (see [04](./04-design-patterns.md)). **Connects to:** patterns (Repository, DI), testing (fakes at layer boundaries), databases (transaction ownership).

## 8. Transactions in application code 🔥

- **One business operation = one transaction**, owned by the **service** layer, not the repository. If a repository method opens its own transaction, you can't compose two of them atomically.
- **Never do I/O inside a transaction** — no HTTP calls, no emails, no S3 uploads. You'll hold locks across the network and cause exactly the deadlock pileups from [03](./03-databases.md).
- **Do side effects after commit** — or, better, via the **outbox pattern**: write the event row inside the transaction, publish after.
- **Retry on serialization failures and deadlocks** with backoff; both are expected, not exceptional.
- Keep them **short**. Long transactions cause lock waits, bloat, and replication lag.

## 9. Concurrency 🔥/🟡

- **Optimistic locking** (a `version` column) for typical user-edit conflicts — cheap, no locks held.
- **Pessimistic locking** (`SELECT ... FOR UPDATE`) when contention is high and retrying is expensive.
- **Atomic single-statement updates** where possible — the safest option of all (`SET balance = balance - 100 WHERE balance >= 100`).
- **Distributed locks** (Redis `SET NX PX` + a unique token, released only by the owner) — useful, but understand they are *advisory* and can expire mid-work. Design so that a lost lock is survivable; prefer idempotency over locking where you can.
- **Bound your concurrency**: worker pool sizes, DB pool size, and per-dependency limits. Unbounded concurrency converts a traffic spike into a total outage.

## 10. Background jobs & queues 🔥

**The rule:** if the user doesn't need the result to render their response, it doesn't belong in the request.

Candidates: emails/notifications, report generation, image/video processing, third-party API calls, search indexing, webhooks, cleanup, analytics.

**Requirements for every job system:**
- **Idempotent handlers** — at-least-once delivery means duplicates *will* happen.
- **Retries with exponential backoff**, a max attempt count, and a **dead-letter queue**.
- **Visibility timeout** long enough for the work, so a slow job isn't re-delivered while still running.
- **Monitoring:** queue depth, oldest-message age, consumer lag, DLQ size. *(Queue depth growing steadily is the earliest warning sign you will ever get that something is wrong.)*
- **Separate queues by priority** so a bulk backfill can't delay a password reset.
- **Scheduled jobs** need a distributed lock or a leader, or every instance runs them simultaneously.

**Connects to:** system design (async processing, backpressure), patterns (Command), production (DLQ monitoring).

## 11. Caching in the application 🔥

Cache-aside with Redis, TTLs chosen from acceptable staleness, delete-on-write invalidation, stampede protection. (Full treatment in [03, Part 5](./03-databases.md).)

Application-specific notes:
- **Key naming:** `v1:user:42:profile` — include a version prefix so you can invalidate a whole class of keys by bumping it.
- **Cache the expensive thing, not the whole response** — usually the query result, not the serialized HTTP body (which ties caching to your API format).
- **Negative caching:** cache "not found" briefly, or a nonexistent-key flood becomes a DB attack.
- **The cache must be optional.** Wrap it in a try/catch: if Redis is down, serve from the database, slower. A cache outage must not be an application outage.

## 12. File & object storage 🟡

Store files in S3/GCS, metadata in the database. Upload and download via **presigned URLs** so bytes never pass through your app. Validate content type and size *server-side* (never trust the client's declared type). Generate unpredictable keys. Scan user uploads if you serve them back to other users, and serve untrusted content from a separate domain to contain XSS.

## 13. Rate limiting 🟡

Token bucket in Redis, keyed per user/API key/IP/tenant, returning `429` with `Retry-After`. Layer it: coarse at the edge (per IP, DDoS-ish), fine in the app (per user, per endpoint — a login endpoint deserves a much tighter limit than a read endpoint). **Connects to:** system design (backpressure), security (brute force protection), networking (429 semantics).

## 14. Configuration & secrets 🔥

- **12-factor:** config from the environment; the same artifact runs in every environment.
- **Validate config at startup and crash immediately if it's wrong.** Failing to boot is vastly better than failing at 3 a.m. on the one code path that reads a missing variable.
- **Secrets never in git.** Use a secret manager; rotate; never log them; scrub them from error reports.
- Keep a checked-in `.env.example` documenting every variable.

## 15. Logging & observability from the app's side 🔥

Structured (JSON) logs, one event per line, with `request_id`/`trace_id` on **every** line. Log at boundaries (request in/out, external call, job start/end), not in the middle of loops. Never log passwords, tokens, card numbers, or PII. `INFO` for business events, `WARN` for recovered problems, `ERROR` for things a human must look at. Full treatment in [09](./09-production-engineering.md).

## 16. Testing 🔥

**The shape that actually works for backends:**

| Level | What | Speed | Use it for |
|---|---|---|---|
| **Unit** | One class/function, fakes for I/O | ms | Business rules, edge cases, error paths — *lots* of these |
| **Integration** | Service + **real** Postgres/Redis in Docker | ~100ms | Repositories, SQL correctness, transactions, migrations |
| **API/E2E** | Real HTTP against the running app | seconds | Happy paths and auth on each endpoint — *a few* of these |

**Principles:**
- **Test behavior, not implementation.** A test that breaks when you rename a private method is a liability.
- **Use real databases in integration tests** (Testcontainers/Docker Compose). Mocked SQL proves nothing — the whole risk is in the SQL.
- **Prefer fakes over mocks.** An `InMemoryOrderRepository` is more readable and less brittle than five `when(...).thenReturn(...)` lines.
- **Each test creates its own data** and is order-independent. Shared fixtures rot.
- **Test the failure paths** — timeouts, duplicate requests, concurrent updates, partial failures. This is where real bugs live, and it's what most test suites completely omit.
- Coverage is a smoke detector, not a goal. 100% coverage of getters tells you nothing.

**Connects to:** DI (makes unit tests possible), databases (integration tests catch the bugs that matter), production (a test suite is what lets you deploy on a Friday).

---

## How it all connects — the one-paragraph version

A request arrives over **TCP+TLS** through a **load balancer** (networking), hits a **stateless** handler (system design) that **validates** input, **authenticates and authorizes** (backend), and calls a **service** that owns a **transaction** (databases) and talks to a **repository** (patterns) whose queries are backed by the right **indexes** (databases). Expensive reads come from a **cache** (system design); non-urgent work goes to a **queue** (system design) with **idempotent** handlers (reliability). Every step emits **structured logs and metrics** (production), every outbound call has a **timeout** (networking/reliability), and the whole thing is **tested** at the right level (engineering) so it can be **deployed** safely (production). *That single sentence is the entire roadmap.*
