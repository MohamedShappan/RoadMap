# 15 — The Final 80/20 Cheat Sheet

> The smallest knowledge set with the largest practical payoff. If you can explain every line here with a mechanism *and* a trade-off, you are a strong backend engineer.

---

## Database — Top 15

1. **Index** — a sorted B-tree making lookups ~O(log n); costs write time and storage. Not free speed; a read/write trade.
2. **Composite index + leftmost prefix** — `(a,b,c)` serves `a`, `a,b`, `a,b,c` — never `b` alone. Equality columns first, range/sort column last.
3. **`EXPLAIN ANALYZE`** — seq scan on a big table, estimated-vs-actual row skew, and `Rows Removed by Filter` are your three tells.
4. **ACID** — atomicity, consistency, isolation, durability. A transaction turns *n* statements into one all-or-nothing unit.
5. **Isolation levels** — READ COMMITTED (per-statement snapshot, Postgres default) vs REPEATABLE READ (per-transaction snapshot, MySQL default). Anomalies: dirty, non-repeatable, phantom, lost update.
6. **Deadlock** — a cycle in lock waits. Prevent with consistent lock ordering, short transactions, no I/O inside transactions, and a retry.
7. **N+1 query** — 1 query + N queries in a loop. The most common real performance bug. Fix with a join, eager load, or batched `IN`.
8. **Keyset pagination** — `WHERE (created_at,id) < (...)` instead of `OFFSET`; constant time at any depth.
9. **Connection pooling** — connections are expensive; pool size ~10–30 per instance, not 500. **Pool exhaustion looks like "the app is slow," not like a DB error.**
10. **Normalization to 3NF** — store each fact once. **Denormalize** only with a measurement, or for historical snapshots (order line prices).
11. **Constraints** — `NOT NULL`, `CHECK`, `FK`, `UNIQUE`. The database is the last line of defense, and `UNIQUE` is how you make an operation idempotent under concurrency.
12. **Read replicas** — scale reads, not writes; introduce replication lag; handle read-your-own-writes.
13. **Partitioning vs sharding** — partition = chunks within one DB (great for data lifecycle: drop a partition instantly). Shard = across DBs (scales writes; costs joins, transactions, and resharding pain).
14. **Cache-aside** — read: check cache, miss → DB → populate. Write: **delete** the key, don't update it. Guard against stampedes.
15. **Choosing a store** — Postgres by default; Redis for hot reads/counters/locks/sessions; Elasticsearch for relevance search (always a secondary index); a document store only for genuinely document-shaped data.

---

## Networking — Top 15

1. **The request lifecycle** — DNS → TCP → TLS → HTTP → CDN → LB → app → cache → DB → response. Narrate it in five minutes.
2. **TCP** — handshake (1 RTT), ordered, reliable, congestion-controlled, head-of-line blocked. **UDP** — no guarantees, no setup; use when late data is worthless.
3. **A connection is a 4-tuple** `(src_ip, src_port, dst_ip, dst_port)`. Ephemeral port exhaustion is real; reuse connections.
4. **DNS** — hierarchical caching governed by TTL. Lower the TTL *before* a migration.
5. **TLS** — confidentiality + integrity + identity. 1–2 extra RTTs; amortize with keep-alive and resumption. Expired certs are the #1 self-inflicted outage.
6. **HTTP verb properties** — GET/HEAD safe+idempotent; PUT/DELETE idempotent; **POST is neither** → POST needs idempotency keys.
7. **Status codes** — 201+Location, 202 queued, 400 vs 401 vs 403 vs 404 vs 409 vs 422, 429+`Retry-After`.
8. **502 / 503 / 504** — bad answer / no backend available / no answer in time.
9. **Connection refused vs timeout** — actively rejected (routing worked, nothing listening or firewall *reject*) vs silently dropped (firewall *drop*, wrong IP, host down).
10. **Reverse proxy / LB** — TLS termination, routing, health checks, rate limiting, hiding topology. L4 vs L7.
11. **Latency numbers** — memory ~100ns, same-DC RTT ~0.5ms, indexed query ~1–10ms, cross-continent ~150ms. Distance is physics.
12. **Timeouts on every outbound call** — a missing timeout is the #1 cause of cascading failure.
13. **Retries with exponential backoff + jitter**, bounded, idempotent-only, with a retry budget. Naive retries create retry storms.
14. **HTTP/1.1 → /2 → /3** — one-at-a-time per connection → multiplexed over TCP (still TCP HOL-blocked) → multiplexed over QUIC/UDP (independent streams).
15. **`curl -v` and `curl -w`** — per-phase timings (DNS / connect / TLS / TTFB) localize a slow request to a layer in one command.

---

## Design Patterns — Top 10

1. **Dependency Injection** — pass dependencies in; makes everything testable. No framework required.
2. **Strategy** — interchangeable algorithms behind one interface. The answer to a growing `switch` on a type.
3. **Repository** — persistence behind a domain-language interface; the seam that lets you add caching or swap stores.
4. **Factory** — centralize conditional construction. Usually just a function returning an interface.
5. **Adapter** — make an external interface fit yours. *Changes the interface, same behavior.*
6. **Decorator / middleware** — layer cross-cutting behavior. *Same interface, adds behavior.* Order matters.
7. **Observer / pub-sub** — one event, many independent reactions. Pair with the **outbox pattern** so events can't be lost or phantom-published.
8. **Command** — an operation as data, so it can be queued, retried, and dead-lettered. Handlers must be idempotent.
9. **Builder** — complex assembly with validation at `build()`. Highest real value: test data builders.
10. **Recognition over memorization** — read the pain, name the shape; apply on the second or third variation, never the first. A pattern must remove duplication or enable a change you can name.

---

## System Design — Top 20

1. **Estimate first** — QPS (avg and peak ≈ 2–3×), storage/year, bandwidth, read:write ratio. End with "the hard dimension is X."
2. **Stateless app tier** — the enabling constraint for horizontal scaling and safe restarts. Push state to Redis/S3/queue.
3. **Vertical before horizontal** — bigger machines are cheap engineering time and correct for longer than people admit.
4. **Cache layers** — browser → CDN → gateway → app memory → Redis → DB buffer pool. A cache must always be optional.
5. **Read replicas** — read scale, replication lag, read-your-own-writes.
6. **Queues + async workers** — the single most useful scaling move. Buys latency, resilience, burst absorption, independent scaling.
7. **Idempotency** — the answer to "the network lost my response." UNIQUE key + stored response.
8. **Timeouts, retries + backoff + jitter, circuit breakers, bulkheads** — the four resilience primitives.
9. **Backpressure & load shedding** — an unbounded queue is a delayed outage. Say no with 429/503 instead.
10. **Rate limiting** — token bucket in Redis, keyed per user/tenant, 429 + `Retry-After`.
11. **CAP, practically** — during a partition, choose C or A **per operation**. The rest of the time you're trading consistency for latency.
12. **Consistency models** — strong / read-your-own-writes / eventual. Pick per feature, never per system.
13. **Sharding** — last resort; the shard key decides everything; you lose joins, transactions, and easy resharding.
14. **Fan-out on write vs read** — precomputed feeds vs merge-at-read; the real answer is hybrid because of celebrities.
15. **Saga + compensating transactions** — how you replace a distributed transaction. Not atomic; compensations must be idempotent.
16. **Outbox pattern** — write the event in the same local transaction as the change; a relay publishes it. Learn this one properly.
17. **Object storage + presigned URLs** — bytes never traverse your app. Never store files in the database or on a local disk.
18. **CDN** — cut latency (physics), offload origin, absorb spikes. Cache-bust with content-hashed filenames.
19. **Single points of failure** — name them explicitly in every design, then remove or consciously accept them.
20. **Observability is part of the design** — say which 3–4 metrics you'd alert on and how you'd debug a reported symptom.

---

## Backend — Top 15

1. **REST design** — noun resources, correct verbs, correct status codes, cursor pagination, allow-listed sort fields.
2. **Validate at the boundary** into typed structures; schema → business rules → DB constraints.
3. **A consistent error contract** — stable machine-readable `code` + `request_id`; never leak internals.
4. **Expected vs unexpected errors** — 4xx aren't alerts; 5xx are. Conflating them creates alert fatigue.
5. **AuthN vs AuthZ** — and **IDOR** is the most common real vulnerability: check ownership per resource, in the service layer.
6. **Sessions vs JWT** — sessions revoke instantly; JWTs don't. Short-lived access token + revocable refresh token is the usual right answer.
7. **Hash passwords with bcrypt/argon2** — slow by design. Never MD5/SHA.
8. **Layering** — handler (HTTP) → service (business rules + transactions) → repository (SQL). Nothing leaks across.
9. **Transaction boundaries in the service**, short, and **never containing network I/O**.
10. **Optimistic locking, pessimistic locking, atomic single-statement updates** — know when each applies.
11. **Background jobs** — anything the user doesn't need synchronously. Idempotent handlers, backoff, DLQ, queue-depth alerting.
12. **Config from environment, validated at startup**; secrets from a manager, never in git, never in logs.
13. **Testing shape** — many unit tests with fakes, integration tests against a **real** database, a few E2E. Test the failure paths.
14. **Structured logs with a request ID on every line**, propagated downstream.
15. **API versioning** — `/v1` for breaking changes only; prefer additive changes and support N-1.

---

## Production Engineering — Top 15

1. **Four golden signals** — latency, traffic, errors, saturation. Instrument these per service and you can run a system.
2. **Percentiles, not averages** — p99 is the user having a bad time. Never average percentiles.
3. **Metrics → traces → logs** — is something wrong, where, and why.
4. **Correlation IDs everywhere**, propagated across every hop.
5. **Liveness vs readiness** — liveness must check nothing external, or a dependency blip becomes a cluster-wide restart storm.
6. **Graceful shutdown** — fail readiness, drain, finish in-flight, close pools. Without it, every deploy drops requests.
7. **Timeouts on every outbound call.** (It's on this list twice for a reason.)
8. **Circuit breakers** — closed → open → half-open. Fail fast so a dead dependency can recover.
9. **Graceful degradation** — decide in advance what you'd shed.
10. **Docker + compose** — multi-stage builds, non-root, no secrets in images, one-command local stack.
11. **CI on every push; build once, promote the same artifact.**
12. **Backward-compatible migrations (expand–contract)** — never rename or drop a column in the same deploy as the code change.
13. **Feature flags** — decouple deploy from release; ship dark, ramp, kill instantly. Remove them afterwards.
14. **Alert on symptoms, not causes.** SLOs and error budgets turn arguments into arithmetic.
15. **Security baseline** — parameterized queries, per-resource authorization, TLS everywhere, least privilege, secret management, dependency scanning, rate-limit anything that costs money.

---

## The five sentences that summarize everything

1. **Most performance problems are database problems, and most database problems are missing indexes or N+1 queries.**
2. **The network loses responses, so every unsafe operation needs to be idempotent and every call needs a timeout.**
3. **Statelessness is what makes horizontal scaling possible; everything else follows from it.**
4. **Every design decision is a trade-off — if you can't name what you gave up, you haven't made a decision, you've made an assumption.**
5. **You can't fix what you can't see: logs, metrics, and traces are part of the system, not an add-on.**
