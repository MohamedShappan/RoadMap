# 01 — The 80/20 Knowledge Map

Every topic, classified. Be ruthless about respecting these tiers.

- 🔥 **MUST KNOW** — the small set that provides the majority of practical value. Learn these to real depth.
- 🟡 **SHOULD KNOW** — genuinely important, but only after 🔥 is solid. Roughly 6–18 months in.
- ⚪ **NICE TO KNOW** — real knowledge, low priority. Learn on demand when a job forces it.

A rough count: ~70 🔥 items across all domains. That is the whole game. Everything else can wait.

---

## 1. Programming & Data Structures

| Tier | Topic | Why |
|---|---|---|
| 🔥 | Arrays/lists, hash maps, sets | 90% of real code. Hash map is the single most-used structure in backend work. |
| 🔥 | Big-O intuition (O(1), O(log n), O(n), O(n²)) | Enough to spot a nested loop over a DB result set. |
| 🔥 | Strings, serialization (JSON) | Every API boundary. |
| 🔥 | Error handling idioms of your language | Half of production quality is error handling. |
| 🔥 | Concurrency basics: threads/goroutines/async, race conditions, locks | You will hit these in week one of a real job. |
| 🟡 | Trees & graph traversal (BFS/DFS) | Occasional; needed for interviews and for understanding B-trees. |
| 🟡 | Queues, stacks, heaps | Show up in rate limiters, schedulers, priority work. |
| 🟡 | Sorting/searching semantics (not implementations) | Know `sort` is O(n log n) and binary search needs order. |
| ⚪ | Advanced DS: tries, union-find, segment trees, DP | Competitive programming, not backend work. |
| ⚪ | Manual memory management, custom allocators | Systems programming niche. |

## 2. Databases (highest-ROI domain in this roadmap)

| Tier | Topic |
|---|---|
| 🔥 | Tables, primary keys, foreign keys, 1:1 / 1:N / N:M relationships |
| 🔥 | Constraints: NOT NULL, UNIQUE, CHECK, FK — and why the DB is your last line of defense |
| 🔥 | SQL: SELECT/WHERE/ORDER/LIMIT, INNER & LEFT JOIN, GROUP BY + aggregates, subqueries |
| 🔥 | Transactions, ACID, COMMIT/ROLLBACK |
| 🔥 | Isolation levels (READ COMMITTED vs REPEATABLE READ) + the anomalies they prevent |
| 🔥 | Indexes: B-tree, composite indexes + column order, covering indexes, selectivity |
| 🔥 | `EXPLAIN` / `EXPLAIN ANALYZE` — reading a query plan, seq scan vs index scan |
| 🔥 | N+1 query problem |
| 🔥 | Normalization to 3NF, then deliberate denormalization |
| 🔥 | Connection pooling |
| 🔥 | Locks & deadlocks: why they happen, how to avoid them (consistent lock ordering) |
| 🔥 | Keyset ("seek") pagination vs OFFSET |
| 🔥 | Read replicas + replication lag |
| 🔥 | Caching a database (cache-aside, TTL, invalidation) |
| 🔥 | When to use PostgreSQL vs Redis vs MongoDB vs Elasticsearch |
| 🟡 | Partitioning (single-DB, by range/hash) |
| 🟡 | Sharding: shard keys, cross-shard queries, resharding pain |
| 🟡 | Eventual consistency & read-your-own-writes |
| 🟡 | MVCC, vacuum/bloat, hot vs cold data |
| 🟡 | Window functions, CTEs |
| 🟡 | Materialized views, DB-level full-text search |
| 🟡 | Optimistic vs pessimistic locking |
| ⚪ | Stored procedures, triggers |
| ⚪ | Recursive CTEs, advanced index types (GiST, BRIN, bitmap) |
| ⚪ | Database internals: WAL format, page layout, buffer pool tuning |
| ⚪ | Graph/time-series/column-store databases |

## 3. Computer Networking

| Tier | Topic |
|---|---|
| 🔥 | The full lifecycle of `https://example.com` (DNS → TCP → TLS → HTTP → LB → app → DB → response) |
| 🔥 | IP addresses, ports, sockets |
| 🔥 | TCP vs UDP — handshake, ordering, reliability, when each is used |
| 🔥 | DNS: records (A, CNAME), resolvers, TTL, caching |
| 🔥 | HTTP: methods, status codes, headers, request/response anatomy |
| 🔥 | HTTPS/TLS conceptually: what it guarantees, certificates, handshake cost |
| 🔥 | REST conventions |
| 🔥 | Load balancers & reverse proxies (and the difference) |
| 🔥 | Latency vs bandwidth vs throughput |
| 🔥 | Timeouts (connect vs read), retries, connection pooling/keep-alive |
| 🔥 | Diagnosing: connection refused, connection timeout, DNS failure, 502/503/504 |
| 🟡 | HTTP/1.1 keep-alive & head-of-line blocking; HTTP/2 multiplexing; HTTP/3 over QUIC (conceptual) |
| 🟡 | WebSockets vs polling vs SSE |
| 🟡 | CDN: edge caching, cache headers, origin shielding |
| 🟡 | NAT, firewalls, security groups, private vs public subnets |
| 🟡 | mTLS, TLS termination points |
| ⚪ | OSI 7-layer model memorization |
| ⚪ | Subnetting math / CIDR arithmetic by hand |
| ⚪ | BGP, routing protocols, ARP, Ethernet frames |
| ⚪ | Writing a TCP stack, raw socket programming |

## 4. Backend Engineering

| Tier | Topic |
|---|---|
| 🔥 | REST API design: resources, verbs, status codes, pagination, filtering |
| 🔥 | Input validation at the boundary |
| 🔥 | Error handling & a consistent error response contract |
| 🔥 | Authentication vs Authorization |
| 🔥 | Sessions vs JWT — real trade-offs (revocation!) |
| 🔥 | Password hashing (bcrypt/argon2), never plaintext |
| 🔥 | Layered architecture: handler → service → repository |
| 🔥 | Dependency Injection |
| 🔥 | Configuration via environment; secrets never in code |
| 🔥 | Structured logging |
| 🔥 | Unit tests vs integration tests; what belongs where |
| 🔥 | Transactions at the service boundary |
| 🔥 | Background jobs & queues |
| 🔥 | Caching in the app layer |
| 🔥 | Idempotency keys |
| 🟡 | OAuth2 / OIDC conceptually (authorization code flow) |
| 🟡 | RBAC and permission modeling |
| 🟡 | API versioning strategies |
| 🟡 | Rate limiting (token bucket, sliding window) |
| 🟡 | File/object storage + presigned URLs |
| 🟡 | Concurrency: optimistic locking, worker pools, connection limits |
| 🟡 | Contract testing, test data builders, fakes vs mocks |
| ⚪ | GraphQL, gRPC (learn when a job needs it) |
| ⚪ | Custom serialization formats, protocol design |
| ⚪ | Building your own framework/ORM |

## 5. Design Patterns & Software Engineering Principles

| Tier | Topic |
|---|---|
| 🔥 | Dependency Injection / Inversion of Control |
| 🔥 | Strategy |
| 🔥 | Repository |
| 🔥 | Factory (simple factory + factory method) |
| 🔥 | Adapter |
| 🔥 | Decorator / middleware chains |
| 🔥 | Observer / pub-sub |
| 🔥 | Single Responsibility + Dependency Inversion (the two SOLID letters that pay) |
| 🔥 | Composition over inheritance |
| 🔥 | Separation of concerns / layering |
| 🔥 | Recognizing pattern *shapes* in problems (the real skill) |
| 🟡 | Builder |
| 🟡 | Command |
| 🟡 | Template Method |
| 🟡 | Facade, Proxy |
| 🟡 | Open/Closed, Liskov, Interface Segregation |
| 🟡 | Refactoring vocabulary (extract method/class, introduce seam) |
| 🟡 | Domain-Driven Design *tactical* ideas (entity, value object, aggregate) |
| ⚪ | The other 12+ GoF patterns (Flyweight, Bridge, Memento, Visitor…) |
| ⚪ | Full DDD strategic design, CQRS, Event Sourcing |
| ⚪ | Hexagonal/Clean/Onion architecture as formal doctrine |

## 6. System Design

| Tier | Topic |
|---|---|
| 🔥 | Building blocks: client, API gateway, LB, app server, DB, cache, queue, object storage, CDN |
| 🔥 | Horizontal vs vertical scaling |
| 🔥 | Stateless services (and why state kills scaling) |
| 🔥 | Caching strategies + invalidation + stampede |
| 🔥 | DB replication, read/write separation |
| 🔥 | Message queues & async processing |
| 🔥 | Idempotency |
| 🔥 | Timeouts, retries + exponential backoff + jitter |
| 🔥 | Rate limiting |
| 🔥 | Back-of-envelope estimation (QPS, storage, bandwidth) |
| 🔥 | Consistency vs availability trade-offs; CAP at a practical level |
| 🔥 | Eventual consistency |
| 🔥 | Single points of failure; redundancy |
| 🔥 | Observability: logs, metrics, traces |
| 🔥 | A repeatable design framework |
| 🟡 | Sharding & partitioning strategies |
| 🟡 | Circuit breakers, bulkheads, backpressure |
| 🟡 | Event-driven architecture; outbox pattern |
| 🟡 | Saga / distributed transactions (and why to avoid 2PC) |
| 🟡 | Consistent hashing |
| 🟡 | Leader election, quorum, replication lag handling |
| 🟡 | CDC (change data capture) |
| 🟡 | Search systems (inverted index conceptually) |
| ⚪ | Paxos/Raft internals |
| ⚪ | Vector clocks, CRDTs |
| ⚪ | Lambda/Kappa architectures, big data stacks |
| ⚪ | Designing a database/consensus system from scratch |

## 7. Production Engineering, Scalability, Reliability, Security

| Tier | Topic |
|---|---|
| 🔥 | Structured logs with correlation/request IDs |
| 🔥 | The four golden signals: latency, traffic, errors, saturation |
| 🔥 | Percentiles (p50/p95/p99) and why averages lie |
| 🔥 | Health checks (liveness vs readiness) |
| 🔥 | Graceful shutdown |
| 🔥 | Timeouts & retries everywhere a network call happens |
| 🔥 | Docker basics: image, container, Dockerfile, compose |
| 🔥 | Environment config + secret management basics |
| 🔥 | OWASP essentials: SQL injection, XSS, CSRF, broken auth, IDOR |
| 🔥 | TLS everywhere; hashing vs encryption |
| 🔥 | A basic CI pipeline (test → build → deploy) |
| 🟡 | Distributed tracing |
| 🟡 | Circuit breakers in practice |
| 🟡 | Kubernetes core objects (pod, deployment, service, ingress, configmap/secret) |
| 🟡 | Blue/green & canary deploys, feature flags |
| 🟡 | Alerting on symptoms, runbooks, on-call basics |
| 🟡 | Load testing & profiling |
| ⚪ | Writing Helm charts / operators / service meshes |
| ⚪ | Terraform mastery, multi-cloud, cost engineering |
| ⚪ | Chaos engineering programs |
| ⚪ | Kernel tuning, eBPF |

---

## The discipline rule

When you find yourself studying something, ask: **which tier is this?** If it's ⚪ and you haven't finished 🔥, you are procrastinating with extra steps. This is the single most common way people lose a year.
