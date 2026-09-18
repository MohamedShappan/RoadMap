# 09 — Production Engineering

> Code that runs on your laptop is a hobby. Code you can **observe, operate, and safely change** is a profession. This is also the section that makes your system design answers credible.

---

## Part 1 — Observability 🔥

The question observability answers: **"something is wrong — what, where, and since when?"**

### The three signals, and what each is for

| Signal | Answers | Cost | Use it to |
|---|---|---|---|
| **Metrics** | *Is something wrong?* | Cheap, aggregated | Alert, dashboard, trend |
| **Logs** | *What exactly happened to this request?* | Expensive at volume | Debug a specific case |
| **Traces** | *Where did the time go across services?* | Moderate (sampled) | Find the slow hop |

The workflow is always: **metric alerts → trace localizes → logs explain.** Build all three and you can debug production without a debugger.

### Logging 🔥
- **Structured (JSON), one event per line.** Grep is not a strategy at scale; you need queryable fields.
- **A correlation/request ID on every line**, propagated to every downstream call (`X-Request-Id` / `traceparent`). Without this, logs from a 10-service request are unreadable noise.
- **Log at boundaries:** request received/completed (with status and duration), external call made, job started/finished, state transition. Not inside loops.
- **Levels with meaning:** `ERROR` = a human must look at this; `WARN` = degraded but handled; `INFO` = business events; `DEBUG` = off in production.
- **Never log** passwords, tokens, cookies, card numbers, or PII. Build a redaction layer, don't rely on discipline.
- **Include context, not prose.** `{"event":"payment_failed","order_id":42,"provider":"stripe","code":"card_declined","duration_ms":812}` beats `"Payment failed!"` by an enormous margin.

### Metrics 🔥
**The four golden signals** (Google SRE): **Latency, Traffic, Errors, Saturation.** If you instrument only these four per service, you can run a system.

- **Types:** counter (monotonic — requests, errors), gauge (point-in-time — queue depth, pool in-use), histogram (distribution — latency).
- **Percentiles, not averages.** If p50 is 50ms and p99 is 4s, 1 in 100 users is having an awful time and the *average* (maybe 90ms) hides it completely. **Averages lie; percentiles don't.** Alert on p99.
- **Never average percentiles across instances** — it's mathematically meaningless. Aggregate histograms.
- **The metrics worth having from day one:** request rate, error rate, latency histogram (all by endpoint), DB pool in-use/wait time, query duration, cache hit rate, queue depth and oldest-message age, external dependency latency and error rate.
- **Cardinality discipline:** never put user IDs, request IDs, or URLs-with-IDs in metric labels. It will destroy your metrics backend. High-cardinality data belongs in logs and traces.

### Tracing 🟡
A trace is a tree of spans following one request across services. It answers "the request took 2s — which hop?" instantly. Use OpenTelemetry, propagate context, and sample (100% of errors, a small percentage of successes). With more than three services, this stops being optional.

### Health checks 🔥
- **Liveness** — "is this process wedged?" Should check almost nothing. If it fails, the orchestrator restarts you.
- **Readiness** — "can I serve traffic right now?" Checks critical dependencies. If it fails, you're pulled from the load balancer but not killed.

**The classic outage:** making the liveness probe check the database. The database has a blip → every instance fails liveness → every instance restarts simultaneously → thundering herd on a recovering database → full outage from a minor blip. **Liveness must not depend on external services.** This is a great interview answer.

### Alerting 🟡
**Alert on symptoms users feel** (error rate, p99 latency, queue age, SLO burn rate), not on causes (CPU at 80% is not an incident). Every alert must be actionable and have a runbook. Alerts that fire routinely and get ignored are worse than no alerts — they train your team to ignore the real one.

**SLO thinking:** "99.9% of requests succeed within 300ms over 30 days" gives you an **error budget**. Budget remaining → ship features. Budget burned → stop and fix reliability. It converts an unwinnable argument into arithmetic.

---

## Part 2 — Resilience 🔥

### Timeouts
Every network call gets one. No exceptions. Set connect and read timeouts separately, and make each one **shorter than the caller's timeout** (otherwise the caller gives up while you keep working — wasted capacity). **A missing timeout is the #1 cause of cascading failure**: one slow dependency consumes every thread in your service, and you go down because someone *else* got slow.

### Retries
Exponential backoff **with jitter**, bounded attempts, **idempotent operations only**.
- Retry: timeouts, 503/429 (respect `Retry-After`), connection errors.
- Don't retry: 400, 401, 403, 404, 422 — the answer won't change.
- **Retry storms:** when a dependency degrades, every client retries at once and finishes the job of killing it. Jitter + a retry budget (cap retries at ~10% of traffic) prevents this.

### Circuit breakers 🟡
**Closed** (normal) → failure rate exceeds a threshold → **Open** (fail immediately, don't even try) → after a cooldown → **Half-open** (allow one probe) → success → Closed.
The point: stop wasting your own resources on a dependency that is already down, and give it room to recover. Pair with a fallback: cached value, default value, or a clear degraded response.

### Bulkheads
Separate resource pools per dependency, so the slow one can't drown the others. Named after ship compartments — flooding one doesn't sink the vessel.

### Idempotency 🔥
Covered in [06](./06-system-design.md) and [08](./08-backend-engineering.md). In production terms: **every consumer of a queue, every webhook handler, and every retryable endpoint must be idempotent**, because at-least-once delivery is the only guarantee you'll ever actually get.

### Graceful shutdown 🔥
On SIGTERM: (1) fail the readiness probe so the LB stops sending new traffic; (2) wait a few seconds for in-flight routing to drain; (3) stop accepting new requests; (4) finish in-flight ones with a deadline; (5) stop consuming from queues and let current jobs finish or return them; (6) close DB pools; (7) exit.
Without this, every single deploy drops requests. Most teams discover this only after chasing phantom 502s for a week.

### Graceful degradation
Decide *in advance* what you'd turn off under stress: recommendations, related items, view counts, non-critical enrichment. A checkout page that loads without the "customers also bought" widget is vastly better than one that doesn't load.

---

## Part 3 — Delivery 🔥

### Docker
- **Image** = the filesystem + metadata; **container** = a running instance. Images are layered and cached — order your Dockerfile so dependencies install before code is copied, or every code change reinstalls everything.
- **Multi-stage builds** — build in a fat image, copy only the artifact into a slim runtime image. Smaller images deploy faster and have less attack surface.
- Run as a **non-root** user. Use a `.dockerignore`. Pin base image versions. Never bake secrets into images (they're in the layer history forever).
- **`docker compose`** for local dev: app + Postgres + Redis with one command. This is the highest-ROI Docker skill — it makes integration testing trivial and onboarding instant.

### CI/CD 🔥
A minimum viable pipeline, and it genuinely is enough:
```
on push:  lint → unit tests → integration tests (with service containers) → build image → push to registry
on main:  deploy to staging → smoke tests → deploy to production
```
- **Every commit runs the tests.** Non-negotiable.
- **Build once, promote the same artifact** through environments. Rebuilding per environment means you didn't test what you shipped.
- **Migrations run before/separately from the deploy**, and must be **backward-compatible** so old and new code can run simultaneously during a rollout. The expand-contract pattern: add a nullable column → deploy code that writes both → backfill → deploy code that reads the new one → drop the old. **Never** rename or drop a column in the same deploy as the code change.
- **Fast rollback** matters more than perfect deploys. Know your rollback command before you need it.

### Deployment strategies 🟡
- **Rolling** — replace instances gradually. The default.
- **Blue/green** — two full environments, flip traffic. Instant rollback, double the infrastructure.
- **Canary** — 5% of traffic to the new version, watch metrics, ramp up. The safest for risky changes.
- **Feature flags** — decouple *deploy* from *release*. Ship dark, enable for 1%, ramp, and kill instantly if metrics move. Rules: flags are temporary (set an expiry and actually remove them), keep the number small, and test both branches. A permanent flag is technical debt with a config UI.

### Kubernetes 🟡 — the useful 20% only
Learn only these, and only when your job needs them: **Pod** (one or more containers), **Deployment** (declares N replicas, handles rolling updates), **Service** (stable internal address + load balancing), **Ingress** (external HTTP routing), **ConfigMap/Secret** (config injection), **liveness/readiness probes**, **resource requests and limits** (get these wrong and you get OOMKills or throttling), and **HPA** (autoscaling on a metric).
Skip for now: Helm authoring, operators, CRDs, service meshes, cluster administration. See [12](./12-do-not-learn-yet.md).

---

## Part 4 — Security fundamentals 🔥

The 20% that prevents most real incidents:

1. **Injection** — SQL, command, template. **Always use parameterized queries.** String-concatenated SQL is the single most exploited bug in web history. Never pass user input to a shell.
2. **Broken authentication** — weak hashing, no rate limit on login (brute force), session fixation, tokens in URLs, no MFA on admin.
3. **Broken access control / IDOR** — the most common real-world flaw. Check ownership on every resource access, server-side. `GET /invoices/1234` must verify *this user* owns invoice 1234.
4. **XSS** — escape output by context; use a Content-Security-Policy; `HttpOnly` cookies so stolen JS can't read tokens.
5. **CSRF** — `SameSite` cookies + CSRF tokens for cookie-authenticated state-changing requests. (Token-in-header APIs are largely immune.)
6. **Sensitive data exposure** — TLS everywhere including internal hops, encrypt at rest, **hash passwords (one-way) vs encrypt data (two-way)** — know the difference and never confuse them.
7. **Security misconfiguration** — debug mode in production, default credentials, public S3 buckets, permissive CORS (`Access-Control-Allow-Origin: *` with credentials), verbose error pages.
8. **Vulnerable dependencies** — automated scanning (Dependabot/Snyk) and an actual patching cadence.
9. **SSRF** — user-supplied URLs fetched by your server can reach internal services and cloud metadata endpoints. Allow-list destinations.
10. **Least privilege** — the app's database user doesn't need `DROP TABLE`; the service's IAM role doesn't need `s3:*`.
11. **Secrets management** — a secret manager, rotation, never in git, never in logs, never in images. If a secret is ever committed, it is compromised — rotate it, don't just delete the commit.
12. **Rate limit everything user-triggerable**, especially login, password reset, and anything that sends an email or SMS (each of those costs you money).

**Mindset:** you are not trying to be unhackable. You are trying to (a) not have the obvious holes, (b) limit the blast radius when something goes wrong, and (c) be able to detect it. Defense in depth — validation *and* parameterized queries *and* least privilege *and* audit logs.

---

## The production readiness checklist

Before anything you build goes live:

- [ ] Structured logs with request IDs
- [ ] Metrics: rate, errors, latency histogram, saturation
- [ ] Liveness and readiness probes (liveness checks nothing external)
- [ ] Graceful shutdown
- [ ] Timeouts on every outbound call
- [ ] Retries with backoff + jitter, on idempotent operations only
- [ ] Idempotency on unsafe endpoints and all queue consumers
- [ ] Config from environment, validated at startup; secrets from a manager
- [ ] Parameterized queries everywhere; authorization checked per resource
- [ ] Database migrations are backward-compatible
- [ ] Automated tests run in CI on every push
- [ ] A documented rollback procedure
- [ ] A runbook: what to check when latency spikes, errors rise, or the queue backs up
- [ ] You know what you'd shed first under load
