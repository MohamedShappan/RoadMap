# 03 — Databases: The 80/20 Deep Dive

> The highest-ROI domain in backend engineering. Most performance incidents, most data corruption bugs, and most system design answers bottom out here.

---

## Part 1 — Relational Fundamentals 🔥

### Why relational databases exist

Before them, applications owned their own files and every program re-implemented searching, locking, and crash recovery — badly. The relational database is a shared, concurrent, crash-safe, queryable store with **integrity guarantees enforced centrally**. The key insight: *your application code is temporary; your data is permanent.* Rules enforced in the database survive rewrites, new services, and bad migrations. Rules enforced only in application code do not.

### Tables, keys, relationships

- **Primary key** — the unique identity of a row. Prefer a surrogate key (`BIGSERIAL` or UUID) over a natural key, because natural keys change (people change emails, countries change codes).
  - `BIGSERIAL`: small, sequential, index-friendly. Downside: guessable and reveals volume.
  - `UUIDv4`: unguessable, generatable client-side, but random → poor index locality and page splits. **`UUIDv7`** (time-ordered) gets you most of both.
- **Foreign key** — a column that must reference an existing row elsewhere. It is not decoration; it prevents orphan rows that *will* otherwise appear.
- **Relationships:**
  - **1:N** — FK on the "many" side. `orders.user_id → users.id`.
  - **N:M** — a junction table: `student_courses(student_id, course_id)`, PK on the pair.
  - **1:1** — FK + UNIQUE, usually to split rarely-used or sensitive columns off a hot table.

### Constraints — the cheapest reliability you will ever buy

```sql
CREATE TABLE orders (
  id           BIGSERIAL PRIMARY KEY,
  user_id      BIGINT      NOT NULL REFERENCES users(id),
  status       TEXT        NOT NULL CHECK (status IN ('pending','paid','shipped','cancelled')),
  total_cents  BIGINT      NOT NULL CHECK (total_cents >= 0),
  idempotency_key TEXT     NOT NULL,
  created_at   TIMESTAMPTZ NOT NULL DEFAULT now(),
  UNIQUE (user_id, idempotency_key)
);
```

**Mental model:** every constraint you skip is a future data-cleanup script written at 2 a.m. `NOT NULL` is free. `CHECK` is free. A `UNIQUE` constraint is also the correct way to make an operation idempotent under concurrency — application-level "check then insert" is a race condition, `UNIQUE` is not.

Store money as integer minor units (`total_cents BIGINT`) or `NUMERIC`, never `FLOAT`. Store timestamps as `TIMESTAMPTZ` in UTC.

### SQL — the 20% of syntax that does 80% of work

```sql
-- JOIN + GROUP BY + aggregate + HAVING: the workhorse shape
SELECT u.id, u.email, COUNT(o.id) AS order_count, COALESCE(SUM(o.total_cents),0) AS spend
FROM users u
LEFT JOIN orders o ON o.user_id = u.id AND o.status = 'paid'
WHERE u.created_at >= now() - interval '90 days'
GROUP BY u.id, u.email
HAVING COUNT(o.id) > 3
ORDER BY spend DESC
LIMIT 20;
```

Things worth truly internalizing:

- **INNER vs LEFT JOIN.** INNER drops rows with no match; LEFT keeps them with NULLs. A classic bug: putting a filter on the *right* table of a LEFT JOIN in the `WHERE` clause silently converts it into an INNER JOIN. Put it in the `ON` clause instead (as above).
- **WHERE vs HAVING.** `WHERE` filters rows before grouping; `HAVING` filters groups after.
- **NULL is not a value, it's "unknown."** `NULL = NULL` is not true. Use `IS NULL`. `COUNT(col)` skips NULLs; `COUNT(*)` doesn't.
- **Subqueries:** prefer `EXISTS` over `IN (SELECT ...)` for correlated existence checks — it short-circuits and handles NULLs sanely.
- **Aggregates:** `COUNT`, `SUM`, `AVG`, `MIN`, `MAX`, plus `COUNT(DISTINCT x)`.
- 🟡 **Window functions** — `ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY created_at DESC)` is how you get "most recent row per group" without a self-join. Learn this in Stage 5; it's the highest-value 🟡 SQL feature.
- 🟡 **CTEs (`WITH`)** — readability for multi-step queries.

### Normalization → then denormalization

**Normalize (to 3NF) by default:**
1. **1NF** — no repeating groups/arrays-as-columns; one value per cell.
2. **2NF** — no partial dependency on part of a composite key.
3. **3NF** — no column depends on another non-key column.

Practically: *store each fact exactly once.* If a customer's address is in three tables, two of them are already wrong.

**Denormalize deliberately when** a read path is hot and joins are proven expensive. Classic legitimate cases:
- `orders.total_cents` copied from the sum of items (also a *historical* correctness requirement — prices change).
- `posts.comment_count` as a counter cache.
- A snapshot of product name/price on `order_items` — you must show what the customer actually bought at that time, not today's data.

**The rule:** denormalization is a cache, and caches go stale. Every denormalized field needs a defined owner and an update path. Never denormalize before you have a measurement proving the join is the problem.

---

## Part 2 — Transactions & Concurrency 🔥

### Why they exist
Because a "single business operation" is usually multiple statements, and the machine can die in between. A transaction turns *n* statements into one all-or-nothing unit.

### ACID, stated usefully

| Property | Real meaning |
|---|---|
| **Atomicity** | All statements commit, or none do. No half-transfers. |
| **Consistency** | The DB moves from one valid state to another — your constraints hold at commit. |
| **Isolation** | Concurrent transactions don't corrupt each other. *How much* they don't is the isolation level. |
| **Durability** | Once committed, it survives a crash (write-ahead log + fsync). |

### Isolation levels — the part everyone gets wrong

The anomalies:
- **Dirty read** — you see another transaction's uncommitted data.
- **Non-repeatable read** — you read a row twice in one transaction and get different values.
- **Phantom read** — you re-run a range query and new rows have appeared.
- **Lost update** — two transactions read-modify-write the same row; one write vanishes.

| Level | Prevents | Notes |
|---|---|---|
| READ UNCOMMITTED | nothing | Never use. Postgres doesn't even implement it distinctly. |
| **READ COMMITTED** | dirty reads | **Postgres default.** Each *statement* sees a fresh snapshot. Non-repeatable reads possible. |
| **REPEATABLE READ** | + non-repeatable reads | **MySQL/InnoDB default.** Whole *transaction* sees one snapshot. In Postgres this also prevents phantoms and may abort with a serialization error you must retry. |
| SERIALIZABLE | everything | Behaves as if transactions ran one at a time. Correct but costly; expect retries. |

**Practical guidance:** stay at the default. When you need to prevent a lost update on a specific row, don't raise the global isolation level — use one of these instead:

```sql
-- Pessimistic: take the row lock now
SELECT balance FROM accounts WHERE id = 1 FOR UPDATE;

-- Optimistic: fail if someone changed it under you
UPDATE accounts SET balance = $1, version = version + 1
WHERE id = 1 AND version = $2;   -- 0 rows affected → retry

-- Best of all when possible: let the DB do the arithmetic atomically
UPDATE accounts SET balance = balance - 100 WHERE id = 1 AND balance >= 100;
```

The third form has no read-modify-write window at all. Prefer it whenever the operation can be expressed as a single statement.

### Locks & deadlocks 🔥

A **deadlock** is a cycle: transaction A holds row 1 and wants row 2; B holds row 2 and wants row 1. The database detects the cycle and kills one with an error.

**The three rules:**
1. **Always acquire locks in a consistent order.** The classic money-transfer deadlock disappears if every transaction locks account IDs in ascending order.
2. **Keep transactions short.** Never do an HTTP call, a file upload, or user input inside a transaction. This is the most common real cause of lock pileups.
3. **Expect and retry.** Deadlocks and serialization failures are normal at scale; wrap transactions in a bounded retry.

Long-running transactions also cause a subtler problem: in Postgres, they block vacuum, which causes table bloat, which slowly degrades everything.

### Connection pooling 🔥

Opening a TCP connection + TLS + auth + backend process startup costs ~10–100ms. Doing it per request is fatal. A pool keeps *n* connections open and hands them out.

- Postgres connections are **expensive** (a process each). Sane pool size is roughly `(cores × 2) + effective_spindles` per instance — typically 10–30, not 500.
- More connections ≠ more throughput. Past a point, you get context-switch thrash and lock contention.
- With many app instances, put **PgBouncer** in front (transaction pooling mode).
- **Pool exhaustion presents as "the API is slow," not as a database error** — requests queue waiting for a connection. Always instrument pool wait time. This is one of the most commonly misdiagnosed production problems.

---

## Part 3 — Indexes & "Why Is This Query Slow?" 🔥🔥

This section is the single most valuable thing in the roadmap. Master it.

### How a B-tree index actually works

A B-tree is a balanced, sorted, shallow tree (typically 3–4 levels even for hundreds of millions of rows). Each node holds many keys, so lookups cost ~3–4 page reads instead of scanning the table.

Consequences you must be able to reason from:
- **Lookups are O(log n)** — near-constant in practice.
- **The index is sorted**, so it serves `=`, `<`, `>`, `BETWEEN`, `ORDER BY`, and prefix `LIKE 'abc%'` — but **not** suffix `LIKE '%abc'`.
- **Every index slows writes** — an INSERT must update every index on the table. Indexes aren't free; they're a read/write trade.
- **Indexes consume memory and disk.** An unused index is pure cost.

### The query plan: `EXPLAIN ANALYZE`

```sql
EXPLAIN (ANALYZE, BUFFERS) SELECT * FROM orders WHERE user_id = 42 AND status = 'paid';
```

What to look for, in priority order:

1. **`Seq Scan` on a large table** — reading every row. Usually the smoking gun. (On a *small* table it's correct and faster than an index.)
2. **Estimated rows vs actual rows wildly different** — the planner is working from stale statistics. `ANALYZE tablename;`.
3. **The most expensive node** — read the plan inside-out; the deepest/most costly node is where your time goes.
4. **`Nested Loop` with a high loop count** — often an N+1 in disguise, or a bad join order.
5. **`Rows Removed by Filter: 900000`** — you fetched a million rows to keep a hundred. Index the filter column.
6. **Sort spilling to disk** (`external merge`) — needs an index that provides the order, or more `work_mem`.

### Why your index isn't being used — the checklist

| Cause | Example | Fix |
|---|---|---|
| Function on the column | `WHERE lower(email) = $1` | Expression index: `CREATE INDEX ON users (lower(email))` |
| Leading wildcard | `WHERE name LIKE '%foo'` | Trigram index, or full-text search, or Elasticsearch |
| Type mismatch | `WHERE id = '42'` with mixed types | Fix the types |
| Wrong column order in composite | index `(status, user_id)`, query on `user_id` only | Reorder or add an index |
| Low selectivity | `WHERE is_active = true` when 95% are true | Index won't help; consider a partial index |
| Table too small | 500 rows | Correct behavior — leave it |
| `OR` across columns | `WHERE a = 1 OR b = 2` | Rewrite as `UNION`, or add both indexes |
| Stale statistics | planner thinks the table has 100 rows | `ANALYZE` |

### Composite indexes and the leftmost-prefix rule 🔥

An index on `(a, b, c)` can serve queries filtering on:
- `a` ✅
- `a, b` ✅
- `a, b, c` ✅
- `b` ❌
- `b, c` ❌

Think of a phone book sorted by (last name, first name): you can find "all Smiths" and "Smith, John," but not "all Johns."

**Design rule:** equality columns first, then the range/sort column last.
For `WHERE tenant_id = ? AND status = ? ORDER BY created_at DESC`, the right index is `(tenant_id, status, created_at DESC)`.

**Covering index (index-only scan):** if the index contains every column the query needs, the database never touches the table.
`CREATE INDEX ON orders (user_id) INCLUDE (status, total_cents);`

### The N+1 query problem 🔥

The most common performance bug in production software, and almost always ORM-induced:

```
1 query:  SELECT * FROM orders LIMIT 100;
100 queries: SELECT * FROM users WHERE id = ?;   -- once per order, inside a loop
```

100ms of work becomes 3 seconds. **Fix:** fetch in a batch — a JOIN, an eager-load/`include`, or `WHERE id IN (...)` followed by an in-memory map. **Detect:** log query counts per request; alert when a single request issues more than ~20 queries. Add this instrumentation to every project you build.

### Pagination at scale 🔥

```sql
-- BAD: the database must scan and discard 1,000,000 rows
SELECT * FROM posts ORDER BY created_at DESC LIMIT 20 OFFSET 1000000;

-- GOOD: keyset / "seek" pagination — constant time at any depth
SELECT * FROM posts
WHERE (created_at, id) < ($last_created_at, $last_id)
ORDER BY created_at DESC, id DESC
LIMIT 20;
```

Keyset pagination requires an index on the sort key and gives you stable results when rows are inserted mid-scroll. The trade-off: no random "jump to page 5000." That's almost always acceptable — and it's exactly why infinite-scroll feeds work this way.

Also: `COUNT(*)` for "1 of 50,000 pages" is itself a full scan. Use an estimate, or drop the total.

### A repeatable "why is this query slow?" procedure

1. **Is it the query at all?** Check app-side time vs DB time. Could be pool wait, N+1, serialization, or network.
2. **Reproduce with real data volume.** Nothing is slow on 100 rows.
3. **`EXPLAIN (ANALYZE, BUFFERS)`.** Find the most expensive node.
4. **Classify:** scanning too much (missing/unusable index), doing too much (bad join, sort, aggregate), or called too often (N+1).
5. **Fix the cheapest way:** add/reorder an index → rewrite the query → reduce rows returned → denormalize → cache.
6. **Re-measure.** Confirm the plan actually changed.
7. **Check the write cost** of any index you added.

Ask these five diagnostic questions in order: *How many rows does it touch? How many does it return? Is there an index it could use? Is it being called N times? Is it waiting on a lock or a connection?*

---

## Part 4 — Scaling a Database 🔥/🟡

Apply these **in order**. Most teams never need step 5.

### 0. Fix your queries first 🔥
Indexing and N+1 elimination routinely deliver 10–100×. Scaling infrastructure to compensate for a missing index is expensive and embarrassing.

### 1. Vertical scaling 🔥
Bigger machine. Boring, instant, and correct far longer than engineers like to admit. A modern single Postgres box handles tens of thousands of TPS. Limit: you run out of machine, and it's a single point of failure.

### 2. Caching 🔥
Put Redis in front of hot reads. See Part 6.

### 3. Read replicas 🔥
Primary handles writes and streams its WAL to replicas that serve reads.

- **Solves:** read-heavy load (most workloads are 90%+ reads).
- **Doesn't solve:** write load. All writes still hit one primary.
- **The catch — replication lag.** A replica is milliseconds to seconds behind. User posts a comment, is redirected, reads from a replica, and their comment isn't there.
- **Fixes:** route reads-after-writes to the primary for a few seconds (sticky), or read the user's own data from the primary, or accept it where staleness is harmless (analytics, search listings).

### 4. Partitioning 🟡
Split one logical table into physical chunks *within the same database*, by range (usually time) or hash.

- **Solves:** huge tables, index bloat, and above all **cheap data lifecycle** — dropping last year's partition is instant; `DELETE FROM events WHERE created_at < ...` on 500M rows is an outage.
- **Doesn't solve:** total write throughput on one machine.

### 5. Sharding 🟡
Split data across *multiple databases* by a shard key (`user_id`, `tenant_id`).

- **Solves:** write throughput and dataset size beyond one machine.
- **Costs, all of them real:** no cross-shard joins; no cross-shard transactions; global uniqueness gets hard; aggregate queries must scatter-gather; **resharding is brutal** (mitigate with consistent hashing or many virtual shards mapped to few physical ones); hot shards (the celebrity problem).
- **Choosing a shard key** is the whole game: it must spread load evenly *and* co-locate data that's queried together. Sharding a social app by `user_id` makes "my timeline" easy and "everyone who liked this post" hard.
- **Rule:** shard last, and only when replicas + caching + partitioning are exhausted.

### Replication topologies
- **Single-primary (async)** — the default. Fast writes, possible data loss on failover.
- **Single-primary (sync)** — no loss, higher write latency, availability tied to the replica.
- **Multi-primary** — write anywhere, but now you own conflict resolution. Avoid unless you genuinely need multi-region writes.

### Eventual consistency 🔥
Once data is in more than one place, "the current value" is not a single thing. Eventual consistency means: stop writing, and all copies converge.

Useful consistency models in practice:
- **Strong** — every read sees the latest write. Money, inventory, auth.
- **Read-your-own-writes** — you always see *your* changes. The pragmatic default for user-facing apps.
- **Eventual** — everyone converges sooner or later. Likes, view counts, feeds, search indexes, analytics.

Design move: pick consistency **per feature, not per system**. A single e-commerce app is strongly consistent for payment and inventory decrement, and eventually consistent for recommendations, review counts, and search.

---

## Part 5 — Caching a Database 🔥

### Cache-aside (the default, learn this one)
```
read:  value = cache.get(k)
       if miss: value = db.query(); cache.set(k, value, ttl); 
write: db.write(); cache.delete(k)     # delete, don't update
```
Delete rather than update on write — updating races with concurrent readers and writes stale values.

Other strategies (know they exist): **write-through** (write cache + DB together, consistent but slower writes), **write-behind** (fast, risks data loss), **read-through** (library does the fetch for you).

### The three hard problems
1. **Invalidation.** TTL is your safety net for everything you forget to invalidate. Short TTL = fresher + more load; long TTL = staler + cheaper. Choose per key based on *how wrong is acceptable*.
2. **Stampede / thundering herd.** A hot key expires; 10,000 requests miss simultaneously and hammer the database. Fixes: a per-key lock so one request refreshes, "stale-while-revalidate" (serve the old value while one worker refreshes), or jittered TTLs so keys don't expire in lockstep.
3. **Cold start.** After a cache flush or deploy, the DB takes the full load. Warm critical keys, or ramp traffic.

Also know: **eviction policies** (LRU is the sane default), and that **caching is not a fix for a bad query** — it just hides it until the cache misses at the worst possible moment.

---

## Part 6 — NoSQL: Only What Pays 🔥

Do not learn ten databases. Learn **when to choose** among four.

### PostgreSQL / MySQL — the default, and you need a reason not to use it
**Use when:** you have relationships, you need transactions, you want strong consistency, your data has a shape, or you're not sure yet.
Modern Postgres also does JSONB (schema-flexible documents), full-text search, arrays, and `LISTEN/NOTIFY`. **It is very often the correct answer to "should we use MongoDB?" and "do we need Elasticsearch yet?"**
**Not great for:** raw key-value throughput at cache speed, true horizontal write scale without extensions, or free-text relevance ranking at scale.

### Redis — in-memory data structure store
**Use when:** caching, sessions, rate limiting (`INCR` + TTL), distributed locks, leaderboards (sorted sets), simple queues, pub/sub, ephemeral counters.
**Why it wins:** sub-millisecond, single-threaded (so operations are atomic — which is what makes it a great rate limiter and lock), rich data structures.
**Watch out:** it's memory-bound and it *can* lose data (persistence is configurable but not a guarantee). **Never make Redis your only copy of data that matters.** Treat it as derived state you can rebuild.

### MongoDB — document store
**Use when:** genuinely schema-variable documents, each read fetches one self-contained document (a product catalog with wildly different attributes per category; event/activity logs; CMS content), and you want easy horizontal scaling.
**Honest assessment:** most teams that chose MongoDB had relational data and would have been better served by Postgres with a JSONB column. The moment you find yourself doing application-side joins across collections, you picked wrong.
**Key concept:** you model for the *query*, not for the entities — embed what's read together, reference what's large or shared.

### Elasticsearch / OpenSearch — search & analytics engine
**Use when:** full-text search with relevance ranking, faceted search, typo tolerance, log/metric aggregation at scale.
**Critical rule:** it is a **secondary index, never a system of record.** Data lives in Postgres and is indexed into Elasticsearch asynchronously. It's eventually consistent, and losing the cluster must be a re-index, not a data loss.
**Don't reach for it** until Postgres full-text search demonstrably isn't enough. That threshold is further out than people assume.

### The decision rule

> **Start with PostgreSQL. Add Redis when you need speed on hot reads or counters. Add Elasticsearch when you need real search. Add a document store only when your data is genuinely document-shaped. Every additional datastore is a permanent operational tax — backups, monitoring, failover, expertise, and one more thing that can be down at 3 a.m.**

In an interview, *"I'd use Postgres, and here's the specific condition under which I'd add Redis"* is a stronger answer than naming five technologies.

---

## Database mastery checkpoint

You've mastered this section when you can:
- Model a non-trivial domain, defend every constraint, and explain your index choices before seeing the queries
- Read an unfamiliar `EXPLAIN ANALYZE` output and name the problem
- Explain why a specific index isn't being used
- Spot an N+1 in code review by reading the code
- Explain read replicas, replication lag, and how you'd handle read-your-own-writes
- Say when you'd shard — and argue for *not* sharding
- Justify choosing Postgres over MongoDB for a given use case, and vice versa
- Design an idempotent write using a `UNIQUE` constraint

Scenario tests are in [14 — Mastery Checkpoints](./14-mastery-checkpoints.md).

---

## References & Recommended Reading

- **Indexes & Query Performance**:
  - 📖 *Use The Index, Luke!* by Markus Winand ([use-the-index-luke.com](https://use-the-index-luke.com/))
  - 📖 *SQL Performance Explained* by Markus Winand
  - 🛠️ *Explain Dalibo* ([explain.dalibo.com](https://explain.dalibo.com/)) & *depesz EXPLAIN* ([explain.depesz.com](https://explain.depesz.com/))
- **Transactions & Distributed Storage**:
  - 📖 *Designing Data-Intensive Applications (DDIA)* by Martin Kleppmann (Chapters 3, 5, 6, 7)
  - 🎓 *CMU 15-445: Intro to Database Systems* by Andy Pavlo (Lectures on Concurrency Control, B+ Trees, Logging)
- **Hands-On SQL Practice**:
  - 💻 *PGExercises* ([pgexercises.com](https://pgexercises.com/))
  - 📖 *PostgreSQL Official Documentation: Indexes & EXPLAIN* ([postgresql.org/docs](https://www.postgresql.org/docs/current/indexes.html))
- **Caching & In-Memory**:
  - 🎓 *Redis University* ([university.redis.com](https://university.redis.com/)) (RU101: Introduction to Redis Data Structures)

