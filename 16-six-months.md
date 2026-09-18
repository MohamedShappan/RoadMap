# 16 — If You Only Had 6 Months

> This is the version I'd follow myself. **2 hours/day, 6 days/week.** Nothing optional. Nothing aspirational. Every week has a deliverable.
>
> If you read only one file in this repository, read this one — then go do week 1.

**Non-negotiable setup, decided once on day one and never revisited:** one typed backend language (Go, Java, C#, or TypeScript — Python with type hints is fine), **PostgreSQL**, **Redis**, **Docker**. No framework shopping. No second language. The tool choice is not the work.

---

## Month 1 — Databases (weeks 1–4)

*Because this is where the leverage is, and everything downstream depends on it.*

| Week | Focus | Deliverable |
|---|---|---|
| **1** | Modeling: tables, PKs, FKs, 1:N and N:M, constraints, normalization to 3NF | An e-commerce schema with full DDL, every constraint justified |
| **2** | SQL fluency: JOINs, GROUP BY, aggregates, subqueries, `EXISTS` | 20 queries of increasing difficulty against 1M seeded rows |
| **3** | Transactions: ACID, isolation levels, locks, deadlocks, connection pooling | Reproduce a lost update, a non-repeatable read, and a real deadlock in two psql sessions |
| **4** | Indexes: B-trees, composite indexes, `EXPLAIN ANALYZE`, selectivity | Take one query from 5s to <50ms and write down every step of the reasoning |

**Read:** [03 — Databases](./03-databases.md), Parts 1–3.
**Month-end checkpoint:** [14, Checkpoint 1](./14-mastery-checkpoints.md). You must be able to answer 1.1 cold.

---

## Month 2 — Networking + Your First Real API (weeks 5–8)

| Week | Focus | Deliverable |
|---|---|---|
| **5** | Networking: the full request lifecycle, TCP/UDP, DNS, TLS, HTTP, LB/proxy, failure modes | Narrate `https://example.com` out loud for 5 minutes, recorded. Annotate a full `curl -v`. |
| **6** | API design: REST, status codes, validation, error contracts, pagination | Project 1 skeleton: routing, layering, migrations, config |
| **7** | Auth (sessions vs JWT, hashing), authorization, IDOR, layering, DI | Project 1: auth + authorization + per-resource ownership checks |
| **8** | Testing (unit with fakes, integration with real Postgres), structured logging, Docker Compose | **Project 1 complete**: `docker compose up` works, tests pass, README is sufficient |

**Read:** [05 — Networking](./05-networking.md), [08 — Backend Engineering](./08-backend-engineering.md).
**Month-end checkpoint:** [14, Checkpoints 2 and 3](./14-mastery-checkpoints.md).

---

## Month 3 — Patterns + Performance (weeks 9–12)

| Week | Focus | Deliverable |
|---|---|---|
| **9** | DI, Strategy, Repository, Factory, Adapter — via the pain each solves | Refactor Project 1: swappable storage + pluggable auth, **with zero test changes** |
| **10** | Decorator/middleware, Observer, the outbox pattern, recognition drills | A caching decorator on your repository + an event bus for side effects |
| **11** | Query plans in anger: seq scans, unusable indexes, N+1, keyset pagination | Seed 10M rows; find and fix three real bottlenecks; record before/after numbers |
| **12** | Caching (cache-aside, TTL, invalidation, stampede), read replicas, replication lag, pool sizing | A written **performance report** with p50/p95/p99 before and after |

**Read:** [04 — Design Patterns](./04-design-patterns.md), [03 — Databases](./03-databases.md) Parts 4–5.
**Month-end checkpoint:** [14, Checkpoints 4 and 5](./14-mastery-checkpoints.md). **Start one timed, spoken system design per week from here on — every week, without exception.**

---

## Month 4 — Async + Production Engineering (weeks 13–16)

| Week | Focus | Deliverable |
|---|---|---|
| **13** | Queues, background jobs, idempotency, retries + backoff + jitter, DLQs | **Project 3 (notification system)** — the purest reliability project |
| **14** | Observability: structured logs, the four golden signals, percentiles, health checks | `/metrics` with a latency histogram; graph p99; liveness vs readiness done correctly |
| **15** | Resilience: timeouts everywhere, circuit breakers, bulkheads, graceful shutdown, degradation | Kill a dependency under load and prove you fail fast and recover |
| **16** | Docker, CI pipeline, backward-compatible migrations, security baseline (OWASP, IDOR, injection) | CI: lint → unit → integration → image. Attack your own API and fix what you find. |

**Read:** [09 — Production Engineering](./09-production-engineering.md).
**Month-end checkpoint:** [14, Checkpoint 6](./14-mastery-checkpoints.md). Write a `RUNBOOK.md` for your own project.

---

## Month 5 — System Design (weeks 17–20)

*Four weeks, ten systems, twice each. This month is mostly spoken practice, not reading.*

| Week | Focus | Deliverable |
|---|---|---|
| **17** | Building blocks + the scaling toolkit + **estimation drills** | 5 back-of-envelope estimates until it's reflexive; the framework memorized |
| **18** | Designs 1–3: URL shortener, chat, notification system | 3 timed 45-min designs, spoken, then compared against [07](./07-system-design-framework.md) |
| **19** | Designs 4–7: e-commerce, file storage, social feed, payments | 4 timed designs. Force yourself to name the bottleneck and 3 trade-offs each. |
| **20** | Designs 8–10: food delivery, video streaming, search + **redo your three weakest** | 6 designs. The second pass is where fluency forms. |

**Read:** [06 — System Design](./06-system-design.md), [07 — Framework + 10 Systems](./07-system-design-framework.md).
**Month-end checkpoint:** [14, Checkpoint 7](./14-mastery-checkpoints.md).

---

## Month 6 — The Capstone + Interview Readiness (weeks 21–24)

| Week | Focus | Deliverable |
|---|---|---|
| **21** | **Project 5**: distributed order processing — saga, outbox, idempotent consumers | 3–4 services, a database each, communicating via a queue |
| **22** | Project 5 continued: failure injection and distributed tracing | Kill any service mid-saga; the system converges; **show it in a trace** |
| **23** | Interview drilling: every question in [11](./11-interview-prep.md), spoken, with trade-offs | Record yourself on one area per day; fix what sounds weak |
| **24** | Project 6 (a full written design doc) + consolidate the cheat sheet + polish your portfolio | A design document you could present and defend for 45 minutes |

**Read:** [11 — Interview Prep](./11-interview-prep.md), [15 — Cheat Sheet](./15-cheatsheet.md).
**Final checkpoint:** the self-assessment at the end of [14](./14-mastery-checkpoints.md). All ten boxes, without notes.

---

## What you deliberately do NOT do in these six months

Read [12](./12-do-not-learn-yet.md) in week 1, then treat this as a hard rule:

- ❌ No second programming language
- ❌ No Kubernetes beyond understanding what a pod and a deployment are
- ❌ No microservices until week 21 (and then only to learn what they cost)
- ❌ No GraphQL, gRPC, Kafka-as-a-study-topic, Terraform, or serverless
- ❌ No event sourcing, CQRS, or DDD books
- ❌ No LeetCode grinding (unless you have a specific algorithmic interview scheduled — then timebox it separately)
- ❌ No learning five databases. Postgres and Redis. That's it.
- ❌ No new project until the current one is *finished*

Every one of those is a legitimate topic. Every one of them is also how six months becomes eighteen.

---

## The weekly rhythm, concretely

```
Mon  45 min theory  →  75 min build
Tue  45 min exercise → 75 min build
Wed  45 min theory  →  75 min build
Thu  45 min exercise → 75 min build
Fri  45 min build   →  75 min build
Sat  45 min active review (explain the week aloud, no notes)
     75 min interview questions aloud + (from week 9) one timed system design
Sun  rest
```

---

## Three rules that decide whether this works

1. **Ship the weekly deliverable.** A week without a deliverable is a week of reading, and reading decays. The deliverable is the proof and the memory.
2. **Say it out loud.** Every Saturday, explain that week's concepts with no notes, to a wall, a rubber duck, or a camera. The gap between "I understood that" and "I can explain that" is enormous, and interviews only measure the second one.
3. **Measure everything you optimize.** Before and after, with numbers. This builds the single most valuable professional habit there is — and it gives you concrete stories for every interview you'll ever have.

---

## Where you'll be after six months

You will be able to design and build a production backend, explain exactly why a query is slow, make an API safe to retry, keep a service alive when its dependencies aren't, reason about scaling from first principles, and defend an architecture out loud for 45 minutes.

That is not "junior who studied a lot." That is a strong practical engineer — and from there, growth comes from real production experience, which is the only thing that can't be learned from a roadmap.

**Start with week 1, today. Not with the tooling. Not with the language debate. With the schema.**
