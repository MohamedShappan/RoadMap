# 02 — The Roadmap (dependency-ordered)

Eight stages. Each one unlocks the next. Do not skip ahead — the ordering is the value.

Times assume ~2 hours/day. Scale by your actual schedule (see [13](./13-study-schedule.md)).

```
STAGE 0 ─ Foundations you actually use ............. 2 weeks
STAGE 1 ─ Databases & SQL .......................... 4 weeks   ★ highest ROI
STAGE 2 ─ Networking for backend devs .............. 2 weeks
STAGE 3 ─ Backend fundamentals ..................... 4 weeks
STAGE 4 ─ Code design & patterns ................... 3 weeks
STAGE 5 ─ DB performance & scaling ................. 3 weeks
STAGE 6 ─ Production engineering ................... 3 weeks
STAGE 7 ─ System design ............................ 6 weeks
                                          total ≈ 27 weeks (~6 months)
```

---

## STAGE 0 — Foundations You Actually Use (2 weeks)

### What to learn
One backend language well enough to stop fighting it: functions, types, interfaces, error handling, collections, JSON serialization, basic concurrency primitives. Plus: hash maps, arrays, sets, and Big-O *intuition*.

### Why it matters
Everything after this is expressed in code. If you're still googling "how do I loop over a map," every later stage costs 3× more. Hash maps deserve special attention — caching, deduplication, grouping, indexing, and rate limiting are all hash maps wearing hats.

### The 20%
- Hash map / array / set: operations and complexity
- Big-O enough to recognize: nested loop over N rows = O(n²) = incident
- Your language's error model (exceptions vs error returns) and how *not* to swallow errors
- JSON encode/decode, including nullable/optional fields
- Concurrency: what a race condition is, what a mutex does, what "this code isn't thread-safe" means
- Git: branch, commit, rebase-or-merge, resolve a conflict

### Safe to skip for now
Dynamic programming, tree/graph algorithms, tries, sorting implementations, LeetCode grinding, "clean code" philosophy books, learning a second language.

### Exercises
1. Implement an LRU cache with a hash map + doubly linked list. (You will meet this again in Redis eviction and in interviews.)
2. Write a function that groups 100k records by a key — first with a nested loop, then with a hash map. Time both. Internalize the difference.
3. Spawn 100 concurrent workers that increment a shared counter. Observe the wrong answer. Fix it with a lock. Understand *why* it was wrong.

### Project
**CLI task manager** storing tasks in a JSON file. Must support add/list/complete/delete/filter, handle a corrupted file gracefully, and have unit tests. Small, but it forces error handling, serialization, and testing.

### Interview questions you should be able to answer
- What's the time complexity of a hash map lookup, and when does it degrade?
- What is a race condition? Give a concrete example.
- Difference between a list and a set? When do you choose each?

### References & Study Sources
- **Data Structures & Big-O**:
  - 📖 *A Common-Sense Guide to Data Structures and Algorithms* by Jay Wengrow (exceptional visual intuition for Big-O, hash tables, and arrays)
- **Language Deep Dive**:
  - 📖 Official docs of your chosen language (e.g., *The Go Programming Language* by Donovan & Kernighan, *Effective Java* by Joshua Bloch, or *The Rust Programming Language*)
- **Concurrency & Threads**:
  - 📄 *Operating Systems: Three Easy Pieces (OSTEP)* — Concurrency virtualization chapters (free at ostep.org)
- **Version Control**:
  - 📖 *Pro Git* by Scott Chacon and Ben Straub (free online at [git-scm.com/book](https://git-scm.com/book))
  - 💻 *Learn Git Branching* ([learngitbranching.js.org](https://learngitbranching.js.org/)) (interactive visual tutorials)

### Mastery signal
You can write a 300-line program with tests without fighting syntax, and you instinctively reach for a hash map instead of a nested loop.

---

## STAGE 1 — Databases & SQL (4 weeks) ★

> Full detail: **[03 — Databases](./03-databases.md)**

### What to learn
Relational modeling, SQL fluency, transactions and isolation, and how indexes actually work.

### Why it matters
This is the highest-leverage stage in the roadmap, for three reasons: (1) most "slow app" problems are database problems; (2) data modeling mistakes are the most expensive mistakes in software, because data outlives code; (3) every system design answer bottoms out at "where does the data live?"

Most developers are weak here. Being strong here makes you visibly senior.

### The 20%
- Tables, PKs, FKs, and modeling 1:1, 1:N, N:M (junction tables)
- Constraints — and the mindset that the database, not your code, is the final guarantor of data integrity
- SQL: JOIN (inner/left), GROUP BY + HAVING, aggregates, subqueries, `EXISTS`
- Transactions: ACID, what a rollback really undoes, transaction boundaries in app code
- Isolation levels: READ COMMITTED vs REPEATABLE READ, and dirty/non-repeatable/phantom reads
- Indexes: B-tree structure, composite index column order, selectivity, the cost of writes
- Normalization to 3NF; when to denormalize on purpose
- Connection pooling: why opening a connection per request is fatal

### Safe to skip for now
Stored procedures, triggers, recursive CTEs, window functions (revisit in stage 5), DB internals (WAL format, page layout), NoSQL of any kind, ORMs as a study topic (use one, don't study it).

### Exercises
1. Model an e-commerce schema by hand: users, products, categories, orders, order_items, payments. Draw the ERD. Write the DDL with every constraint you can justify.
2. Load ~1M fake rows. Write 15 queries of increasing difficulty: top customers by spend, monthly revenue, products never ordered, second-highest order value per user.
3. Open two psql sessions. Start a transaction in each. Reproduce: a dirty-read-prevented case, a lost update, and a real deadlock. Read the error message carefully.
4. Take your slowest query. Run `EXPLAIN ANALYZE`. Add an index. Run it again. Write down the before/after numbers.
5. Deliberately create a composite index on `(a, b)` and prove it helps `WHERE a = ? AND b = ?` and `WHERE a = ?` but not `WHERE b = ?`.

### Project
**Library management schema + query layer.** No HTTP yet. Books, authors, members, loans, reservations. Enforce "a copy can't be loaned twice simultaneously" *at the database level*. Write a loan-checkout function that uses a transaction correctly. Add indexes justified by `EXPLAIN` output.

### Interview questions
- What is an index, and how does it make reads fast? What does it cost?
- What is a transaction? What does ACID actually guarantee?
- What's the difference between READ COMMITTED and REPEATABLE READ?
- What causes a deadlock, and how do you prevent one?
- INNER vs LEFT JOIN — give a case where the choice changes the answer.
- Why would you denormalize?

### References & Study Sources
- **Relational Modeling & Schema Design**:
  - 📖 *Database Design for Mere Mortals* by Michael J. Hernandez (approachable guide to normalization, keys, and table relationships)
  - 📖 *PostgreSQL Official Documentation: Data Definition & Constraints* ([postgresql.org/docs](https://www.postgresql.org/docs/current/ddl-constraints.html))
- **SQL Practice**:
  - 💻 *PGExercises* ([pgexercises.com](https://pgexercises.com/)) (interactive exercises with immediate execution on PostgreSQL data)
  - 💻 *SQLBolt* ([sqlbolt.com](https://sqlbolt.com/)) (interactive query drills for joins, filters, and aggregations)
- **Transactions & Concurrency**:
  - 📖 *Designing Data-Intensive Applications (DDIA)* by Martin Kleppmann — Chapter 7: *Transactions* (authoritative coverage of ACID, race conditions, dirty/phantom reads, and isolation levels)
  - 🎓 *CMU 15-445: Intro to Database Systems* by Andy Pavlo (Lectures on Concurrency Control and Two-Phase Locking on YouTube)

### Mastery signal
Given a business description, you can produce a sane normalized schema in 20 minutes, defend every constraint, and explain which indexes you'd add and why — *before* seeing the queries.

---

## STAGE 2 — Networking for Backend Developers (2 weeks)

> Full detail: **[05 — Networking](./05-networking.md)**

### What to learn
Everything that happens between two processes talking, and how it fails.

### Why it matters
Distributed systems are just "networked systems," and the network is the part that lies to you: it's slow, it drops things, and it fails in ways local code never does. Every timeout, retry, circuit breaker and idempotency key in later stages exists because of facts you learn here.

### The 20%
- The complete lifecycle of `https://example.com` — be able to narrate it for 5 minutes straight
- IP, ports, sockets: what a connection actually *is* (a 4-tuple)
- TCP: handshake, ordering, reliability, backpressure. UDP: when you'd take the trade.
- DNS: resolution path, record types, TTL, and why a DNS change "didn't take effect"
- HTTP: methods, idempotency/safety of verbs, status code families, key headers
- TLS conceptually: what it guarantees (confidentiality, integrity, identity), handshake cost, why you terminate at the LB
- Load balancer vs reverse proxy vs forward proxy
- Latency vs bandwidth — and why 100 sequential 10ms calls is a 1-second page
- Timeouts (connect vs read), retries, keep-alive, connection pooling
- Failure vocabulary: connection refused vs timeout vs DNS failure vs 502 vs 503 vs 504

### Safe to skip for now
OSI-layer memorization, subnetting arithmetic, BGP/routing protocols, Ethernet/ARP, writing socket code from scratch, HTTP/2 frame formats.

### Exercises
1. `dig example.com`, `dig +trace example.com`. Explain every hop.
2. `curl -v https://example.com`. Annotate every line of the verbose output.
3. Capture a TCP handshake with `tcpdump`/Wireshark. Find SYN, SYN-ACK, ACK.
4. Run a server, then: (a) connect to a closed port → *connection refused*; (b) connect to a firewalled/blackholed IP → *connection timeout*. Feel the difference.
5. Write an HTTP client with a 2-second timeout hitting an endpoint that sleeps 5 seconds. Then add 3 retries with exponential backoff + jitter.
6. Put nginx in front of two app instances. Kill one. Watch traffic shift. Kill both → observe a 502.

### Project
**A tiny reverse proxy** in your language: listen on a port, forward to N backends round-robin, add health checks that eject unhealthy backends, apply a per-backend timeout, and add an `X-Request-Id` header. ~200 lines, and it will permanently demystify load balancers.

### Interview questions
- What happens when you type a URL and hit enter?
- TCP vs UDP — and name a real system that uses each.
- What is DNS? What is a TTL and why does it bite you?
- What does TLS give you? Why is the handshake expensive?
- What's a reverse proxy, and what do you use one for?
- What causes a 502? A 503? A 504? How do they differ?
- Connection refused vs connection timeout — what does each tell you about the failure?

### References & Study Sources
- **Transport & Web Protocols**:
  - 📖 *High Performance Browser Networking (HPBN)* by Ilya Grigorik (free online at [hpbn.co](https://hpbn.co/)) — Chapters on TCP, UDP, TLS, and HTTP/1.1 to HTTP/2/3
  - 📖 *Computer Networking: A Top-Down Approach* by Kurose & Ross (Application and Transport Layer chapters)
- **Production Architecture & Proxies**:
  - 📄 *Cloudflare Learning Center* ([cloudflare.com/learning](https://www.cloudflare.com/learning/)) — clear guides on DNS resolution, CDN, reverse proxies, and TLS handshakes
  - 📄 *MDN Web Docs: HTTP* ([developer.mozilla.org/en-US/docs/Web/HTTP](https://developer.mozilla.org/en-US/docs/Web/HTTP)) — status code reference, headers, and caching mechanisms
- **Hands-On Inspection Tools**:
  - 🛠️ `curl -v`, `dig +trace`, and Wireshark / `tcpdump` for packet inspection

### Mastery signal
Given "users report the site is slow," you can name six distinct layers where it could be, and the command you'd run to rule each one in or out.

---

## STAGE 3 — Backend Fundamentals (4 weeks)

> Full detail: **[08 — Backend Engineering](./08-backend-engineering.md)**

### What to learn
How to build an API that a team could actually run in production: design, auth, validation, errors, logging, config, testing, transactions.

### Why it matters
This is your actual job. Stages 1 and 2 gave you the substrate; this is where you assemble it into something that serves requests, doesn't leak secrets, doesn't corrupt data, and can be debugged by someone else.

### The 20%
- REST design: resource nouns, correct verbs, correct status codes, pagination, filtering
- Request validation at the edge; never trust a client
- A single consistent error contract (`{code, message, details, request_id}`) and mapping exceptions to it
- AuthN vs AuthZ; sessions vs JWT (and JWT's revocation problem); password hashing with bcrypt/argon2
- Layering: handler → service → repository. Business logic never in the handler, SQL never in the service.
- Dependency Injection — pass dependencies in, don't construct them inside
- Config from environment; secrets never committed
- Structured logging with a request ID threaded through every log line
- Testing: fast unit tests for logic, integration tests against a *real* database in Docker
- Transaction boundaries: one business operation = one transaction, owned by the service layer
- Background jobs: what must not happen inside a request
- Idempotency keys for unsafe operations

### Safe to skip for now
GraphQL, gRPC, microservices, event sourcing, CQRS, hexagonal architecture as doctrine, OAuth *implementation* (concepts only), Kubernetes.

### Exercises
1. Take an endpoint and write out its full contract: request schema, response schema, every possible status code, and what each means.
2. Implement auth twice — once with server-side sessions, once with JWT. Then implement logout for both. The pain you feel in the JWT version *is* the lesson.
3. Add request-ID middleware; make a log line from deep in the service layer carry it.
4. Write one unit test with a fake repository and one integration test against Postgres in Docker. Note which one catches a broken SQL query.
5. Create a "transfer money between accounts" service method. Make it correct under concurrency. Prove it with a concurrent test.
6. Move email sending out of the request path into a background job.

### Project
**Production-quality REST API** (see [Project 1](./10-projects.md)). Auth, CRUD, validation, error contract, pagination, structured logs, config, migrations, unit + integration tests, Dockerfile, README with setup steps.

### Interview questions
- How do you design a REST API for X? What status codes and why?
- 401 vs 403? 400 vs 422? When is 409 right?
- Sessions vs JWT — trade-offs? How do you revoke a JWT?
- Where do you put business logic, and why not in the controller?
- What do you unit test vs integration test?
- Where does a transaction begin and end in your codebase?
- How do you make a payment endpoint safe to retry?

### References & Study Sources
- **REST API Design**:
  - 📖 *Build APIs You Won't Hate* by Phil Sturgeon (practical guide on idempotency, status codes, serialization, and pagination)
  - 📄 *Google Cloud API Design Guide* ([cloud.google.com/apis/design](https://cloud.google.com/apis/design)) & *Microsoft REST API Guidelines*
- **Authentication & Security Standards**:
  - 📄 *OWASP Authentication Cheat Sheet* & *JSON Web Token (JWT) Cheat Sheet* ([cheatsheetseries.owasp.org](https://cheatsheetseries.owasp.org/))
  - 📄 *RFC 6749 (OAuth 2.0 Authorization Framework)* for mental models
- **Testing & Application Architecture**:
  - 📖 *Unit Testing Principles, Practices, and Patterns* by Vladimir Khorikov (definitive guide on unit vs integration tests, mocks vs stubs vs fakes)
  - 📄 *The Twelve-Factor App* ([12factor.net](https://12factor.net/)) (config, environment separation, backing services)

### Mastery signal
You could hand your project to another developer with only the README, and they could run it, understand the layering, and add an endpoint without asking you questions.

---

## STAGE 4 — Code Design & Patterns (3 weeks)

> Full detail: **[04 — Design Patterns](./04-design-patterns.md)**

### What to learn
The ten patterns you'll actually use, plus the skill of *recognizing* which problem shape you're in.

### Why it matters
Deliberately placed after Stage 3: patterns are answers to pain, and you now have pain. Your Stage 3 project has an `if provider == "stripe"` somewhere, a constructor that news-up a database client, and a service that's impossible to test. Those are Strategy, DI, and Repository asking to be born.

### The 20%
- **Dependency Injection** — the single highest-value pattern; makes everything testable
- **Strategy** — "same operation, interchangeable algorithms"
- **Repository** — isolate persistence from business logic
- **Factory** — centralize conditional construction
- **Adapter** — make a third-party interface fit yours; shield your core from vendor churn
- **Decorator / middleware** — layer cross-cutting behavior (logging, retry, cache, auth)
- **Observer / pub-sub** — one event, many independent reactions
- SRP + Dependency Inversion; composition over inheritance
- **Pattern recognition**: reading a problem and hearing which pattern it's shaped like

### Safe to skip for now
The other 15 GoF patterns, UML, Event Sourcing/CQRS, DDD strategic design, formal Clean Architecture, "rewrite everything with patterns."

### Exercises
1. Find every `if type == ...` / `switch` on a type in your Stage 3 project. Refactor one into Strategy. Notice what got better *and* what got more indirect.
2. Add a second payment provider without touching existing provider code (Strategy + Factory).
3. Wrap your repository in a caching decorator. Business logic must not change at all.
4. Replace direct instantiation with constructor injection across a service. Then write a test with a fake — notice that it's now trivial.
5. Add an event bus: on `OrderPlaced`, fire email + analytics + inventory handlers, none of which know about each other.
6. **Anti-exercise:** deliberately over-apply patterns to a 50-line script. Feel the damage. This inoculates you against pattern addiction.

### Project
Refactor your Stage 3 API: swappable storage backends, pluggable auth, middleware chain, event-driven side effects — without breaking a single test. **The tests passing unchanged is the proof the refactor was structural, not behavioral.**

### Interview questions
- When would you use Strategy? How is it different from just an if/else?
- Factory vs Builder?
- Adapter vs Decorator?
- Why Dependency Injection? What does it buy you concretely?
- When is a design pattern the wrong choice?
- What does "composition over inheritance" mean in practice?

### References & Study Sources
- **Pattern Guides & Recognition**:
  - 📖 *Head First Design Patterns* by Eric Freeman & Elisabeth Robson (the most intuitive, anti-academic walkthrough of Strategy, Observer, Decorator, and Factory)
  - 💻 *Refactoring.Guru: Design Patterns* ([refactoring.guru/design-patterns](https://refactoring.guru/design-patterns)) (visual catalogs, structure diagrams, and real-world code implementations)
- **Refactoring & Clean Architecture**:
  - 📖 *Refactoring to Patterns* by Joshua Kerievsky (shows how to evolve messy conditionals into patterns naturally rather than upfront over-engineering)
  - 📄 Martin Fowler's Architecture & Design Catalog ([martinfowler.com](https://martinfowler.com/)) — articles on Repository, Gateway, and Dependency Injection

### Mastery signal
You can look at a requirement — "we need to support three shipping calculators" — and immediately say "Strategy, injected via a factory keyed on carrier," *and* you can also say when it's not worth it.

---

## STAGE 5 — Database Performance & Scaling (3 weeks)

> Full detail: **[03 — Databases](./03-databases.md), performance & scaling sections**

### What to learn
How to answer "why is this query slow?" under pressure, and what to do when one database isn't enough.

### Why it matters
This is the skill that separates a mid-level engineer from a senior one in the eyes of everyone around them. Being the person who can open a query plan and find the problem in 10 minutes is career-defining.

### The 20%
- `EXPLAIN ANALYZE`: seq scan vs index scan vs bitmap scan, nested loop vs hash join, estimated vs actual rows
- Why an index isn't used: wrong column order, function on the column, low selectivity, type mismatch, small table
- Composite index design and the leftmost-prefix rule
- N+1 detection and elimination (eager loading, batch fetch, `IN` queries)
- Pagination at scale: keyset over OFFSET
- Locks in practice: long transactions, lock waits, deadlock loops
- Connection pool sizing (pool exhaustion presents as "the app is slow," not as a DB error)
- Read replicas + replication lag + read-your-own-writes
- Caching: cache-aside, TTL choice, invalidation, stampede protection
- Partitioning vs sharding — what each solves and what each costs

### Safe to skip for now
Writing your own sharding layer, DB internals tuning (`shared_buffers`, vacuum tuning), multi-region active-active, Vitess/Citus specifics.

### Exercises
1. Seed a 10M-row table. Write a query that takes >5 seconds. Fix it to <50ms. Document every step of the reasoning.
2. Build a deliberate N+1 (list 100 orders, fetch each order's user). Measure. Fix it. Measure again.
3. Implement OFFSET pagination on 10M rows; time page 1 vs page 100,000. Then implement keyset pagination and compare.
4. Add a cache-aside layer to your hottest endpoint. Then deliberately create a stale-data bug. Then fix it with proper invalidation.
5. Set up a Postgres read replica with Docker. Route reads to it. Then write-then-immediately-read and observe the stale read. Fix it (read-your-own-writes from primary).
6. Hold a transaction open for 60 seconds while another updates the same row. Watch the lock wait. Then engineer a deadlock.
7. Set your connection pool to 2. Fire 50 concurrent requests. Observe how pool exhaustion *feels* from the client side.

### Project
**Performance clinic.** Take your Stage 3/4 API, seed it with realistic volume (1M+ rows), load test it, and produce a written report: baseline p50/p95/p99, the three biggest bottlenecks found via query plans, the fix for each, and the after numbers. This report is interview gold.

### Interview questions
- Walk me through diagnosing a slow query.
- You added an index and nothing got faster. Why?
- What's an N+1 query and how do you spot one?
- Why is `OFFSET 1000000` slow?
- How do read replicas help, and what do they break?
- When do you shard, and what do you lose?
- Cache invalidation strategies — which do you pick and why?

### References & Study Sources
- **Indexing & Query Performance**:
  - 📖 *Use The Index, Luke!* by Markus Winand ([use-the-index-luke.com](https://use-the-index-luke.com/)) (the definitive reference on B-trees, composite index ordering, and index-only scans)
  - 📖 *SQL Performance Explained* by Markus Winand (the condensed physical book covering performance tuning on PostgreSQL & MySQL)
  - 🛠️ *Explain Dalibo* ([explain.dalibo.com](https://explain.dalibo.com/)) & *depesz EXPLAIN* ([explain.depesz.com](https://explain.depesz.com/)) (tools to visualize and diagnose `EXPLAIN (ANALYZE, BUFFERS)` trees)
- **Replication, Partitioning & Scaling**:
  - 📖 *Designing Data-Intensive Applications (DDIA)* by Martin Kleppmann — Chapter 5: *Replication* (replication lag, topologies) & Chapter 6: *Partitioning* (sharding strategies and secondary indexes)
- **Caching & In-Memory Stores**:
  - 🎓 *Redis University* ([university.redis.com](https://university.redis.com/)) — Course *RU101: Introduction to Redis Data Structures*
  - 📄 *AWS Whitepaper: Database Caching Strategies Using Redis* (thundering herd, TTLs, and cache-aside patterns)

### Mastery signal
Given a query plan you've never seen, you can point at the expensive node and propose two fixes with a trade-off for each.

---

## STAGE 6 — Production Engineering (3 weeks)

> Full detail: **[09 — Production Engineering](./09-production-engineering.md)**

### What to learn
Observability and resilience: knowing what your system is doing, and keeping it alive when dependencies aren't.

### Why it matters
Code that works on your laptop is a hobby. Code you can operate — observe, debug, deploy safely, and degrade gracefully — is a profession. This stage is also what makes System Design answers credible: anyone can draw a box labeled "service"; you'll be able to say how you'd know it was broken.

### The 20%
- Structured logs, correlation IDs, appropriate levels, never logging secrets/PII
- Metrics: the four golden signals; counters vs gauges vs histograms; p50/p95/p99
- Tracing conceptually: a request's path across services
- Health checks: liveness vs readiness (and why conflating them causes outages)
- Graceful shutdown: stop accepting, drain in-flight, close pools
- Timeouts on *every* outbound call; retries with exponential backoff + jitter; retry only idempotent work
- Circuit breakers: fail fast instead of piling up
- Idempotency in production workflows
- Docker: image vs container, layer caching, compose for local dev
- Config & secrets: 12-factor env config, secret managers
- CI: run tests on every push, build the image, deploy on green
- Security baseline: OWASP top items, parameterized queries, TLS, least privilege, dependency scanning

### Safe to skip for now
Kubernetes beyond core objects, service meshes, Terraform mastery, chaos engineering programs, custom Prometheus exporters, cost optimization.

### Exercises
1. Add request-ID-threaded structured logging; trace one request end to end through the logs.
2. Expose `/metrics` with request count, error count, and a latency histogram. Graph p99.
3. Implement `/health/live` and `/health/ready`. Make readiness go false when the DB is unreachable.
4. Implement graceful shutdown; verify no request is dropped during a restart under load.
5. Wrap an external API call with timeout + 3 retries + backoff + jitter + a circuit breaker. Simulate the dependency being down. Verify you fail fast.
6. Containerize the app; `docker compose up` must bring up app + Postgres + Redis with one command.
7. Write a CI pipeline: lint → unit → integration (with services) → build image.
8. Attack your own API: SQL injection attempt, IDOR (fetch another user's resource by ID), missing auth on one endpoint. Fix what you find.

### Project
Make your Stage 3–5 API **operable**: full observability, resilience patterns, Docker Compose, CI pipeline, a `RUNBOOK.md` describing what to check when latency spikes or errors rise.

### Interview questions
- How do you debug a production issue you can't reproduce locally?
- Why percentiles instead of averages?
- Liveness vs readiness probe?
- When is retrying dangerous?
- What's a circuit breaker and why not just retry more?
- How do you handle secrets?
- Name three common web vulnerabilities and their fixes.

### References & Study Sources
- **Observability & Reliability**:
  - 📖 *Site Reliability Engineering (SRE) Book* by Google ([sre.google/sre-book/](https://sre.google/sre-book/)) — Chapter 6: *Monitoring Distributed Systems* (the Four Golden Signals)
  - 📖 *Release It! Design and Deploy Production-Ready Software* by Michael T. Nygard (essential reading on Circuit Breakers, Bulkheads, Timeouts, and cascade failures)
  - 📄 *OpenTelemetry Documentation* ([opentelemetry.io/docs](https://opentelemetry.io/docs/)) — concepts of Traces, Spans, Metrics, and Correlation IDs
- **Containers & Operations**:
  - 📖 *Docker Deep Dive* by Nigel Poulton & Official Docker Guides ([docs.docker.com/get-started](https://docs.docker.com/get-started/))
- **Production Web Security**:
  - 📄 *OWASP Top 10 Security Risks* ([owasp.org/Top10/](https://owasp.org/Top10/)) — focus on SQL injection, Broken Object-Level Authorization (IDOR), and Security Misconfigurations

### Mastery signal
Something breaks in your own project and you find the cause from logs and metrics alone — without adding a print statement.

---

## STAGE 7 — System Design (6 weeks)

> Full detail: **[06 — System Design](./06-system-design.md)** and **[07 — Framework + Worked Systems](./07-system-design-framework.md)**

### What to learn
Compose everything: building blocks, scaling techniques, distributed-systems realities, a repeatable framework, then 10 worked designs.

### Why it matters
System design is how senior engineering is evaluated — in interviews and in real architectural decisions. Because you did stages 1–6 first, every box you draw is backed by knowledge you can defend when the interviewer pushes.

### Structure of the stage
- **Weeks 1–2 — Building blocks:** client, API gateway, LB, app tier, DB, cache, queue, object store, CDN. What each does, costs, and fails at.
- **Week 3 — Scaling toolkit:** horizontal scaling, statelessness, caching layers, replication, read/write split, queues & async, rate limiting, idempotency, retries, circuit breakers, backpressure.
- **Week 4 — Distributed realities:** CAP practically, consistency models, eventual consistency, distributed transactions and sagas, the outbox pattern, observability across services.
- **Weeks 5–6 — The framework + drills:** learn the framework in [07](./07-system-design-framework.md), then design 10 systems, 2–3 per week, 45 minutes each, out loud, on a whiteboard.

### The 20%
Estimation, statelessness, caching, replication, queues, idempotency, consistency trade-offs, and the framework itself. If you're strong on those seven, you can survive almost any design question.

### Safe to skip for now
Raft/Paxos internals, CRDTs, vector clocks, exotic architectures, memorizing any specific company's real architecture.

### Exercises
1. Back-of-envelope drills: 10M DAU, 20 posts read each → QPS? Storage for 1B images at 200KB? Bandwidth for 100k concurrent 1080p streams? Do five of these until estimation is reflexive.
2. Take your Stage 3 API and write "how would this survive 100× traffic?" — identify the first, second, and third things to break.
3. Draw 10 systems from [07](./07-system-design-framework.md) on paper, 45 min each, speaking aloud. Record yourself once; watch it. It will be humbling and useful.
4. For each design, force yourself to name three trade-offs you *chose* and what you gave up.

### Project
**Project 5 and 6** in [10 — Projects](./10-projects.md): build a distributed order-processing system (real code: queue, workers, idempotency, outbox, retries, observability), then produce a full written design document for a large-scale system.

### Interview questions
- Design a URL shortener / chat app / news feed / notification system.
- How do you scale a database?
- When would you use Redis? When is it the wrong tool?
- Why a message queue instead of a direct call?
- What is eventual consistency, and when is it unacceptable?
- What is idempotency and how do you implement it?
- How would you handle 1M requests per second?
- CAP theorem — and what do you actually pick in practice?

### References & Study Sources
- **System Design Core Guides & Interview Primers**:
  - 📖 *System Design Interview – An Insider's Guide* (Volume 1 & 2) by Alex Xu (structured blueprints for rate limiters, key-value stores, distributed message queues, notification services)
  - 💻 *The System Design Primer* by Donne Martin ([github.com/donnemartin/system-design-primer](https://github.com/donnemartin/system-design-primer)) (comprehensive open-source collection of system design topics, diagrams, and exercises)
- **Deep Distributed Systems Principles**:
  - 📖 *Designing Data-Intensive Applications (DDIA)* by Martin Kleppmann (the industry standard for data systems, batch/stream processing, and consistency models)
- **Real-World Engineering Case Studies**:
  - 📄 *High Scalability Architecture Case Studies* ([highscalability.com](http://highscalability.com/))
  - 📄 Engineering blogs: Uber Engineering Blog, Netflix TechBlog, Meta Engineering, and Stripe Engineering

### Mastery signal
You can take an unfamiliar prompt, run the framework in 45 minutes, defend every component against "why not X?", name the bottleneck at each scale tier, and explicitly state what you traded away.

---

## After Stage 7

You are now a strong practical backend engineer. Growth from here is not more topics — it's depth through real production experience, plus selective 🟡 items from [01](./01-knowledge-map.md) as your work demands them. Resist the urge to start collecting ⚪ topics.
