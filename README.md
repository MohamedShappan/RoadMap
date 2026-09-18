# The 80/20 Backend Engineer Roadmap

> The smallest set of knowledge that makes you a strong, employable, production-capable backend engineer — and eventually strong at system design interviews.

This is not a syllabus. It is a **prioritized, dependency-ordered** path. Everything here was chosen against one filter:

> *Does this concept show up repeatedly in real production systems, in debugging, or in design interviews — and is it transferable across languages and frameworks?*

If the answer was no, it was demoted or cut. Explicitly cut topics live in [12 — Do Not Learn Yet](./12-do-not-learn-yet.md), and that file is as important as any other.

---

## One-page overview

### The core thesis

Backend engineering is **five recurring problems** wearing different costumes:

| The real problem | Where it shows up |
|---|---|
| **Where does the data live, and how do I get it fast?** | Databases, indexes, caching, sharding |
| **How do two machines talk reliably over an unreliable wire?** | TCP, HTTP, timeouts, retries, idempotency |
| **How do I keep code changeable as it grows?** | Design patterns, DI, layering, testing |
| **How do I survive 100× traffic and partial failure?** | System design, queues, replication, circuit breakers |
| **How do I know what's happening at 3 a.m.?** | Logging, metrics, tracing, health checks |

Every topic below serves one of those five. If you can't map a topic to one of them, it's probably in the "skip for now" pile.

### The eight stages (dependency-ordered)

```
STAGE 0  Programming + data structures you actually use        (2 weeks)
STAGE 1  Databases & SQL — the real bottleneck of most systems (4 weeks)  ← highest ROI
STAGE 2  Networking for backend devs — the request lifecycle   (2 weeks)
STAGE 3  Backend fundamentals — APIs, auth, errors, testing    (4 weeks)
STAGE 4  Code design — patterns, DI, maintainability           (3 weeks)
STAGE 5  Database performance & scaling — "why is it slow?"    (3 weeks)
STAGE 6  Production engineering — observability & resilience   (3 weeks)
STAGE 7  System design — building blocks → framework → drills  (6 weeks)
```

**Why this order and not the obvious one:**

- **Databases before networking, and long before system design.** Most production incidents and most "the app is slow" problems are database problems. You cannot reason about caching, replication, sharding, or CAP without first understanding indexes, transactions, and query plans. Databases are the single highest-ROI area in this entire roadmap.
- **Networking before backend architecture.** "What is a 504?" and "why did this request hang for 30 seconds?" are networking questions. Load balancers and reverse proxies are the vocabulary system design is written in.
- **Design patterns *after* you've written a real API.** Patterns learned before you've felt the pain they solve become cargo cult. You need to have written a 2000-line service that's hard to change before Strategy and DI mean anything.
- **System design last.** System design is not a separate subject — it is the *composition* of stages 1–6. People who study it first memorize diagrams; people who study it last can defend every box on the whiteboard.

### The 20% that carries 80% of the value

If you learned only these, you'd already be above average:

1. **Indexes + `EXPLAIN`** — why a query is slow and how to fix it
2. **Transactions, ACID, isolation levels** — and what a deadlock actually is
3. **Data modeling** — normalize, then denormalize deliberately
4. **N+1 queries** — the most common real-world performance bug in existence
5. **The full request lifecycle** — DNS → TCP → TLS → HTTP → LB → app → DB → response
6. **TCP vs UDP, timeouts, retries, connection pooling**
7. **REST API design + status codes + error contracts**
8. **AuthN vs AuthZ; sessions vs JWT and when each is wrong**
9. **Caching: where, what TTL, and how it goes stale**
10. **Queues and async processing** — the answer to half of all scaling questions
11. **Idempotency** — the answer to half of all reliability questions
12. **Horizontal scaling + stateless services**
13. **Replication, read replicas, eventual consistency**
14. **Dependency Injection + Strategy + Repository** — the three patterns you'll use weekly
15. **Logs, metrics, traces** — and how to debug with them
16. **Timeouts, retries with backoff + jitter, circuit breakers**
17. **Testing: unit vs integration, and what to test at each level**
18. **A repeatable system design framework** you can run on any problem

### How to use this repo

| File | What's in it |
|---|---|
| [01 — Knowledge Map](./01-knowledge-map.md) | Every topic, classified 🔥 MUST / 🟡 SHOULD / ⚪ NICE |
| [02 — The Roadmap](./02-roadmap-stages.md) | The 8 stages, in order, with exercises, projects, checkpoints |
| [03 — Databases](./03-databases.md) | Deep dive: SQL, transactions, indexes, performance, scaling, NoSQL |
| [04 — Design Patterns](./04-design-patterns.md) | The 10 patterns that matter + how to *recognize* them |
| [05 — Networking](./05-networking.md) | The request lifecycle + production failure modes |
| [06 — System Design](./06-system-design.md) | Building blocks → intermediate → advanced |
| [07 — Design Framework + Systems](./07-system-design-framework.md) | The repeatable framework applied to 10 real systems |
| [08 — Backend Engineering](./08-backend-engineering.md) | APIs, auth, validation, jobs, concurrency, testing |
| [09 — Production Engineering](./09-production-engineering.md) | Observability, resilience, CI/CD, Docker, security |
| [10 — Projects](./10-projects.md) | 6 projects, each forcing new concepts |
| [11 — Interview Prep](./11-interview-prep.md) | Must-answer questions with model answers |
| [12 — Do Not Learn Yet](./12-do-not-learn-yet.md) | **Read this early.** What to skip and why |
| [13 — Study Schedule](./13-study-schedule.md) | 1h / 2h / 3h per day plans |
| [14 — Mastery Checkpoints](./14-mastery-checkpoints.md) | Scenario-based tests, not definitions |
| [15 — Cheat Sheet](./15-cheatsheet.md) | The final condensed knowledge set |
| [16 — If You Only Had 6 Months](./16-six-months.md) | **The focused version. Start here if you're impatient.** |

### Ground rules

1. **Build > read.** Roughly 50% of your time should be writing code. Reading about indexes teaches you nothing; adding an index to a 5M-row table and watching `EXPLAIN` change teaches you permanently.
2. **One language, one database, one cache.** Pick a typed backend language (Go, Java, C#, TypeScript, or Python with types) + PostgreSQL + Redis. Don't diversify until stage 7. Transferable knowledge comes from depth, not breadth.
3. **Always ask "why does this exist?" before "how does it work?"** Every concept here is a response to a specific pain. If you don't know the pain, you'll misuse the solution.
4. **Finish the projects.** A stage isn't complete when you've read it; it's complete when you pass its checkpoint in [14](./14-mastery-checkpoints.md).
5. **Avoid tutorial hell.** Watch/read at most enough to start, then build and get stuck. Getting stuck *is* the learning.
