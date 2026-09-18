# 14 — Mastery Checkpoints

> These are **not** definition quizzes. Each is a scenario where the right answer is a *process*: what you'd investigate, in what order, and what you'd rule out.

**How to use them:** answer out loud, with a timer (5–10 minutes each), before reading the "what a strong answer includes" notes. Being able to recognize a good answer is not the same as producing one.

**The grading rubric for every question:**
- 🥉 You name a plausible cause.
- 🥈 You name several causes **and how you'd distinguish between them**.
- 🥇 You give an **ordered investigation** (cheapest/most likely first), say what data you'd look at, and name the **trade-off** in your fix.

---

## Checkpoint 0 — Foundations

**0.1** Two concurrent requests both read a counter as 5, increment it, and write 6. You expected 7. Explain what happened and give three different fixes at three different layers.
> *Strong answer includes:* read-modify-write race. Fixes: (a) application-level lock — works within one process only; (b) database atomic update `SET n = n + 1` — correct and cheapest; (c) optimistic locking with a version column and retry; (d) `SELECT ... FOR UPDATE`. Note that (a) breaks the moment you run two instances — which is why this is really a *distributed* problem.

**0.2** A function takes 200ms for 1,000 records and 20 seconds for 10,000. What does that tell you, and where would you look?
> 10× data → 100× time means O(n²). Look for a nested loop, or a lookup inside a loop that should be a hash map — or an N+1 query.

---

## Checkpoint 1 — Databases

**1.1** *Your API suddenly became slow. Database CPU is at 90%. What do you investigate?*
> **This is the canonical question.** A strong answer is ordered:
> 1. **What changed?** Deploy, migration, traffic spike, a new feature, a dropped index, a batch job? Check the deploy timeline first — it's the highest-probability cause and the cheapest to check.
> 2. **Find the expensive queries** — `pg_stat_statements` sorted by total time (not mean time; a fast query called 100k times is the usual culprit). Check `pg_stat_activity` for long-running or blocked queries.
> 3. **Classify:** is it one new slow query, or the same queries now running more often (N+1 introduced, cache hit rate dropped, a traffic spike)? Check your cache hit rate — a cache that just started missing looks exactly like this.
> 4. **Check for lock waits** — high CPU with low throughput may be contention, not work.
> 5. **Check the plans** — `EXPLAIN ANALYZE` the top offenders. Did a plan flip because statistics went stale after a bulk load?
> 6. **Mitigate now, fix properly after:** rate limit or shed the offending endpoint, kill a runaway query, scale reads to a replica. Then add the index or fix the N+1.
> 🥇 also mentions: distinguishing "the DB is the cause" from "the DB is the victim" (a retry storm from the app can look identical), and that connection pool exhaustion presents as app slowness with *low* DB CPU — the opposite signature.

**1.2** A query using an index yesterday is doing a sequential scan today, with no code change. Why?
> Statistics went stale (bulk insert/delete → `ANALYZE`); the data distribution changed so the planner now estimates the index isn't selective enough; a parameter value is much less selective than before (parameter sniffing); the table grew past a threshold; or someone dropped/invalidated the index.

**1.3** You add an index and the query gets no faster. Give four explanations.
> Wrong column order for a composite predicate (leftmost prefix); a function or type cast on the column; the query was never scan-bound (it was lock-bound, pool-bound, or network-bound); the planner isn't using it (stale stats, low selectivity); or the bottleneck was returning a million rows, not finding them.

**1.4** Users report that a comment they just posted doesn't appear when the page reloads — but it appears a few seconds later. What's happening, and give two fixes with trade-offs.
> Replication lag: the write went to the primary, the reload read from a replica. Fixes: (a) route reads to the primary for N seconds after a user's write (sticky) — simple, adds primary load; (b) read the user's own data from the primary always; (c) return the created object from the write response and render optimistically — no extra load, but only fixes the immediate case; (d) track a write LSN/timestamp per user and only use a replica that has caught up — correct, more complex.

**1.5** Your `orders` table has 500M rows and `DELETE FROM orders WHERE created_at < '2020-01-01'` has been running for two hours, and now other queries are slowing down. What went wrong and what should you have done?
> A single huge DELETE holds locks, generates enormous WAL, bloats the table, and blocks vacuum. Right answers: delete in small batches with a sleep between them; or — the correct design — **partition by time so you can `DROP` a partition instantly.** Recovery now: cancel it, batch it, then `VACUUM`.

**1.6** A page shows 20 orders and issues 61 queries. Explain and fix.
> N+1 (probably two of them: order → customer, order → items). Fix with a join or eager loading, or batch with `WHERE id IN (...)` and map in memory. Prevent recurrence with a per-request query-count assertion in tests.

---

## Checkpoint 2 — Networking

**2.1** Your service returns 504 for 5% of requests. Everything looks healthy on dashboards. Walk me through it.
> 504 = the proxy timed out waiting on the backend. Ordered: is it one endpoint or all? (One → a specific slow query or dependency. All → a shared resource: pool, CPU, GC.) Check the **p99, not the average** — a 5% failure rate is invisible in a mean. Look for: an unindexed query on a rare-but-large record, connection pool exhaustion (check pool wait time), a downstream call with no timeout, lock contention, or GC pauses. Compare the LB's timeout to your app's own timeouts — if the app's timeout is longer than the proxy's, the proxy gives up first and you never see an error in your own logs, which explains the "everything looks healthy."

**2.2** Deploying a new version, you see a burst of 502s for ~30 seconds each time, then it's fine. Why?
> Old instances are terminated while still handling requests, or new instances receive traffic before they're ready. Fixes: graceful shutdown (fail readiness → drain → finish in-flight → exit), a readiness probe that's actually accurate, and connection draining at the load balancer.

**2.3** Service A calls service B. B becomes slow (10s per request instead of 100ms). Within two minutes, service A is completely down — even for endpoints that don't call B. Explain.
> Thread/connection pool exhaustion. Requests to B occupy all of A's workers, so unrelated requests can't get one. This is the classic cascading failure. Fixes: a timeout on the call to B (the root fix), bulkheads (separate pool per dependency), a circuit breaker so A fails fast, and a fallback/degraded response.

**2.4** `curl` from your laptop works. The same call from inside the container times out. Name four possible causes.
> Different DNS resolution inside the container (search domains, cluster DNS); a network policy/security group blocking egress; no route (the service is on a private network your laptop reaches via VPN but the container doesn't); the container resolving to `localhost` which is now the container itself, not your host; or a proxy env var set in one environment and not the other.

**2.5** Your team's retry logic turned a 30-second dependency blip into a 20-minute outage. How?
> A retry storm: every client retried immediately and simultaneously, multiplying load on a dependency that was already struggling, preventing it from recovering — and synchronized retries created repeated thundering herds. Fixes: exponential backoff **with jitter**, bounded attempts, a retry budget (cap retries at a percentage of traffic), and a circuit breaker.

---

## Checkpoint 3 — Backend Engineering

**3.1** *Your service receives the same payment request twice. How do you prevent duplicate processing?*
> Idempotency key with a **UNIQUE constraint**: the client sends a key, the server records `(key → response)` in the same transaction as the work, and returns the stored response for any duplicate. The UNIQUE constraint — not an application-level "check if exists" — is what makes the *concurrent* duplicate safe. 🥇 also covers: what the key should be scoped to (per user, per operation), TTL/retention of keys, what to do if the same key arrives with a *different* body (reject with 422 — it's a client bug), and the harder variant: "we called the payment provider and never got a response" → write an intent row before the call and reconcile by querying the provider with the same key.

**3.2** A user reports they were charged twice, and your logs show only one request. Where else could the duplicate come from?
> A retried webhook processed twice; a queue message redelivered after a visibility timeout expired mid-processing; a background job scheduled on every instance instead of one; a saga compensation that ran and then the original succeeded; or a client retry that hit a different instance behind the LB. The lesson: **idempotency must be at the consumer, not only at the HTTP edge.**

**3.3** Your integration tests pass; production breaks on a query. How is that possible, and what does it tell you about your test strategy?
> The tests mocked the database. Mocked SQL proves nothing — the entire risk lives in the SQL and the schema. Fix: integration tests against a real Postgres in Docker, with migrations applied, and realistic data volume for anything performance-sensitive.

**3.4** A new developer puts an HTTP call to a payment provider inside a database transaction. Explain every reason that's bad.
> The transaction is held open for the duration of a network call (seconds), holding locks → lock contention and deadlocks for everyone else; it blocks vacuum and bloats the table; a timeout leaves you unsure whether the charge happened while your transaction rolls back (money charged, no order recorded); and it couples your database's health to a third party's availability. Correct pattern: commit the local state first with the outbox pattern, then perform the external call in a worker, idempotently.

**3.5** You need to add a `NOT NULL` column to a 50M-row table used by a service that deploys continuously. Describe the safe sequence.
> Expand–contract: (1) add the column as nullable with no default rewrite; (2) deploy code that writes both old and new; (3) backfill in batches; (4) add the NOT NULL constraint (validate separately if the DB supports `NOT VALID` then `VALIDATE`); (5) deploy code that reads the new column; (6) later, remove the old one. Never combine a schema change and a code change that depend on each other in a single deploy — during a rolling deploy, both versions run simultaneously.

---

## Checkpoint 4 — Design Patterns (answers to the drills in [04](./04-design-patterns.md))

1. 6-branch export format switch → **Strategy** + **Factory** for selection. Cost: harder to see all formats in one place.
2. Log/time every repository call → **Decorator**. Cost: deeper stack traces.
3. Email/push/SMS/Slack notifications → **Strategy** per channel, chosen by a **Factory**; **Adapter** per provider SDK.
4. Untestable because it opens its own DB connection → **Dependency Injection**.
5. Stripe → Adyen with a small blast radius → **Adapter** behind an interface you own (plus Strategy if both run simultaneously).
6. 9-parameter constructor → **Builder**, or named/default arguments if your language has them. For tests specifically, a **test data builder**.
7. Retry webhooks with backoff then dead-letter → **Command** (the job as data) on a queue; the handler must be **idempotent**.
8. Three reports, same 5 steps, different data source → **Template Method**, or better, **Strategy** injected for the varying step (composition ages better).
9. Plan upgrade → six subsystems react, list growing → **Observer / pub-sub**, with the **outbox pattern** if the subscribers are other services.
10. Raw SQL in the service layer, want caching → **Repository** to create the boundary, then a caching **Decorator** on it.

**4.11** You've applied five patterns to a 200-line module and a colleague says it's over-engineered. How do you decide who's right?
> The test: for each abstraction, name the concrete change it enables or the duplication it removes. If you can't — "we might need it" doesn't count — remove it. Indirection has a real, permanent cost paid by every future reader.

---

## Checkpoint 5 — Scaling & Performance

**5.1** *Your database can no longer handle read traffic. What options do you have?*
> Ordered by cost/benefit: (1) **fix the queries** — indexes and N+1 elimination routinely give 10–100× for hours of work; (2) **cache** hot reads in Redis, with a stated invalidation strategy; (3) **scale vertically** — often the cheapest total-cost answer; (4) **read replicas**, accepting replication lag and handling read-your-own-writes; (5) **CDN/edge caching** if any of it is publicly cacheable; (6) **denormalize or precompute** expensive aggregates; (7) **partition** large tables; (8) **shard** — last, with a named shard key and a clear statement of what you lose. 🥇 states the trade-off at each step and says which metric would tell you the step worked.

**5.2** A traffic spike is coming (a marketing campaign at a known time). You have one week. What do you do?
> Load test first to find the actual breaking point — don't guess. Then: pre-scale (instances, DB size, connection pools), warm caches, verify autoscaling reacts fast enough (it usually doesn't for a sudden spike), add rate limiting, identify what you'd shed and build the switch *now* (feature flags for non-essential features), check that the queue can buffer the write path, and write a runbook with who does what. And confirm your monitoring will actually tell you what's failing.

**5.3** Your cache hit rate drops from 95% to 40% overnight and the database is melting. Give three causes.
> A deploy changed the cache key format (very common — a version prefix in the key would have made this visible); the cache was flushed or a Redis instance restarted, so everything is cold; TTLs were shortened; a new feature reads with high-cardinality keys that never repeat; or eviction pressure — the working set grew past available memory, so entries are evicted before they're re-read.

**5.4** p50 latency is 40ms. p99 is 6 seconds. What kinds of causes produce that shape, and how do you find it?
> A small subset of requests hits something pathological. Candidates: a specific endpoint or a specific user's data volume (an unindexed query that's fine for 10 rows and awful for 100k); cache misses; lock waits; connection pool waits; GC pauses; a slow replica in the pool; or a downstream dependency's own tail latency. Find it with tracing (sample the slow requests specifically), and log the request duration alongside user/endpoint/row-count so you can segment.

---

## Checkpoint 6 — Production Engineering

**6.1** A user reports a bug from three hours ago. They have no screenshot. What do you need in place to investigate, and how do you proceed?
> Need: structured logs with a request ID, ideally surfaced to the user in the error response; user/tenant ID on log lines; retained logs; traces; and metrics to check whether it was systemic or isolated. Process: find their requests by user ID and time window, check for errors, compare against the metrics for that window (was there a spike?), then check deploy and incident timelines.

**6.2** Your liveness probe checks the database. The database has a 30-second blip. What happens?
> Every instance fails liveness → the orchestrator restarts them all simultaneously → they all reconnect at once → a thundering herd on the recovering database → the blip becomes a full outage, possibly a crash loop. **Liveness must check only the process itself; dependency checks belong in readiness.**

**6.3** You need to roll back a deploy, but it included a database migration that dropped a column. What now?
> You can't cleanly roll back — this is the failure that expand–contract exists to prevent. Options now: roll forward with a fix, or restore the column and backfill from a backup/WAL (with data loss for anything written since). The lesson: migrations must be backward-compatible and destructive changes happen at least one deploy *after* the code stops using the column.

**6.4** Your on-call is paged 15 times a night; 14 are non-actionable. What's the real problem and how do you fix it?
> Alerting on causes (CPU, memory, disk percentage) instead of symptoms users feel. Fix: define SLOs, alert on SLO burn rate, error rate, and p99 latency; make everything else a dashboard, not a page. Every remaining alert gets a runbook. Alert fatigue *causes* outages — the one real page gets ignored.

**6.5** You have a `password` field showing up in your production logs. Walk through the response.
> Treat it as an incident: stop the bleeding (redact at the log pipeline immediately, then fix the code), assess exposure (who had access to those logs, for how long, how many users), force password resets for affected accounts, purge the logs if you can, and document it. Then prevent recurrence: a structural redaction layer in the logger plus a test that asserts sensitive fields never serialize — not a code-review reminder, which will fail again.

---

## Checkpoint 7 — System Design

**7.1** Design a URL shortener. You have 45 minutes. *(Then compare against [07](./07-system-design-framework.md).)*
> Grade yourself on: did you estimate before designing? Did you name the hard dimension? Did you define the API and data model before drawing boxes? Did you name the bottleneck explicitly? Did you state at least three trade-offs?

**7.2** An interviewer says "now assume one user has 50 million followers." What does that break, and what do you do?
> Fan-out on write collapses — one post becomes 50M writes. Hybrid approach: push for normal users, pull for celebrities, merged at read time. Also mention: rate limiting the fan-out, prioritizing active followers, and that this asymmetry is *why* hybrid exists.

**7.3** You've designed a system with a queue between two services. The interviewer asks: "what if the queue is down?"
> Answer honestly and completely: producers buffer locally with a bounded buffer, then shed load or fail the request with 503 + `Retry-After`; or fall back to synchronous processing for critical paths if that's safe. Also: the queue itself should be replicated/managed; monitor queue depth and publish failures; and — the key point — decide *in advance* whether accepting the request without being able to enqueue it is acceptable. If losing the message is unacceptable, the outbox pattern in your own database is the answer: commit the intent locally, publish later.

**7.4** Design a system, then answer: "what breaks first at 10× scale, and what breaks first at 100×?"
> 🥇 answers name a *different* component at each tier and say which metric would warn you before it broke.

---

## Final self-assessment

You are ready to interview as a strong backend engineer when you can, without notes:

- [ ] Narrate the full lifecycle of an HTTPS request for five minutes
- [ ] Read an unfamiliar `EXPLAIN ANALYZE` and name the problem
- [ ] Give an ordered investigation for "the API got slow"
- [ ] Explain idempotency and implement it correctly under concurrency
- [ ] Explain why a 504 differs from a 502 and what each implicates
- [ ] Design a system end-to-end in 45 minutes with estimation and trade-offs
- [ ] Name a design pattern from a problem description *and* argue against using it
- [ ] Explain what your own project would need to survive 100× traffic
- [ ] Describe how you'd debug a production issue you can't reproduce
- [ ] State a trade-off — what you bought and what you gave up — for every design decision you make
