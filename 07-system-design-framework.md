# 07 — The System Design Framework + 10 Worked Systems

---

# Part 1 — A repeatable framework

Your framework is improved from the standard one in three ways: **requirements and estimation are merged into a single early "shape the problem" phase**, there's an explicit **"name the hardest part"** step (this is what interviewers are actually listening for), and **failure/observability are not an afterthought bolted on at the end** — they're part of designing each component.

Budget for a 45-minute interview in brackets.

### Phase A — Shape the problem *(8 min)*

**1. Clarify and scope.** Ask 3–5 questions, then *state your assumptions and move*. Never spend 15 minutes asking questions.
> "Is this read-heavy? Do we need real-time or is a few seconds of delay fine? Mobile and web? One region or global? Is this greenfield or are we scaling something that exists?"

**2. Functional requirements.** Pick the **3–4 core features** and explicitly defer the rest.
> "I'll design post-a-message, read-a-conversation, and delivery/read receipts. I'll treat search, media, and group admin as out of scope unless you want them."

**3. Non-functional requirements — this is where seniority shows.**
Availability target · latency target (p99, not average) · consistency needs *per operation* · durability ("can we lose a message? a payment?") · scale · security/compliance.
> "Messages must never be lost — durability is non-negotiable. Ordering within a conversation matters. Global ordering across conversations doesn't."

**4. Estimate scale.** QPS (avg and peak), storage/year, bandwidth, and the read:write ratio. **End with a sentence naming the hard dimension.**
> "50k read QPS, 300 write QPS, 1 TB/year. It's read-heavy by 200:1, so the design problem is read amplification, not write throughput."

### Phase B — Design the core *(15 min)*

**5. Define the API.** A few endpoints with their inputs/outputs. This forces precision and pins down the data contract.
> `POST /messages {conversation_id, client_msg_id, body} → 202 {message_id, seq}`

**6. Design the data model.** Entities, keys, relationships, access patterns. **Choose the store *after* you know the access patterns**, and say why.
> "Messages are keyed `(conversation_id, seq)` because every read is 'last N in this conversation' — so I want them physically clustered by conversation."

**7. High-level architecture.** Draw the happy path end to end: client → LB → service → data store. Keep it boring and complete. No premature microservices, no components you can't justify.

### Phase C — Make it survive *(15 min)*

**8. Identify the bottleneck.** Say it explicitly, backed by your estimate. *"At 50k QPS the database is the first thing to break."* Everything after this is a response to a named problem — that's what makes a design sound reasoned rather than memorized.

**9. Scale the read path.** Stateless app tier + horizontal scaling → cache → read replicas → CDN. In that order.

**10. Scale the write path.** Batching, async processing, partitioning/sharding with a named shard key, and what that key costs you.

**11. Add async processing.** Move everything off the critical path that the user doesn't need to wait for. Name the queue, the workers, and what the user sees meanwhile.

**12. Handle failure.** For each component: what happens when it's down, slow, or partitioned? Timeouts, retries + backoff, idempotency, circuit breakers, graceful degradation, dead-letter queues. **Explicitly name your single points of failure.**

**13. Add observability.** What are the 3–4 metrics you'd alert on? What SLO? How would you debug "some users say messages are delayed"?

### Phase D — Defend it *(7 min)*

**14. State trade-offs deliberately.** Name 3 choices you made, the alternative, and why you chose as you did. *"I chose fan-out on write for feed reads; it costs a lot of write amplification and I'd need a fallback for celebrity accounts."*

**15. Name what you'd do next.** *"If this grew 10×, the next thing to break is X, and I'd address it with Y."* This shows you know the design has a lifespan, which is the truth about all designs.

### Things to say out loud (they earn real credit)
- "Let me estimate before I design, so I know what's actually hard."
- "I'll start with the simplest thing that works and scale it when I can name the bottleneck."
- "That's a trade-off: I'm buying X with Y."
- "This is a single point of failure; here's how I'd remove it — or why I'd accept it for now."
- "I'd measure this before optimizing it."

---

# Part 2 — Ten systems, the 20% that matters

For each: the shape of the problem, the core design, and **the one or two things the interview is really about**.

---

## 1. URL Shortener
**Really about:** ID generation, read-heavy caching, and storage estimation. The classic warm-up.

- **Scale:** 100M new URLs/day is unrealistic; use 1M writes/day, 100:1 read ratio → ~10 write QPS, ~1000 read QPS. Storage: 500 bytes × 365M/yr ≈ 180 GB/yr. **Trivial writes, heavy reads.**
- **API:** `POST /urls {long_url, custom_alias?, ttl?} → {short_url}`; `GET /{code} → 301/302`.
- **Key generation — the real question.** Options: (a) hash the URL (MD5) and take 7 chars, handling collisions; (b) **base62-encode an auto-increment / snowflake ID** — no collisions, but sequential and enumerable; (c) pre-generate a key pool in a separate table and hand keys out. Base62 of a distributed unique ID is the clean answer. 62^7 ≈ 3.5 trillion codes.
- **Data:** `urls(code PK, long_url, user_id, created_at, expires_at)`. A single key-value lookup — **this is a case where a KV store or heavily cached Postgres both work.**
- **Reads:** Redis cache with a long TTL (URLs are immutable — this is the rare case where caching is genuinely easy), plus a CDN in front of the redirect if global.
- **301 vs 302:** 301 is cached by browsers forever (fast, but you lose click analytics and can never change the target). **302 is usually the right answer** because analytics matter.
- **Analytics:** don't write a row synchronously per click — fire an event to a queue and aggregate asynchronously.
- **Trade-off to name:** custom aliases need a uniqueness check → a UNIQUE constraint, and a race you must handle at the DB level.

## 2. Chat Application
**Really about:** stateful connections and message ordering/delivery guarantees.

- **Scale:** 10M DAU, 50 messages/user/day → 500M messages/day ≈ 6k writes/s avg, ~15k peak. Millions of *concurrent WebSocket connections* — that's the hard dimension, not QPS.
- **Connections:** WebSocket gateway servers, one persistent connection per user. **Problem: user A is connected to gateway 1, user B to gateway 7.** Solution: a shared pub-sub layer (Redis pub/sub or Kafka) — the gateway subscribes to channels for its connected users; the message service publishes; the right gateway delivers. Maintain a `user_id → gateway` presence map in Redis with a TTL heartbeat.
- **Data model:** `messages(conversation_id, seq, sender_id, body, created_at)`, partitioned/clustered by `conversation_id`, with `seq` from a per-conversation counter. Reads are always "last 50 in conversation X" → keyset pagination on `seq`.
- **Delivery guarantee:** client generates a `client_msg_id`; server dedupes on it (idempotency) and assigns the authoritative `seq`. Ack back to the sender. Offline users: persist, then push via APNs/FCM.
- **Ordering:** guarantee it *per conversation* only. Global ordering is unnecessary and expensive — say this.
- **Trade-offs to name:** WebSockets break statelessness, so deploys must drain connections gracefully (clients need reconnect + resume-from-`seq` logic). Group chat fan-out: write once, fan out on read for large groups.

## 3. Notification System
**Really about:** queues, fan-out, retries, idempotency, and rate limiting. **The most "backend-realistic" design of the ten.**

- **Shape:** an API takes a notification request, resolves recipients and their preferences, renders per-channel content, and dispatches to email/SMS/push providers.
- **Architecture:** `API → validate → enqueue → dispatcher → per-channel queues → channel workers → provider`.
- **Why per-channel queues:** SMS providers rate-limit differently from push, and a failing email provider must not block push delivery. **Isolation is the design point** (bulkheads).
- **Must-haves:** idempotency key per notification (never send the same alert twice); user preferences and quiet hours checked at dispatch, not at enqueue; retry with exponential backoff; dead-letter queue for permanently failed sends; per-user rate limiting/digesting so a bug can't send 10,000 emails.
- **Templates:** versioned, with locale support, rendered at send time.
- **Prioritization:** separate queues for transactional (password reset, OTP) vs marketing. Never let a bulk campaign delay an OTP.
- **Observability:** delivery rate, bounce rate, queue depth, consumer lag, provider error rate per channel.
- **Trade-off:** at-least-once delivery means duplicates are possible; dedupe on a `(user, notification_key)` UNIQUE within a time window.

## 4. E-commerce System
**Really about:** transactions, inventory under concurrency, and separating strong from eventual consistency.

- **Core services:** catalog (read-heavy, cacheable, eventually consistent), cart (session-ish, Redis + durable backup), **inventory (strongly consistent)**, orders (strongly consistent), payments (external, idempotent), search (Elasticsearch, async-indexed).
- **The inventory question is the interview.** Two customers, one item left. Answer: a conditional atomic update — `UPDATE inventory SET available = available - 1 WHERE sku = ? AND available >= 1` — and check rows-affected. No read-modify-write. For high-contention flash sales, pre-allocate into Redis with a compensating reconciliation, and accept complexity only if the scale demands it.
- **Reservations:** hold stock for 15 minutes during checkout, with a TTL-based expiry job to release abandoned carts.
- **Order flow as a saga:** create order (pending) → charge payment → reserve inventory → confirm. Any failure triggers compensations (refund, release). Use the **outbox pattern** so "order created" events can't be lost or phantom-published.
- **Consistency split to state explicitly:** product listings, reviews, recommendations, and search = eventually consistent and cached. Inventory decrement, order creation, payment = strongly consistent, single database, real transactions.
- **Prices:** snapshot price and product name onto `order_items` at purchase time. This is denormalization as *historical correctness*, not as optimization.

## 5. File Storage System (Dropbox-like)
**Really about:** object storage, metadata vs blobs, chunking, and deduplication.

- **Split the two:** blob bytes in object storage (S3); metadata in a relational DB (`files`, `versions`, `chunks`, `permissions`). Never put bytes in the database.
- **Upload path:** client asks the API for a **presigned URL** and uploads directly to S3 — bytes never traverse your servers. This one decision removes your biggest bandwidth and scaling problem; say it early.
- **Chunking:** split files into ~4 MB chunks, hash each (SHA-256). Enables resumable uploads, delta sync (only changed chunks), and **deduplication** — if the hash exists, don't store it again. Dedup across all users is a huge storage win.
- **Sync:** clients hold a cursor; long-poll or WebSocket for change notifications; reconcile by comparing chunk hashes.
- **Conflicts:** last-write-wins is simplest but loses data; the pragmatic answer is to keep both versions ("conflicted copy") and let the user decide.
- **Sharing/permissions:** an ACL table checked before issuing any presigned URL; presigned URLs must be short-lived.
- **Trade-off:** metadata DB is the bottleneck and the thing to shard (by `user_id`), not the blob store — S3 scales itself.

## 6. Social Media Feed
**Really about:** fan-out on write vs read, and the celebrity problem. **The canonical "there's no free lunch" design.**

- **Fan-out on write (push):** when a user posts, write the post ID into every follower's precomputed feed list (Redis list per user). Reads are O(1) and instant. **Cost:** a user with 10M followers generates 10M writes per post.
- **Fan-out on read (pull):** at read time, fetch recent posts from everyone you follow and merge. Writes are cheap. **Cost:** reads are expensive and slow, especially for users following thousands.
- **The real answer is hybrid:** push for normal users, pull for celebrities (above a follower threshold). A user's feed = their precomputed list merged at read time with the small number of celebrities they follow. **Name this hybrid explicitly — it's the whole point of the question.**
- **Storage:** feeds in Redis, capped at ~500–1000 entries (nobody scrolls further); posts in the database/object storage; media on a CDN.
- **Ranking:** start chronological, mention that a ranking service can re-order the candidate set, and don't rabbit-hole into ML.
- **Consistency:** eventual is completely fine — nobody can tell if a post appears 2 seconds late. **Except for your own posts**, which must appear immediately (read-your-own-writes: merge the user's own recent posts client-side or from the primary).

## 7. Food Delivery System
**Really about:** geospatial queries, real-time tracking, and multi-party state machines.

- **Entities:** customers, restaurants, couriers, orders. The order is a **state machine**: placed → accepted → preparing → picked_up → delivered (with cancellation paths). Model it explicitly and guard transitions in the database (`UPDATE ... WHERE status = 'accepted'`), never in application `if`s alone.
- **Geospatial:** "restaurants near me" and "couriers near this restaurant" → geohash or PostGIS/S2 cells; Redis `GEOADD`/`GEOSEARCH` works well for the live courier index.
- **Courier location:** high-write, low-value-per-write (every 5s). Write to Redis, not Postgres; persist a downsampled track asynchronously. This is a good place to demonstrate you know not everything needs durability.
- **Dispatch/matching:** a background service that scores nearby couriers (distance, current load, direction) and assigns. Needs to be idempotent and handle courier rejection and timeouts.
- **Real-time updates to the customer:** WebSocket or push notifications, driven by order state changes.
- **Trade-off:** the matching service is stateful and hard to scale; partition it geographically (by city/region) — geography is a natural, non-hot shard key.

## 8. Payment System
**Really about:** correctness, idempotency, and auditability. **You will be judged on rigor, not scale.**

- **Non-negotiables:** money is never a float (integer minor units); every state change is an append-only ledger entry (double-entry bookkeeping: every transaction has balanced debits and credits); nothing is ever hard-deleted.
- **Idempotency is the core.** Every charge carries an `Idempotency-Key` with a UNIQUE constraint; a retry returns the original result. Say exactly how you'd store and check it.
- **External provider calls:** the dangerous window is "we called Stripe and the response was lost." Resolution: write an intent row *before* calling, with the idempotency key, then reconcile by querying the provider for that key on retry. **Never assume a timeout means it didn't happen.**
- **Async + webhooks:** payment authorization is async in reality; you accept the request (`202`), process it, and receive a provider webhook. Webhooks must be verified (signature), idempotent (they're delivered more than once), and processed via a queue.
- **Reconciliation:** a daily job comparing your ledger against the provider's settlement report. Every real payment system has one. Mentioning it signals genuine experience.
- **Consistency:** strong, always. This is the system where you say "I choose consistency over availability during a partition — refusing a payment is recoverable; double-charging is not."
- **Security:** never store raw card data (use provider tokens, stay out of PCI scope), encrypt at rest, audit log every access.

## 9. Video Streaming System
**Really about:** the CDN, the transcoding pipeline, and adaptive bitrate. **The storage/bandwidth estimate does the talking.**

- **Estimate first:** 1 hour of 1080p ≈ 1–3 GB; 500k concurrent viewers at 5 Mbps = 2.5 Tbps of egress. **The conclusion is immediate: this system is a CDN problem, and bandwidth is the dominant cost.**
- **Upload → process:** upload to object storage via presigned URL → enqueue a transcoding job → workers produce multiple renditions (240p…4K) → segment into ~4–10s chunks → generate an HLS/DASH manifest → push to CDN.
- **Transcoding** is embarrassingly parallel: split the video, transcode chunks in parallel across a worker fleet, reassemble. Perfect illustration of a queue + worker pool.
- **Playback:** client fetches the manifest, then fetches segments, switching renditions based on measured bandwidth (**adaptive bitrate**). Segments are immutable → infinitely cacheable → near-100% CDN hit rate.
- **Metadata** (titles, view counts, watch history) is a completely ordinary database + cache problem — don't let it eat your time.
- **Live streaming variant:** the trade-off is latency vs reliability. Smaller segments = lower latency = more requests and more re-buffering; low-latency HLS or WebRTC for sub-second.
- **Trade-off to name:** storing every rendition of every video costs a fortune — transcode popular content eagerly, unpopular content on demand.

## 10. Search System
**Really about:** the inverted index, and indexing as an asynchronous pipeline.

- **Core idea:** an **inverted index** maps term → list of document IDs. That's it. Build the intuition: tokenize → normalize (lowercase, stem) → for each term, store a posting list. Query = intersect posting lists + rank.
- **Ranking:** TF-IDF/BM25 as a baseline (rare terms matter more; long documents matter less). Mention you'd layer business signals (recency, popularity) on top. Don't go deeper unless asked.
- **The architecture is a pipeline:** source of truth (Postgres) → CDC or outbox events → indexing workers → search cluster. **Eventually consistent by design**, and say so: new items appear in search within seconds, not instantly.
- **Serving:** shard the index by document (scatter-gather across shards, merge top-k), replicate each shard for read scale and availability. Cache popular queries.
- **Autocomplete** is a separate system: a trie or a prefix index in Redis, driven by popular queries — not your main search cluster.
- **Trade-off:** the search index is never the system of record. Rebuilding it from the database must be a routine, tested operation — not an emergency.

---

## How to practice these

Do not read them again. **Do this instead:**

1. Set a 45-minute timer. Whiteboard (paper is fine) and talk out loud.
2. Run the framework in order. Do not skip estimation — that's the step people skip and the step interviewers notice.
3. Force yourself to name: the bottleneck, three trade-offs, and every single point of failure.
4. Only then read the notes above and list what you missed.
5. Redo the same system a week later. The second pass is where the fluency forms.

Two to three systems per week for four weeks covers all ten, twice.
