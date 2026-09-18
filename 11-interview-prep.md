# 11 — Interview Preparation

> These are the questions that actually get asked. For each, a **model answer sketch** — not a script to memorize, but the shape of a strong response. A good answer states the mechanism, then the trade-off.

**The universal structure for any technical answer:**
1. What problem does it solve? (why it exists)
2. How does it work? (mechanism)
3. What does it cost? (trade-off)
4. When would you *not* use it?

Step 3 and 4 are what separate senior answers from junior ones. Most candidates stop at step 2.

---

## Databases

**What is an index, and how does it speed up queries?**
> A separate sorted data structure — usually a B-tree — mapping column values to row locations. Because it's sorted and shallow (3–4 levels even for hundreds of millions of rows), the database finds a value in ~4 page reads instead of scanning every row. It serves equality, ranges, prefix matches, and `ORDER BY`. The cost: every insert/update must maintain every index, plus storage. So indexes are a read/write trade, not free speed.

**Why didn't my index get used?**
> Common causes: a function on the column (`lower(email)` needs an expression index), a leading wildcard `LIKE '%x'`, a type mismatch, the wrong column order in a composite index (leftmost-prefix rule), low selectivity so a scan is genuinely cheaper, a table small enough that scanning wins, or stale statistics. I'd confirm with `EXPLAIN ANALYZE` rather than guess.

**What is a transaction? What does ACID guarantee?**
> A group of statements that succeed or fail as a unit. Atomicity: all or nothing. Consistency: constraints hold at commit. Isolation: concurrent transactions don't corrupt each other — how much is set by the isolation level. Durability: once committed, it survives a crash, via the write-ahead log.

**READ COMMITTED vs REPEATABLE READ?**
> READ COMMITTED (Postgres default) gives each *statement* a fresh snapshot, so you never see uncommitted data but can get different results reading the same row twice in one transaction. REPEATABLE READ (MySQL default) gives the whole *transaction* one snapshot, preventing non-repeatable reads. In Postgres, REPEATABLE READ also prevents phantoms and may abort with a serialization error you must retry.

**What causes a deadlock, and how do you prevent one?**
> A cycle in lock waits: A holds row 1 and wants row 2; B holds row 2 and wants row 1. The database detects the cycle and kills one. Prevention: acquire locks in a consistent order (e.g. always by ascending ID), keep transactions short, never do network I/O inside a transaction, and wrap transactions in a bounded retry — at scale, deadlocks are expected, not exceptional.

**How would you optimize a slow query?** *(walk through the process, don't list tricks)*
> First confirm it's the query and not pool exhaustion or an N+1 — check app-side vs DB time and query count per request. Then `EXPLAIN (ANALYZE, BUFFERS)` against production-like data volume. I look for a seq scan on a big table, a large gap between estimated and actual rows, and `Rows Removed by Filter`. Then classify: scanning too much (missing or unusable index), doing too much (bad join order, sort spilling to disk), or called too often (N+1). Fix cheapest-first — index, then query rewrite, then reduce rows returned, then denormalize, then cache. Re-measure and confirm the plan actually changed, and check what the new index costs on writes.

**What's an N+1 query?**
> One query fetches N rows, then a query fires per row inside a loop — usually ORM lazy loading. 100ms becomes 3 seconds. Fix with a join, eager loading, or a batched `IN` query plus an in-memory map. I detect it by logging query count per request and alerting above a threshold.

**Why is `OFFSET 1000000` slow, and what's the alternative?**
> The database must generate and discard a million rows to get to yours — it's O(offset). Keyset pagination uses a `WHERE (created_at, id) < (last_seen)` predicate on an indexed column, so it's constant time at any depth and stable when rows are inserted mid-scroll. The trade-off is losing random page jumps, which is almost always acceptable.

**How do you scale a database?**
> In this order: fix the queries and indexes first (routinely 10–100×), then scale vertically (cheap and boring, and correct for longer than people think), then cache hot reads, then add read replicas for read load, then partition huge tables for lifecycle and index size, and only then shard. Each step costs complexity, so I'd want a measurement justifying the next one.

**When would you shard, and what do you lose?**
> When writes or dataset size exceed one machine and replicas/caching are exhausted. You lose cross-shard joins and transactions, global uniqueness gets hard, aggregates become scatter-gather, and resharding is painful — mitigated by consistent hashing or many virtual shards. The shard key is the whole decision: it must distribute evenly and co-locate data queried together.

**Normalization vs denormalization?**
> Normalize by default so each fact is stored once — duplicated data drifts. Denormalize deliberately when a hot read path is provably join-bound, or when you need a historical snapshot (order line items must record the price at purchase time, not today's). Treat every denormalized field as a cache with a named owner and an update path.

**SQL vs NoSQL — how do you choose?**
> I start with Postgres unless I have a specific reason not to — it gives me transactions, constraints, joins, JSONB for flexible fields, and decent full-text search. I'd add Redis for hot reads, counters, sessions, and rate limiting. Elasticsearch when I need real relevance ranking. A document store when data is genuinely document-shaped and always read as one self-contained unit. Every extra datastore is a permanent operational tax.

---

## Networking

**What happens when you type a URL and press enter?**
> *(See the full 11-step narration in [05](./05-networking.md).)* Compressed: URL parse and HSTS check → DNS resolution through browser/OS/recursive resolver caches → TCP three-way handshake → TLS handshake with certificate validation and session key derivation → HTTP request → possibly served by the CDN edge → load balancer terminates TLS and picks a healthy backend → app middleware, routing, handler → cache lookup, then database via a pooled connection → response back through the LB, compressed, with cache headers → browser parses and repeats for subresources. And at every step there's a distinct failure mode.

**TCP vs UDP?**
> TCP is connection-oriented, ordered, reliable, with flow and congestion control — at the cost of a handshake round trip and head-of-line blocking. UDP is fire-and-forget datagrams with no guarantees and no setup. Use UDP when late data is worthless (voice, video, gaming), for DNS, or when you want to implement smarter reliability yourself — which is exactly what QUIC/HTTP3 does to escape TCP's head-of-line blocking.

**What is DNS, and what's a TTL?**
> The system mapping names to IPs, resolved through a hierarchy of caches: browser → OS → recursive resolver → root → TLD → authoritative. TTL controls how long each layer caches an answer. It bites you during migrations — a long TTL means your change takes hours to propagate — so you lower the TTL well in advance of a planned change.

**What does TLS give you and why is the handshake expensive?**
> Confidentiality, integrity, and server identity via a certificate chained to a trusted CA. The handshake costs 1–2 extra round trips for negotiation and key exchange before any data flows, plus asymmetric crypto. Mitigated by session resumption, keep-alive, and terminating TLS at an edge close to the user.

**What's a reverse proxy, and why use one?**
> A server in front of your application servers. It terminates TLS, load balances across healthy instances, routes by path or host, caches, compresses, rate limits, and hides your internal topology. It centralizes cross-cutting concerns instead of duplicating them in every service.

**What causes a 504? How is it different from 502 and 503?**
> 504 means the proxy reached a backend but the backend didn't respond within the proxy's timeout — so it's a *slowness* problem: a slow query, exhausted connection pool, a slow downstream with no timeout, lock contention, or a GC pause. 502 means the backend answered with something invalid or dropped the connection — the app crashed or is broken. 503 means there was no healthy backend at all, or you're deliberately shedding load. Bad answer, no answer, late answer.

**Connection refused vs connection timeout?**
> Refused means something actively replied "no" — nothing is listening on that port, or a firewall rejected it. Timeout means packets vanished with no response at all — usually a firewall or security group *dropping* traffic, a wrong IP, or a down host. Refused tells you routing worked; timeout tells you it may not have.

**How do you prevent one slow dependency from taking down your service?**
> Timeouts on every outbound call — a missing timeout is an unbounded resource leak and the number one cause of cascading failure. Then bounded retries with exponential backoff and jitter on idempotent operations only, a circuit breaker so I fail fast instead of queueing work for something already down, bulkheads so each dependency has its own pool, and a fallback: cached value, default, or a degraded response.

---

## System Design

**How do you scale a web application from 1,000 to 1,000,000 users?**
> Make the app tier stateless and put it behind a load balancer so I can add instances. Move sessions to Redis and files to object storage. Then measure and find the bottleneck — it's almost always the database. Add caching for hot reads, then read replicas. Move everything the user doesn't need synchronously into a queue with workers. Put static assets on a CDN. Only then consider partitioning or sharding. At each step I want a metric telling me what broke, rather than pre-building for scale I don't have.

**When would you use Redis? When is it the wrong tool?**
> Right for: caching, sessions, rate limiting (`INCR` + TTL, atomic because it's single-threaded), distributed locks, leaderboards via sorted sets, ephemeral counters, and pub/sub. Wrong when it's your only copy of data that matters — it's memory-bound and persistence is configurable, not guaranteed. I treat Redis as derived state I can always rebuild, and I make sure the system still works, slower, when Redis is down.

**Why use a message queue instead of calling the service directly?**
> Three reasons: latency (the user gets a response immediately while work happens after), resilience (a failing downstream doesn't fail the user's request — the message waits and retries), and buffering (a traffic spike becomes queue depth instead of dropped requests), plus independent scaling of producers and consumers. The costs: eventual consistency, at-least-once delivery so consumers must be idempotent, ordering complexity, and new things to monitor — queue depth, consumer lag, dead-letter queue.

**What is eventual consistency, and when is it unacceptable?**
> It means replicas converge once writes stop; a read may return stale data in the meantime. Fine for feeds, view counts, search indexes, recommendations, analytics. Unacceptable for anything where being wrong causes a real-world loss: account balances, inventory decrements on the last unit, authorization decisions. The key point is it's a per-operation choice, not a system-wide one — the same e-commerce app is strongly consistent at checkout and eventually consistent on the product page.

**What is idempotency, and how do you implement it?**
> An operation you can apply repeatedly with the same result as applying it once. It matters because networks lose *responses*, so a client can never be sure whether its request succeeded, and it will retry. Implementation: the client sends a unique idempotency key; the server stores key → response with a UNIQUE constraint, records the result in the same transaction as the work, and returns the stored response on any duplicate. The UNIQUE constraint also resolves the concurrent-duplicate race, which application-level "check then insert" cannot.

**Explain the CAP theorem.**
> During a network partition you must choose between consistency and availability. I'd add two things: it's about behavior *during a partition*, not a database's permanent personality; and the rest of the time the real trade is consistency versus latency. In practice the choice is per operation — for a payment I'd refuse the request rather than risk a double charge; for a product page I'd serve slightly stale data rather than an error.

**How would you handle millions of requests per second?**
> First I'd estimate to find which dimension is actually hard — reads, writes, or bandwidth — because the answers differ completely. Then: push as much as possible to the CDN and edge caches so most requests never reach me; a stateless horizontally scaled app tier; aggressive caching on hot reads; read replicas; async processing for all non-critical work; partitioning or sharding on the write path with a shard key that spreads evenly; and rate limiting plus load shedding so overload degrades rather than collapses. At that scale I'd also assume constant partial failure, so timeouts, circuit breakers, and idempotency are structural, not optional.

**How do you handle a distributed transaction?**
> I'd first try to avoid one by keeping a single business operation inside one database. When I can't, a saga: a sequence of local transactions each with a compensating action, so a failed inventory reservation triggers a refund. It isn't atomic — there's a visible inconsistency window, and compensations must be idempotent. And I'd use the outbox pattern so the event is published if and only if the local transaction committed, which removes the classic "we committed but lost the event" failure.

**Fan-out on write vs fan-out on read?**
> On write, I push a new post into every follower's precomputed feed — reads are instant, but a celebrity with 10M followers costs 10M writes per post. On read, I merge posts from everyone you follow at request time — writes are cheap, reads are slow. The real answer is hybrid: push for normal users, pull for celebrities, and merge at read time.

---

## Design Patterns

**When would you use Strategy, and how is it different from an if/else?**
> When there are several interchangeable implementations of the same operation and the list keeps growing — payment providers, shipping calculators, export formats. It differs from an if/else in that adding a case means adding a class rather than editing existing code, so nothing existing can regress and each implementation is independently testable. With two stable branches, I'd just write the if — the pattern earns its indirection around the third variation.

**Factory vs Builder?**
> Factory answers "which implementation do I get?" and centralizes conditional construction. Builder answers "how do I assemble this complex object step by step?" and exists for many optional parameters with validation at the end. In languages with named and default arguments, Builder is mostly unnecessary — except for test data builders, which are genuinely valuable.

**Adapter vs Decorator?**
> Both wrap an object. Adapter *changes the interface* — making a vendor SDK fit the interface my domain defines — with the same behavior. Decorator *keeps the interface* and adds behavior — caching, logging, retry. The quick test: if the wrapper implements the same interface it wraps, it's a decorator.

**Why use Dependency Injection?**
> So a class declares what it needs instead of constructing it. Concretely: I can unit test the service with a fake repository and a fake clock instead of a real database and real time; I can swap implementations per environment; and the constructor becomes honest documentation of the dependencies. It doesn't require a framework — passing constructor arguments *is* dependency injection.

**When is a design pattern the wrong choice?**
> When it adds indirection without removing duplication or enabling a change I can actually name. A Strategy interface with one implementation, a Repository that forwards one-to-one to the ORM, or a five-deep decorator stack that makes stack traces unreadable — those cost every future reader a hop and buy nothing. I apply the pattern on the second or third variation, not the first.

---

## Behavioral questions with a technical core

These are asked constantly and candidates under-prepare them. Have a concrete story ready for each — your projects from [10](./10-projects.md) are legitimate material.

- **Tell me about a performance problem you debugged.** → Use your Project 4 report. Symptom → hypothesis → measurement → fix → verification, with numbers.
- **Tell me about a technical decision you'd make differently.** → Shows judgment and honesty. Your "what I'd do differently" docs are for this.
- **How do you decide when something is over-engineered?** → Talk about naming the concrete change the abstraction enables.
- **Describe a time you disagreed on an architectural decision.** → Focus on how you made the trade-off explicit rather than on who won.
- **How do you approach a bug you can't reproduce?** → Logs, metrics, traces, narrowing by layer, adding instrumentation rather than guessing.

---

## How to practice

1. **Say answers out loud.** Written fluency ≠ spoken fluency, and interviews are spoken.
2. **Record yourself once per area.** Painful, extremely effective.
3. **For every answer, force yourself to add the trade-off sentence.** That habit alone raises your perceived level.
4. **When you don't know something, say so and reason from principles.** "I haven't used that, but based on how X works I'd expect..." is a *good* answer. Bluffing is the only fatal one.
5. **Do 10 system designs out loud with a timer** before your first real system design interview. There is no substitute.
