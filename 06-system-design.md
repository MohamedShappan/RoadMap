# 06 — System Design: Beginner → Intermediate → Advanced

> System design is not a separate subject. It is the composition of everything in stages 1–6. Every box you draw should be one you could implement.

---

# LEVEL 1 — The Building Blocks 🔥

Know, for each: **what it does, what it costs, and how it fails.** That last one is what separates a real answer from a memorized diagram.

### Client
Browser, mobile app, another service. **Assume it is hostile and unreliable**: it retries, it's on a bad network, it runs an old version you can't force-update (hence API versioning), and it lies about every input (hence server-side validation).

### API Gateway / Edge
The single front door: TLS termination, routing, authentication, rate limiting, request-ID injection, and sometimes response aggregation.
**Cost:** one more hop, and a potential single point of failure (run it redundantly).
**Why it exists:** cross-cutting concerns belong in one place, not duplicated in every service.

### Load Balancer
Distributes across healthy instances. L4 (TCP) vs L7 (HTTP-aware). Health checks eject bad instances.
**Fails when:** health checks are meaningless (200 while the DB is down), or all backends fail at once (→ 503), or sticky sessions concentrate load.

### Application Server (the stateless tier)
Your business logic. **Must be stateless** — any instance can serve any request — because that's what makes horizontal scaling and safe restarts possible. Session state goes to Redis; files go to object storage; in-progress work goes to a queue.
**Fails when:** one dependency is slow and there's no timeout (thread pool exhaustion).

### Database
The system of record. Strong consistency, transactions, integrity. **Almost always the first real bottleneck**, and usually the hardest thing to scale.
**Fails when:** missing index, lock contention, connection pool exhausted, replica lag, disk full.

### Cache (Redis/Memcached)
Fast key-value store for hot reads and derived values.
**Cost:** staleness, an extra failure mode, and memory. **Fails when:** it's down (can you survive it?), or a hot key expires and stampedes the DB, or you made it your source of truth.
**Rule: a cache must always be optional.** If the system can't run with a cold cache, you've built a distributed single point of failure.

### Message Queue (SQS / RabbitMQ / Kafka)
Decouples producers from consumers; buffers bursts; enables retries and async work.
**Gives you:** the ability to accept work now and do it later, isolation between services, and smoothing of spikes.
**Costs:** eventual consistency, at-least-once delivery (**so consumers must be idempotent**), ordering complexity, and a new thing to monitor (queue depth, consumer lag, dead-letter queue).
**Queue vs log:** a classic queue (SQS/Rabbit) deletes a message after it's consumed; a log (Kafka) retains it and lets multiple independent consumer groups replay from an offset. Choose the log when several systems need the same event stream or you want replay.

### Object Storage (S3/GCS)
Cheap, durable, effectively infinite blob storage.
**Rule: never store files in your database, and never store files on an app server's local disk** (it breaks statelessness). Serve and receive them via **presigned URLs** so bytes bypass your app entirely.

### CDN
Edge cache for static assets and cacheable responses. Cuts latency and origin load.

### Search Index (Elasticsearch)
Inverted index for full-text relevance and faceting. **Always a secondary index**, fed asynchronously, never the system of record.

### Blob of "workers"
The other half of the queue: a horizontally scalable pool that consumes jobs. Scale workers independently from web servers — that independence is a major reason to introduce a queue at all.

---

# LEVEL 2 — The Scaling Toolkit 🔥

### Vertical vs horizontal scaling
- **Vertical** — bigger machine. Zero code change, instant, and correct for far longer than people admit. Ceiling: the biggest machine, and it's a SPOF.
- **Horizontal** — more machines. Effectively unlimited and gives redundancy, but requires statelessness, introduces coordination, and makes consistency your problem.
**Say this in interviews:** *"I'd scale vertically first because it's free engineering time, and design so horizontal scaling is possible when I need it."*

### Stateless services 🔥
The enabling constraint for everything else. Push state outward: sessions → Redis, uploads → S3, jobs → queue, config → env. Then any instance can die at any moment and you just... don't care.

### Caching layers 🔥
Cache at every level, with decreasing generality:
`browser → CDN → API gateway → application memory → Redis → database buffer pool`

Decisions to make explicitly:
- **What to cache** — expensive to compute, frequently read, tolerably stale.
- **TTL** — driven by "how wrong may this be?" Not by vibes.
- **Invalidation** — write-through delete, TTL, or event-driven.
- **Stampede protection** — locking, stale-while-revalidate, jittered TTLs.
- **Hot key handling** — replicate the key, or cache it locally in the app for a few seconds.

### Database replication & read/write separation 🔥
Primary takes writes; replicas serve reads. Solves read scale, not write scale. Introduces replication lag → handle read-your-own-writes. (Details in [03](./03-databases.md).)

### Message queues & async processing 🔥
**The single most useful scaling move in system design.** Ask of every operation in a request: *does the user need this to have finished before they get a response?*

Convert synchronous to asynchronous:
```
Before: POST /orders → charge card → send email → update analytics → index for search → 200 (2.5s, fails if any step fails)
After:  POST /orders → persist order + outbox row → 202 Accepted (80ms)
        workers: charge, email, analytics, search-index — each retried independently
```
**Gains:** latency, resilience (one failing dependency doesn't fail the request), burst absorption, independent scaling.
**Costs:** eventual consistency (the user sees "processing"), duplicate deliveries (→ idempotency), ordering issues, and operational surface (DLQ, lag monitoring).

### Rate limiting 🔥
Protects you from abuse, from bugs, and from one tenant starving others.
- **Token bucket** — allows bursts, refills at a steady rate. The usual right answer.
- **Sliding window counter** — smooth, cheap, approximate.
- **Fixed window** — simplest; suffers from double-burst at boundaries.
Implement centrally in Redis (`INCR` + TTL, or a Lua script for atomicity), key by user/API key/IP/tenant. Return **429 with `Retry-After`**. Distinguish *throttling* (slow down) from *quota* (monthly limit).

### Idempotency 🔥🔥
**The answer to "the network lied to me."** If a client can't tell whether its request succeeded, it will retry — so make retries harmless.

Implementation (know this cold):
1. Client generates a unique `Idempotency-Key` per logical operation and sends it on retries.
2. Server stores `(key, request_hash, response, status)` with a **UNIQUE constraint on the key**.
3. First request: do the work and record the result in the *same transaction*.
4. Duplicate request: return the stored response without re-doing the work.
5. Concurrent duplicate: the UNIQUE constraint makes one lose; it waits and returns the stored result.

Also: prefer naturally idempotent designs — `PUT` with a client-supplied ID, `SET status='paid'` instead of `increment`, and conditional updates (`WHERE status='pending'`).

### Retries, timeouts, circuit breakers, backpressure 🟡
- **Timeout** everything. A missing timeout is an unbounded resource leak.
- **Retry** with exponential backoff + jitter, bounded, **idempotent operations only**, with a retry budget.
- **Circuit breaker** — closed → (failures exceed threshold) → open (fail fast immediately) → after a cooldown → half-open (let one probe through) → closed. Prevents you from queueing work for a dependency that is already down.
- **Bulkhead** — separate connection/thread pools per dependency so one slow dependency can't consume all your capacity.
- **Backpressure** — when you're overloaded, *say so* rather than accumulating unbounded queues. Bounded queues, load shedding (drop low-priority work), 429/503 with `Retry-After`. **An unbounded queue is not resilience — it's a delayed outage with worse latency.**
- **Graceful degradation** — serve stale cache, hide the recommendations widget, disable non-essential features. Deciding *in advance* what you'd shed is a senior move.

---

# LEVEL 3 — Distributed Systems, Practically 🟡

### CAP theorem, honestly
When a network **partition** (P) happens — and it will — you must choose between **consistency** (every read sees the latest write) and **availability** (every request gets a non-error response).

**What actually matters in interviews and in practice:**
- CAP is about behavior *during a partition*, not a permanent personality trait of a database. The rest of the time you're trading consistency against **latency** (that's PACELC, and it's the more useful lens day to day).
- The choice is **per operation**, not per system. In a payments app: balance updates are CP (refuse rather than double-spend); the transaction history page can be AP (slightly stale is fine).
- Saying "MongoDB is AP and Postgres is CA" is a red flag. Say instead: *"For this operation, during a partition, I'd rather return an error than a wrong balance — so I'd take consistency and accept reduced availability on that path."*

### Consistency models worth naming
- **Strong / linearizable** — reads see the latest write. Expensive; needs coordination.
- **Read-your-own-writes** — you see your changes; others may lag. The pragmatic default for user-facing apps.
- **Monotonic reads** — you never see time go backwards (important when load balancing across replicas of differing lag).
- **Eventual** — converges if writes stop. Fine for counts, feeds, search, recommendations.

### Distributed transactions 🟡
**First advice: avoid them.** Design so that a single business operation touches a single database when you can. Then:
- **2PC** — a coordinator makes everyone prepare, then commit. Correct, but the coordinator is a SPOF and locks are held across the network. Rarely used in modern systems.
- **Saga** — a sequence of local transactions, each with a **compensating action**. Order created → payment charged → inventory reserved; if inventory fails, issue a refund. Choreographed (events) or orchestrated (a coordinator service). Not atomic — there's a window where the world is inconsistent, and compensation must itself be idempotent.
- **Outbox pattern** 🔥 — the practical workhorse. Write the business change *and* the event row in one local transaction; a relay process publishes the event and marks it sent. Guarantees "the event is published if and only if the change committed," at-least-once. Learn this one properly; it's the correct answer to a very common interview follow-up.

### Event-driven architecture 🟡
Services emit facts (`OrderPlaced`), others react. Gives loose coupling, independent scaling, and replay.
**Costs:** debugging spans many services (→ distributed tracing), eventual consistency everywhere, schema evolution of events becomes an API contract, and duplicates require idempotent consumers.

### Consistent hashing 🟡
Placing keys across N nodes so that adding/removing a node moves ~1/N of keys instead of remapping everything. Virtual nodes even out the distribution. Used in caches, shard routing, and DHTs. Know the concept and *why* naive `hash(key) % N` is bad (it remaps nearly everything when N changes).

### Leader election, quorums 🟡
- **Quorum:** with N replicas, if W + R > N, reads see the latest write. (N=3, W=2, R=2 is the classic.)
- **Leader election** via Raft/ZooKeeper/etcd. Know *that* consensus algorithms provide this and when you need one (single writer, distributed locks, cluster membership). **Do not study Raft internals now** — see [12](./12-do-not-learn-yet.md).

### Observability in distributed systems 🔥
With one service, logs suffice. With ten, you need:
- **Correlation/trace IDs** propagated across every hop (`traceparent`).
- **Distributed tracing** to see where a 2-second request spent its time.
- **RED metrics per service** (Rate, Errors, Duration) and **USE for resources** (Utilization, Saturation, Errors).
- **Service-level objectives** — "99.9% of requests under 300ms" — which turn vague "is it healthy?" into a number, and give you an error budget to spend on shipping.

---

## Estimation: back-of-the-envelope 🔥

Memorize these; they're all you need.

**Numbers every engineer should know**
| Operation | Time |
|---|---|
| L1 cache / memory reference | ~1–100 ns |
| SSD random read | ~100 µs |
| Same-datacenter round trip | ~0.5 ms |
| Simple indexed DB query | ~1–10 ms |
| Disk seek (HDD) | ~10 ms |
| Cross-continent round trip | ~150 ms |

**Rules of thumb**
- 1 million seconds ≈ 12 days; **86,400 seconds/day ≈ 100k** (use 100k for mental math).
- Peak QPS ≈ 2–3× average QPS.
- Reads outnumber writes ~100:1 in typical consumer apps.
- 1 KB text, ~200 KB image, ~2 MB photo, ~1 GB/hour of 1080p video.
- A single well-indexed Postgres box: thousands of TPS. A Redis instance: ~100k ops/s. One app server: ~1–5k rps for simple work.

**Worked example — a Twitter-like feed, 100M DAU**
```
Reads:  100M users × 20 feed loads/day = 2B reads/day ÷ 100k s = 20k QPS avg → ~50k peak
Writes: 100M × 0.1 tweets/day         = 10M writes/day ÷ 100k = 100 QPS avg → ~300 peak
Storage: 10M tweets × 300 bytes ≈ 3 GB/day ≈ 1.1 TB/year (text is cheap; media is not)
Conclusion: read-heavy by 200:1 → fan-out on write, aggressive caching, read replicas.
           Writes are trivial — one database handles them. Don't shard for writes.
```
**The point of estimation is not the number.** It's identifying which dimension is hard, so you spend your design effort there. Always end an estimate with "so the hard part is X."

---

## Anti-patterns that flag inexperience

- **Microservices as the opening move.** "I'd start with a modular monolith; I'd split out the service that needs independent scaling when I have evidence" is a *stronger* answer.
- **Sharding before caching and replicas.**
- **A cache your system can't run without.**
- **No timeouts** anywhere in the diagram.
- **Unbounded queues** presented as resilience.
- **Ignoring the write path.** Everyone designs reads; the interesting failures are in writes.
- **Naming technologies instead of properties.** Say "I need low-latency key-value with TTL" *then* say Redis.
- **No estimation**, so no idea which part is actually hard.
- **Forgetting the human layer** — deploys, migrations, on-call, and "how would we know this broke?"
