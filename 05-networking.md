# 05 — Computer Networking for Backend Engineers

> You don't need the OSI model. You need to know exactly what happens between two processes, what it costs, and how it fails.

---

## Part 1 — The mental model

Three facts drive every design decision in distributed systems:

1. **The network is slow relative to local work.** A memory access is ~100ns. A same-datacenter round trip is ~0.5ms (5,000×). Cross-continent is ~150ms (1,500,000×). This is why N+1 queries and chatty service calls destroy performance.
2. **The network is unreliable.** Packets drop, connections reset, DNS goes stale, and — the worst case — a request succeeds but the *response* is lost, so the caller can't tell whether it happened. **This single fact is why idempotency exists.**
3. **You can't tell "slow" from "dead."** There is no way to distinguish a crashed server from a slow one except by waiting. That's what a timeout is: a guess, that you must make explicitly, because the default is "wait forever."

---

## Part 2 — The layers you actually use 🔥

### IP, ports, sockets
- **IP address** identifies a *machine*; a **port** identifies a *process* on it.
- A **connection** is a 4-tuple: `(src_ip, src_port, dst_ip, dst_port)`. That's why one server on port 443 handles many clients — each has a different source port.
- A **socket** is your program's handle to one endpoint of that connection.
- Ports 0–1023 are privileged. Know 22, 53, 80, 443, 5432 (Postgres), 6379 (Redis), 3306 (MySQL).
- **Ephemeral port exhaustion** is a real production failure: a client opening thousands of short-lived connections runs out of source ports (worsened by `TIME_WAIT`). The fix is connection reuse — keep-alive and pooling.

### TCP 🔥
- **Connection-oriented**, ordered, reliable, with flow control and congestion control.
- **3-way handshake:** SYN → SYN-ACK → ACK. That's one full round trip *before any data moves*. Add TLS and it's 2–3 RTTs. **This is why connection reuse matters so much.**
- **Reliability** via sequence numbers, acknowledgements, and retransmission.
- **Flow control** (receiver window) and **congestion control** (slow start) — TCP deliberately backs off when the network is loaded. This is backpressure at the transport layer.
- **Head-of-line blocking:** one lost packet stalls everything behind it on that connection.

### UDP 🔥
- Fire and forget: no handshake, no ordering, no retransmission, no congestion control. Just datagrams.
- **Use when** latency matters more than completeness, or you want to build your own reliability: DNS, video/voice (a late packet is useless anyway), gaming, metrics/logs shipping (StatsD), and QUIC/HTTP3 (which rebuilds reliability in userspace to escape TCP's head-of-line blocking).

**Interview answer:** *"TCP gives ordered, reliable delivery at the cost of a handshake and head-of-line blocking. UDP gives you raw datagrams with no guarantees but no setup cost. You pick UDP when late data is worthless, or when you want to implement smarter reliability yourself — which is exactly what QUIC does."*

### DNS 🔥
Turning `example.com` into an IP.

**Resolution path:** stub resolver (OS cache) → recursive resolver (ISP / 8.8.8.8) → root → TLD (`.com`) → authoritative nameserver → answer. Caching happens at every layer, governed by **TTL**.

Records worth knowing: **A** (IPv4), **AAAA** (IPv6), **CNAME** (alias), **MX** (mail), **TXT** (verification), **NS** (delegation), **SRV**.

**Production realities:**
- *"I changed DNS and nothing happened"* — TTL. Lower the TTL **before** a planned migration (e.g. to 60s a day ahead), then change.
- DNS-based load balancing and failover are coarse and slow because of caching. Prefer a real load balancer with a stable address.
- Clients cache DNS too — some JVMs cached forever by default, a famous source of outages after failover.
- Internal service discovery in Kubernetes is DNS (`service.namespace.svc.cluster.local`).

### TLS / HTTPS 🔥
**What it guarantees:** confidentiality (encrypted), integrity (untampered), and **identity** (the certificate proves the server is who it claims, via a chain to a trusted CA).

**Handshake, conceptually:** client hello (supported ciphers) → server hello + certificate → client validates the chain and expiry → key exchange (ECDHE, giving forward secrecy) → both derive a symmetric session key → all further traffic uses fast symmetric crypto.

**Costs and mitigations:** TLS 1.2 adds 2 RTTs, TLS 1.3 adds 1 (and 0-RTT for resumption). Session resumption and keep-alive amortize it. This is why you terminate TLS at the load balancer/CDN close to the user and often use plain HTTP (or lighter mTLS) internally.

**Things that bite:** expired certificates (the #1 self-inflicted outage — monitor expiry and automate renewal), missing intermediate certificates (works in browsers, fails in curl/Java), hostname mismatch, and clock skew making valid certs look invalid.

### HTTP 🔥
**Anatomy:** request line (`GET /users/42 HTTP/1.1`), headers, blank line, body. Response: status line, headers, body.

**Methods and their properties** — this table is interview bait and real design guidance:

| Method | Safe (no change) | Idempotent (repeat = same) |
|---|---|---|
| GET, HEAD | ✅ | ✅ |
| PUT, DELETE | ❌ | ✅ |
| POST | ❌ | ❌ ← why POST needs idempotency keys |
| PATCH | ❌ | depends on your semantics |

**Status codes that matter:**
- `200` OK · `201` Created (+ `Location`) · `202` Accepted (async, work queued) · `204` No Content
- `301` permanent / `302` temporary redirect · `304` Not Modified (conditional GET)
- `400` malformed · `401` not authenticated · `403` authenticated but not allowed · `404` not found · `409` conflict · `422` semantically invalid · `429` rate limited (send `Retry-After`)
- `500` our bug · `502` bad gateway · `503` unavailable/overloaded · `504` gateway timeout

**Headers that matter:** `Content-Type`, `Authorization`, `Cache-Control`, `ETag`/`If-None-Match`, `Accept-Encoding`, `X-Request-Id`/`traceparent`, `Retry-After`, `X-Forwarded-For` (the real client IP behind a proxy).

### HTTP versions 🟡 (conceptual only)
- **HTTP/1.1** — one request at a time per connection; keep-alive reuses the connection; browsers open ~6 parallel connections to work around it. Head-of-line blocking at the HTTP layer.
- **HTTP/2** — multiplexes many streams over one TCP connection, plus header compression. Fixes HTTP-layer HOL blocking, but a TCP packet loss still stalls all streams (TCP-layer HOL blocking remains). Also the transport for gRPC.
- **HTTP/3** — HTTP over QUIC over UDP. Streams are independent, so packet loss affects only one. Faster connection setup (crypto + transport handshake combined). Big win on lossy/mobile networks.

**Interview-sufficient summary:** *"1.1 = one-at-a-time per connection; 2 = multiplexed over one TCP connection but still subject to TCP head-of-line blocking; 3 = multiplexed over UDP/QUIC, so loss on one stream doesn't stall the others."*

### REST 🔥
Resources as nouns, HTTP verbs as actions, statelessness, and meaningful status codes.
```
GET    /orders?status=paid&limit=20&cursor=abc
POST   /orders
GET    /orders/42
PATCH  /orders/42
DELETE /orders/42
POST   /orders/42/refunds     ← an action that is genuinely a resource
```
Stateless means the server keeps no per-client session in memory — **which is exactly what makes horizontal scaling possible.** That connection between REST and scalability is worth stating explicitly in interviews.

### WebSockets 🟡
Starts as HTTP, sends `Upgrade: websocket`, then becomes a persistent bidirectional connection.

**Use when** the server must push in real time: chat, live dashboards, collaborative editing, trading.
**Don't use when** polling every 30 seconds would do, or when updates are one-directional (**Server-Sent Events** are simpler and reconnect automatically).
**Costs:** connections are *stateful*, which breaks the stateless model — you now need sticky routing or a shared pub-sub layer (Redis) so any server can reach any user's connection. Also: load balancer idle timeouts kill idle sockets, so you need heartbeats.

### Load balancers, reverse proxies, proxies 🔥
- **Reverse proxy** — sits in front of *your servers*. Terminates TLS, routes by path/host, caches, compresses, rate limits, hides your topology. (nginx, Envoy, HAProxy.)
- **Load balancer** — a reverse proxy whose main job is distributing traffic across instances with health checks. **L4** (TCP, fast, opaque) vs **L7** (HTTP-aware: route by path, header, cookie).
- **Forward proxy** — sits in front of *clients*, for egress control, filtering, or corporate policy.
- **Algorithms:** round robin, least connections (better with uneven request cost), consistent hashing (for cache affinity), weighted (for canaries).
- **Health checks** eject bad instances — but *only if the check is meaningful*. A check that returns 200 while the database is down will happily route traffic into a broken server.

### NAT & firewalls 🟡
- **NAT** maps many private IPs to one public IP; it's why inbound connections to a machine behind NAT need port forwarding, and why you see private ranges (10.x, 172.16–31.x, 192.168.x) internally.
- **Firewalls / security groups** allow or deny by IP, port, and direction. **The key production symptom:** a *dropped* packet gives you a **connection timeout**; an actively *rejected* one gives **connection refused**. That distinction alone will save you hours.
- Baseline architecture: public subnet holds the load balancer; private subnets hold app servers and databases; the database accepts connections only from the app security group. Never expose a database to the internet.

### CDN 🟡
Geographically distributed caches near users. Serves static assets, and increasingly cached API responses.
**Why:** cuts latency (distance is physics), offloads origin traffic, absorbs traffic spikes and some DDoS.
**Key mechanics:** cache keys, `Cache-Control`/`max-age`, `ETag` validation, purge/invalidation, and **cache-busting via content-hashed filenames** (`app.a3f9c1.js`) — which is strictly better than purging.

---

## Part 3 — What happens when you type `https://example.com` 🔥🔥

Be able to narrate this for five minutes. It's the single most common networking interview question, and it's genuinely the map of the whole domain.

1. **URL parsing** — scheme `https`, host `example.com`, implicit port 443, path `/`. The browser checks HSTS (forcing HTTPS) and its own caches.
2. **DNS resolution** — browser cache → OS cache → hosts file → recursive resolver → (root → TLD → authoritative) → an A/AAAA record, cached per TTL. Often the answer is a CDN or load balancer address, possibly geo-targeted (anycast).
3. **TCP connection** — SYN → SYN-ACK → ACK to port 443. One RTT. (With HTTP/3, QUIC over UDP instead.)
4. **TLS handshake** — hello/certificate/validation/key exchange; a symmetric session key is derived. 1–2 RTTs, or ~0 on resumption. The browser verifies the cert chain, hostname, and expiry.
5. **HTTP request sent** — `GET / HTTP/2`, with `Host`, `User-Agent`, `Accept`, cookies.
6. **Edge / CDN** — if the object is cached at the edge, you get a response here and steps 7–11 never happen. This is why CDNs are such leverage.
7. **Load balancer** — terminates TLS, picks a healthy backend (round robin / least connections), adds `X-Forwarded-For`, opens or reuses a pooled connection to the app.
8. **Application server** — middleware chain (request ID → logging → auth → rate limit) → route matched → handler → service layer.
9. **Data layer** — check cache (Redis) first; on a miss, borrow a connection from the pool, query Postgres (index lookup, possibly a read replica), populate the cache. Maybe enqueue async work to a queue.
10. **Response travels back** — app → LB → (CDN may store it) → browser. Compressed (gzip/brotli), with caching headers.
11. **Browser renders** — parses HTML, discovers subresources, and repeats the whole process for each (mostly reusing connections), executes JS, which fires more API calls.

**Then the follow-up you should volunteer:** *"and at every one of those steps there's a distinct failure mode"* — which leads into the next section and demonstrates real operational experience.

---

## Part 4 — Production failure modes 🔥

This is the highest-value practical section in this file. Learn to map a symptom to a layer.

| Symptom | What it means | Where to look |
|---|---|---|
| **Connection refused** | Something answered and said *no*: nothing is listening on that port, or a firewall actively rejected it | Is the process running? Right port? Bound to `0.0.0.0` not `127.0.0.1`? Security group rules? |
| **Connection timeout** | Packets went into a void — no response at all | Firewall/security group dropping, wrong IP, host down, routing/VPC misconfiguration, network partition |
| **DNS failure** (`NXDOMAIN` / resolution failed) | The name couldn't be resolved | Typo, expired domain, missing record, resolver down, wrong search domain inside a container/K8s |
| **TLS errors** | Certificate chain, hostname, or expiry problem | `openssl s_client -connect host:443`; check expiry, intermediates, SNI, clock skew |
| **502 Bad Gateway** | The proxy reached the backend but got garbage or a dropped connection | **Your app crashed / returned an invalid response / died mid-request.** Check app logs and OOM kills first. |
| **503 Service Unavailable** | The proxy has **no healthy backend**, or you're being deliberately shed | All instances failed health checks, deploy gone wrong, autoscaling not caught up, circuit breaker open |
| **504 Gateway Timeout** | The backend was alive but **too slow** — the proxy's timeout fired | Slow query, exhausted connection pool, a slow downstream dependency, lock contention, GC pause |
| **Slow requests / high p99 with a fine p50** | A subset of requests hits something pathological | Cold cache, one slow shard/replica, a lock, GC, an N+1 on rare-but-large records, a noisy neighbor |
| **Intermittent 5xx after a deploy** | Old connections routed to dying pods | Missing graceful shutdown / readiness probes / connection draining |
| **Everything slow, CPU low** | You're *waiting*, not computing | Connection pool exhaustion, thread pool starvation, a downstream dependency with no timeout |

**The 502 vs 503 vs 504 distinction in one line each:**
- **502** — the backend gave a *bad answer* (it's broken).
- **503** — there *is no* backend available (nothing healthy, or overload).
- **504** — the backend never answered *in time* (it's slow).

### The three settings that prevent most cascading failures
1. **Timeouts on every outbound call.** No timeout means one slow dependency exhausts all your threads/connections and takes your service down with it. This is the #1 cause of cascading failure.
2. **Retries with exponential backoff + jitter — only on idempotent operations.** Naive immediate retries turn a blip into a self-inflicted DDoS (the "retry storm"). Jitter prevents synchronized retries. Cap total attempts, and budget retries (e.g. max 10% of traffic).
3. **Connection pooling / keep-alive.** Avoids paying the TCP+TLS handshake per request, and bounds concurrency into your dependencies.

Add **circuit breakers** on top (see [09](./09-production-engineering.md)): after N consecutive failures, stop calling and fail fast, so you don't queue work for a dependency that's already down.

### The debugging toolkit
```bash
dig +trace example.com            # full DNS resolution path
curl -v https://example.com       # DNS, TCP, TLS, headers, all visible
curl -w "@curl-format.txt" -o /dev/null -s URL   # per-phase timing: dns, connect, tls, ttfb
openssl s_client -connect host:443 -servername host   # certificate chain
ss -tnp / netstat -an             # what's listening, connection states, TIME_WAIT pileups
tcpdump -i any port 5432          # what's actually on the wire
traceroute / mtr host             # where the path breaks
nc -zv host 5432                  # is the port reachable at all?
```
Learning to read `curl -w` phase timings — DNS vs connect vs TLS vs time-to-first-byte — lets you localize a slow request to a layer in one command. Do this until it's reflex.
