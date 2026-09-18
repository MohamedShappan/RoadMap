# 12 — The "DO NOT LEARN YET" List

> Possibly the most valuable file here. Every item below is something smart, motivated developers routinely spend months on, at a stage where it returns almost nothing. Each entry: **why you can skip it, when it becomes useful, and what to do instead.**

The meta-pattern: most of these are **solutions to problems you don't have yet.** A solution without its problem is just trivia — it doesn't stick, and it crowds out the things that would.

---

## Architecture & distributed systems

### Kubernetes (beyond the basics)
**Why skip:** it solves orchestration of many services across many machines. With one service and one database, it is pure overhead, and a huge amount of its surface area (operators, CRDs, Helm authoring, service meshes, cluster admin) is platform-team work, not backend work.
**When useful:** when your job runs on it — then learn pods, deployments, services, ingress, configmaps/secrets, probes, and resource limits. That's a week, not six months.
**Instead:** learn Docker and `docker compose` properly. That gives you 80% of the containerization value and makes your integration tests work.

### Microservices
**Why skip:** microservices trade *code complexity* for *operational and distributed-systems complexity*. Before you've felt a monolith's pain, you can't make that trade well — and you'll pay the costs (network failures, distributed transactions, deployment coordination, debugging across services) for benefits you don't need.
**When useful:** when independent deployment or independent scaling of one part is a real, current problem — usually with multiple teams.
**Instead:** build a well-layered **modular monolith** with clean internal boundaries. If those boundaries are right, extracting a service later is mechanical. Do Project 5 to learn the costs deliberately, in a controlled setting.

### Event Sourcing & CQRS
**Why skip:** enormous complexity — event versioning, replay, projections, eventual consistency everywhere — for benefits (perfect audit, temporal queries) that most systems don't need. It is one of the most commonly regretted architectural choices in the industry.
**When useful:** genuine audit/regulatory requirements, or complex domains where "how did we get to this state?" is a first-class business question.
**Instead:** a plain audit log table. It gets you 90% of the benefit for 2% of the cost.

### Domain-Driven Design (strategic / the full book)
**Why skip:** the strategic half (bounded contexts, context maps, ubiquitous language across teams) is organizational design. It's meaningful with several teams and a complex domain; on your own it's vocabulary without referents.
**When useful:** a complex business domain with multiple teams.
**Instead:** learn the *tactical* bits that transfer immediately — entities, value objects, keeping business rules out of controllers and out of the database layer.

### Raft, Paxos, consensus internals
**Why skip:** you will almost certainly never implement consensus. etcd, ZooKeeper, and your database already did.
**When useful:** building infrastructure, or a specific interview at an infra company.
**Instead:** know *what* consensus provides (a consistent leader/lock/membership view), that quorums need W+R>N, and when you'd reach for a system that offers it.

### CRDTs, vector clocks, Lamport timestamps
**Why skip:** deep theory for a narrow class of problems (offline-first collaborative editing, multi-primary conflict resolution).
**When useful:** building a collaborative editor or a multi-region multi-writer store.
**Instead:** understand eventual consistency, last-write-wins, and why conflicts happen at all.

### Service meshes, Istio, Envoy configuration
**Why skip:** infrastructure-team tooling that solves problems (mTLS everywhere, traffic policy across 50 services) you won't have.
**When useful:** large service fleets, and usually someone else's job.
**Instead:** understand what they *do* — timeouts, retries, mTLS, traffic splitting — because you should be able to implement all of those in application code anyway.

---

## Databases

### Database internals (WAL format, page layout, buffer pool tuning)
**Why skip:** fascinating, and almost never actionable. DBA/infra territory.
**When useful:** you're a DBA, or you've exhausted query-level optimization on a genuinely large system.
**Instead:** `EXPLAIN ANALYZE`. Reading query plans has perhaps 50× the practical ROI of understanding storage internals.

### Learning five databases
**Why skip:** shallow knowledge of Cassandra, DynamoDB, Neo4j, ClickHouse, and CockroachDB is worth far less than deep knowledge of Postgres. It also produces the worst interview answer there is: naming technologies instead of reasoning about properties.
**When useful:** when a specific job or a specific access pattern demands one.
**Instead:** master PostgreSQL. Learn Redis for caching. Learn *the decision criteria* for when a different class of store would win — that's the transferable part.

### ORM internals / writing your own ORM
**Why skip:** it's a rabbit hole that teaches you about the ORM, not about databases.
**When useful:** basically never for an application developer.
**Instead:** learn SQL properly and learn to see the SQL your ORM generates. The skill that matters is *"I can tell what queries this code will produce."*

### Sharding
**Why skip:** it's the last resort, and 95% of systems never need it. Learning to shard before learning to index is like learning to fly a plane before learning to drive.
**When useful:** after indexes, caching, replicas, and partitioning are exhausted.
**Instead:** master indexing, query plans, caching, and read replicas. Know sharding *conceptually* for interviews — shard key selection, cross-shard pain, resharding — which takes an hour, not a month.

---

## Code & languages

### Learning a second (and third) language
**Why skip:** the second language teaches you syntax you already conceptually know, while the depth you lack is in databases, networking, and design — which are language-independent.
**When useful:** a job requires it (then it takes 2–3 weeks, because concepts transfer), or you want a genuinely different paradigm after your fundamentals are solid.
**Instead:** go deeper in one language. Concurrency, profiling, memory behavior, its testing ecosystem.

### Memorizing all 23 GoF patterns
**Why skip:** ~10 are used regularly; the rest are historical artifacts of 1990s C++ and Java. Memorizing definitions also actively causes over-engineering.
**When useful:** never, as memorization.
**Instead:** [04](./04-design-patterns.md) — 10 patterns, learned through the *pain* each one resolves.

### Functional programming theory (monads, category theory)
**Why skip:** high-effort, low immediate ROI for backend CRUD-and-scale work.
**When useful:** you work in Haskell/Scala/F#, or after fundamentals are solid and you want new mental models.
**Instead:** adopt the practical parts today — pure functions, immutability by default, avoiding shared mutable state. Those improve your code immediately with no theory.

### Advanced algorithms (DP, graph algorithms, competitive programming)
**Why skip:** near-zero overlap with backend engineering. A year of LeetCode makes you better at LeetCode.
**When useful:** interviewing at companies with heavy algorithmic screens — then do *focused* prep (~150 problems over 2–3 months), not open-ended grinding.
**Instead:** hash maps, arrays, sorting semantics, and Big-O intuition. Then build things.

---

## Tooling & platform

### Terraform / IaC mastery
**Why skip:** platform-engineering depth. Knowing it exists and reading it is enough early.
**When useful:** you own infrastructure.
**Instead:** be able to read Terraform and understand what it provisions.

### Advanced observability stacks (writing exporters, PromQL mastery, custom dashboards)
**Why skip:** tool depth before you have the concepts.
**When useful:** when you own the platform, or when your service genuinely needs custom instrumentation.
**Instead:** understand the four golden signals, percentiles, and correlation IDs, and be able to *use* an existing dashboard to answer a question.

### gRPC, GraphQL, Protocol Buffers
**Why skip:** they solve specific problems (efficient internal RPC; flexible client-driven queries) that REST handles adequately until you hit those problems. GraphQL in particular introduces N+1 and caching challenges that you should understand *in the simple case* first.
**When useful:** your job uses them. Each takes about a week once REST is second nature.
**Instead:** master REST, HTTP semantics, and API design. All of it transfers.

### Chaos engineering programs
**Why skip:** you need reliable systems and good observability *before* deliberately breaking things.
**When useful:** mature systems with real SLOs and on-call rotations.
**Instead:** manually kill a dependency in your own project and watch what happens. Same lesson, zero infrastructure.

### Serverless architectures
**Why skip:** a deployment model, not a body of knowledge. Its constraints (cold starts, execution limits, connection pooling problems with databases) are confusing until you understand the normal model well.
**When useful:** a job uses it, or you have genuinely spiky low-volume workloads.
**Instead:** understand stateless services and horizontal scaling. Serverless is then a one-day concept.

### AI/ML engineering
**Why skip:** an entirely separate discipline. Adding it now means being mediocre at two things.
**When useful:** after you're a strong backend engineer — at which point "backend engineer who can ship ML-backed features" is a genuinely valuable profile.
**Instead:** finish this roadmap. Being strong at data modeling, queues, and API design is *also* exactly what ML systems need.

---

## Habits to avoid, not just topics

- **Tutorial hell.** Watching a 40-hour course feels like progress and produces almost none. Cap consumption at whatever you need to get unstuck, then build.
- **Endless setup and tooling.** Perfect dotfiles, the ideal editor, the perfect project structure — all procrastination in a productive costume.
- **Reading about system design without doing it out loud.** Recognition is not recall. You must speak the designs.
- **Starting projects you never finish.** Five half-built projects teach less than one completed one, and none of them is portfolio material.
- **Collecting tools instead of depth.** "I've used 12 technologies" is a weaker signal than "I can explain exactly why this query got slow and how I fixed it."
- **Optimizing before measuring.** If you can't name the metric that will improve, you're guessing.

---

## The single test for any topic

> **"What problem does this solve, and do I have that problem right now — or will I within six months?"**

If the answer is no, it goes on this list. Revisit it when the problem arrives; you'll learn it in a fraction of the time because you'll have the context that makes it stick.
