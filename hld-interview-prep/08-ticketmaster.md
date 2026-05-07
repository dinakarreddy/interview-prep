# 08 — Design a Ticketmaster / Flash-Sale System

> The "high-contention finite inventory under massive simultaneous demand" archetype. The interviewer cares less about the catalog and more about whether you correctly identify that **inventory contention** and the **virtual waiting room** are the only things that matter, and you spend your time there. Taylor Swift on-sale: 14M people queuing for 50K seats. That single number is the design.

---

## 0. Problem statement (as the interviewer would drop it)

> **Interviewer:** "Design a system like Ticketmaster. Users browse events, pick seats, and check out. The interesting cases are popular on-sales — Taylor Swift, Super Bowl, Coachella — where you have, say, fifty thousand seats and millions of people hitting the system at the exact same second. Walk me through how you'd build it."

That's the whole problem. Your job: do not start drawing services. Scope it first, then attack the two things that make this problem different from any other e-commerce system.

---

## 1. Clarifying questions

Pick 5–6. Interleave with the interviewer — don't fire a checklist.

> **Candidate:** "Before I design, let me scope this. A few questions:"
>
> 1. **"What's the spike profile? Steady-state e-commerce or massive flash sales?"** — *Expect: both, but the design is dominated by the flash-sale shape. Taylor Swift = 14M concurrent users on one event, 50K seats, sale opens at a fixed minute.*
> 2. **"Is it assigned-seat or general admission?"** — *Critical. Assigned seat means each seat is a unique inventory unit (50K SKUs per event); GA means a single counter (1 SKU). Assume assigned for the hard version.*
> 3. **"How long does a hold last during checkout?"** — *Expect 7–10 minutes. Drives the inventory locking model and TTL behavior.*
> 4. **"Do we own payments, or is it Stripe/Braintree?"** — *Assume integrated third-party. Payment is slow (3–5s) and fallible — that shapes seat-release semantics.*
> 5. **"What's the latency target during sale-start?"** — *Expect: page renders < 1s, seat-pick acknowledgement < 200ms, queue position update every few seconds. Latency degrades gracefully (rather than erroring) under load.*
> 6. **"What's the consistency requirement on inventory? Is overselling acceptable?"** — *Zero overselling. Underselling (a seat dropped due to crash) is recoverable; overselling is a refund nightmare and a legal mess.*
> 7. **"Is fairness a product requirement or just FIFO?"** — *Expect: fairness is real. Verified Fan presale, lottery-based admission, anti-bot. Not just first-click-wins.*
>
> **Out of scope (state explicitly):**
> - User authentication / identity (assume an Auth service exists)
> - Notification / email delivery (separate problem)
> - Recommendations / discovery
> - Secondary market (resale)
> - Refunds / customer service
> - Event creation by promoters (CMS side)
>
> **Candidate:** "I'm going to design the **on-sale critical path**: virtual waiting room → seat selection → hold → checkout → payment → ticket issuance, plus the steady-state browse/catalog read path. The dominant constraints are the 280× read amplification before a hot sale and the 14M:50K contention ratio at sale-start. Sound right?"

> **Interviewer:** "Yes. Focus on the spike."

---

## 2. Functional requirements

| Priority | Requirement |
|---|---|
| P0 | User browses events list and event detail page (heavily cached pre-sale) |
| P0 | User joins virtual waiting room before sale start; sees position |
| P0 | At sale start, system admits users at a controlled rate to seat-selection |
| P0 | User selects 1–8 seats from a real-time seat map |
| P0 | Selected seats are held for the user with a TTL (7–10 min) |
| P0 | User checks out: payment via integrated processor; on success, ticket issued |
| P0 | If hold expires or payment fails, seats are released back to inventory |
| P0 | Zero oversell, ever |
| P1 | Verified Fan / tiered presale (lottery-based admission codes) |
| P1 | Bot defense: CAPTCHA, device fingerprinting, account-age checks |
| P1 | Per-event admission rate-limit knob (operator can dial up/down live) |
| P2 | Best-available auto-pick (system picks N best seats together) |
| Out | Auth, notifications, recommendations, secondary market, refunds, CMS |

---

## 3. Non-functional requirements

| Dimension | Target | Justification |
|---|---|---|
| Availability — browse | 99.95% | Front door; users discovering events |
| Availability — sale path | 99.99% during a major on-sale (≤ 4 min downtime/year on the hot path) | Lost revenue + brand damage if we're down on Taylor Swift day |
| Latency — event page | p99 < 800ms first paint pre-sale; CDN-cached | 280× read amplification means we serve from edge, not origin |
| Latency — seat-pick ack | p99 < 200ms | User sees a seat turn from green to yellow in real-time |
| Latency — checkout submit | p99 < 5s end-to-end (payment dominates) | Payment processor RTT |
| Consistency — inventory | Linearizable on the seat object (no double-sell) | Hard correctness requirement |
| Consistency — seat map UI | Eventually consistent within 1–2s | Seat map lag is OK; oversell is not |
| Durability | Zero ticket loss after payment captured | Money has changed hands |
| Scale — peak users on a single event | 14M concurrent waiting-room sessions | Taylor Swift on-sale public number |
| Scale — peak request rate at sale start | ~5M req/sec aggregate (mostly waiting-room polls) | 14M users polling every 2–3s |
| Scale — write rate (hot path) | 50K successful seat-claims spread over ~10 min ≈ 80–500 writes/sec for *successful* claims, but contention attempts an order higher | One write per seat sold |

---

## 4. Capacity estimation

> **Candidate:** "Let me ground this in the Taylor Swift number, then back into steady-state. The flash-sale shape is the dominant design driver — steady-state is easy."

### Steady-state (whole platform)

- **100M ticketed events / year** (industry-rough; mix of small clubs, concerts, sports).
- **500K daily ticket sales** at the median day → ~6 sales/sec average platform-wide.
- **Read:write ratio** in steady state: ~50:1 (browsing vs purchasing).
- **Average reads/sec** ≈ 300/sec. Trivial.

### Flash-sale (Taylor Swift class — the interesting case)

- **14M concurrent users** queuing to buy.
- **50K seats** in the venue.
- **Read amplification at sale start** = 14M / 50K = **280×**. (For every successful purchase, 280 people *also* tried.)
- **Sale duration**: ~10 minutes from open to sold-out.
- **Successful writes**: 50K seat-claim writes spread over 600 seconds = **83 writes/sec sustained**, but bursty (peaks of ~500/sec in the first 30 seconds).
- **Attempted writes** under contention: easily 100×–500× the successful rate, depending on retry behavior. So the inventory subsystem must absorb **~50K contention attempts/sec** at peak even though only ~500 succeed.
- **Waiting-room polls**: 14M users × 1 poll every 3 sec = **~5M req/sec** at peak. This is the dominant load on the system, by a huge margin.
- **CDN-shielded reads** (event page before sale start): millions of refresh-clicks in the seconds before T-0. Must be 100% CDN-served — origin would melt.

### Storage

- **Events table**: ~100M lifetime events × ~5 KB metadata = **500 GB**. Trivial.
- **Seats table**: 100M events × avg 2K seats = 200B rows lifetime, but most are immutable post-sale. Active hot set: a few thousand events × 50K seats = **a few hundred million** active seats at any time.
- **Orders table**: 500K/day × 365 = 180M/year × ~2 KB = **~360 GB/year**. Manageable.
- **Audit log (Kafka)**: every inventory action persisted for reconciliation. ~10× the order volume. Retention 30 days → **~10 TB** rolling.

### Bandwidth

- Edge egress at peak: ~14M concurrent users × ~5 KB/poll × 1/3s = **23 GB/s** at the CDN. Static-served from edge.
- Origin egress at peak (after CDN absorbs): ~50K req/sec × 5 KB = **250 MB/s** to origin services. Tractable.

### The single number that drives the design

**14M:50K = 280:1 read-amplification, with the writes being on a contended hot key (a single venue's inventory) within a 10-minute window.**

That's not an e-commerce problem. That's a flash-sale problem. The whole architecture is built around it.

---

## 5. API design

> **Candidate:** "Public REST surface. I'll keep this focused on the on-sale path."

### Browse / catalog (read-heavy, CDN-cached)
```
GET /v1/events?city=&date=&genre=        → 200 [event summaries]
GET /v1/events/{event_id}                → 200 {event metadata, sale_start_ts, status}
GET /v1/events/{event_id}/seatmap        → 200 {sections, layout, prices}  (CDN, immutable)
```

### Virtual waiting room
```
POST /v1/events/{event_id}/queue/join
  body: { user_id, captcha_token, device_fingerprint }
  → 200 {queue_token, position, estimated_wait_seconds}

GET /v1/queue/{queue_token}/status
  → 200 {position, status: "waiting"|"admitted"|"expired", admit_token?}
  (long-poll or short-poll; client polls every 2–3s)

GET /v1/queue/{queue_token}/heartbeat   (keeps queue position alive while tab is open)
  → 204
```

### Seat selection (post-admission)
```
GET /v1/events/{event_id}/availability?section={section_id}
  Headers: Admit-Token: <signed>
  → 200 {seats: [{seat_id, status: "open"|"held"|"sold"}]}   (~1s stale OK)

POST /v1/events/{event_id}/holds
  Headers: Admit-Token: <signed>
  body: { seat_ids: [s1, s2, s3], idempotency_key }
  → 201 {hold_id, seat_ids, expires_at}        (success)
  → 409 {conflicting_seat_ids, current_status} (one or more taken)
```

### Checkout
```
POST /v1/holds/{hold_id}/checkout
  body: { payment_method_token, billing_address, idempotency_key }
  → 200 {order_id, status: "confirmed"|"payment_pending", tickets: [...]}
  → 402 {reason: "payment_failed"} → seats auto-released

DELETE /v1/holds/{hold_id}    (user backs out)
  → 204 → seats released immediately
```

### Anti-patterns to avoid
- **Don't return raw seat IDs in the seatmap public API** — seat IDs should be opaque/signed so attackers can't fuzz POST /holds with arbitrary IDs.
- **Don't make `POST /holds` non-idempotent** — clients will retry on timeout; without idempotency keys, retries cause double-holds and the user thinks they got nothing.
- **Don't trust client-side timers** — sale-start, hold expiry, admission times are all server-issued. Client clocks are adversarial.
- **Don't put admission state in the URL** — `Admit-Token` is a signed JWT-ish artifact bound to (user, event, admit_time). One token, one user, one event.

---

## 6. Data model

### `events` (PostgreSQL — small table, low write rate)
| Column | Type | |
|---|---|---|
| event_id | uuid | PK |
| name | text | |
| venue_id | uuid | FK |
| sale_start_ts | timestamptz | |
| sale_end_ts | timestamptz | |
| status | enum (draft, presale, on_sale, sold_out, postponed) | |
| total_seats | int | |
| seats_sold | int | denormalized; eventually-consistent counter for browse views |

### `seats` (PostgreSQL — sharded by event_id)
| Column | Type | |
|---|---|---|
| seat_id | uuid | PK |
| event_id | uuid | shard key |
| section | text | |
| row, num | text, int | |
| price_cents | int | |
| status | enum (open, held, sold) | ground truth, but most reads come from Redis |
| version | int | optimistic CAS |
| held_by_user_id | uuid | nullable |
| hold_expires_at | timestamptz | nullable |

> The PG seats table is the **ledger of record**, not the hot read path. The hot path is Redis.

### `orders` (PostgreSQL — sharded by order_id; ACID required)
| Column | Type | |
|---|---|---|
| order_id | uuid | PK |
| user_id | uuid | |
| event_id | uuid | |
| seat_ids | uuid[] | |
| total_cents | int | |
| status | enum (pending, paid, failed, refunded) | |
| payment_intent_id | text | from Stripe |
| created_at | timestamptz | |
| idempotency_key | text | unique |

### `holds` (Redis — TTL'd hash; PG mirror for durability)
- Key: `hold:{hold_id}` → `{user_id, event_id, seat_ids, expires_at}`
- TTL: 7–10 min (matches business policy)
- Mirror written async to PG `seats.held_by_user_id` for crash recovery

### Inventory (Redis — the hot path)
- Per-event hash: `seats:{event_id}` → `{seat_id: status_byte}` where `status_byte ∈ {O, H, S}`
- Per-section atomic counter: `available:{event_id}:{section_id}` → integer (used for "best available" picks and section-level rate limiting)
- Pre-warmed at sale-start T-5min by an Inventory Loader job (PG → Redis)

### Why these choices

- **PostgreSQL for orders**: ACID is non-negotiable. Money has changed hands. Need transactional guarantees: did this user pay, and which seats?
- **PostgreSQL for seats (ground truth)**: small enough table, sharded by event, supports SQL audit queries.
- **Redis for inventory hot path**: single-threaded → atomic operations are O(1) and *naturally serializable*. `WATCH/MULTI/EXEC` or Lua scripts give us CAS-like semantics. 100K ops/sec/instance. No leader election. No multi-version overhead.
- **Kafka for audit**: every inventory action (hold-created, hold-released, seat-sold) emitted to a topic. Powers reconciliation between Redis and PG, plus downstream (analytics, fraud).
- **Snowflake-style IDs for hold_id, order_id**: time-sortable, no central allocator, debuggable.

---

## 7. High-level architecture

```
                    ┌─────────────────────────────────────────────────────┐
                    │              CDN + WAF + Bot Defense                │
                    │       (Cloudflare/Akamai — bot mgmt, JA3,           │
                    │        device fingerprinting, CAPTCHA challenge)    │
                    └─────────────────────┬───────────────────────────────┘
                                          │
                                ┌─────────▼──────────┐
                                │   API Gateway      │ (auth, rate-limit, mTLS)
                                └────┬────────┬──────┘
                                     │        │
            ┌────────────────────────┘        └───────────────────────────┐
            │ steady-state browse                              on-sale path│
            ▼                                                              ▼
    ┌────────────────┐                                       ┌────────────────────┐
    │  Catalog Svc   │                                       │ Waiting Room Svc   │
    │  (event meta,  │                                       │ (queue per event,  │
    │   seatmap)     │                                       │  admission policy) │
    └────────┬───────┘                                       └────┬──────────┬────┘
             │                                                    │          │
             ▼                                                    ▼          │ admit
        ┌────────┐                                          ┌──────────┐     │ token
        │ Redis  │                                          │  Redis   │     │
        │ (event │                                          │ (per-    │     │
        │ cache) │                                          │  event   │     │
        └────────┘                                          │  queue   │     │
                                                            │  ZSET)   │     │
                                                            └──────────┘     │
                                                                             │
                                                            ┌────────────────▼────┐
                                                            │  Inventory Svc      │
                                                            │  (atomic seat hold) │
                                                            └─┬────────────────┬──┘
                                                              │                │
                                                              ▼                ▼
                                                         ┌────────┐      ┌─────────┐
                                                         │ Redis  │      │  Kafka  │
                                                         │ seats: │      │ inv.    │
                                                         │ {evt}  │      │ events  │
                                                         └───┬────┘      └────┬────┘
                                                             │ async           │
                                                             ▼                 ▼
                                                         ┌────────┐    ┌─────────────┐
                                                         │  PG    │    │ Reconciler  │
                                                         │ seats  │    │ (Redis ↔ PG)│
                                                         │(ledger)│    └─────────────┘
                                                         └────────┘

                            ┌────────────────┐         ┌────────────────┐
                            │ Hold Svc (TTL  │◀────────│ Checkout Svc   │
                            │ enforcement,   │         │ (orchestrates  │
                            │ release on     │         │  payment, seat │
                            │ expiry)        │         │  finalization) │
                            └───────┬────────┘         └───────┬────────┘
                                    │                          │
                                    │                          ▼
                                    │                  ┌──────────────┐
                                    │                  │  Payment Svc │──→ Stripe / Braintree
                                    │                  └──────┬───────┘
                                    │                         │
                                    └────────────┬────────────┘
                                                 ▼
                                          ┌────────────┐
                                          │ Orders DB  │ (PostgreSQL, ACID)
                                          └────────────┘
```

### Request walk-through — the on-sale critical path

**Phase 0 — pre-sale (T-1 hour to T-0).**
1. Event page is **fully static** at the CDN. The "Buy Tickets" button is greyed out, with a server-issued countdown.
2. Inventory Loader pre-warms Redis at T-5min: PG → Redis seat status hash, per-event.
3. Waiting Room Service pre-creates the queue ZSET in Redis.
4. Bot defense is at maximum aggression: CAPTCHA challenges, JA3 fingerprinting, IP reputation.

**Phase 1 — sale-start (T-0 to T+30s).**
1. Client `POST /v1/events/{id}/queue/join` (clicks Buy).
2. WAF + bot-defense scoring at the edge. High-risk requests get an interactive challenge.
3. Request hits API Gateway → Waiting Room Service.
4. Waiting Room Service does `ZADD queue:{event_id} {randomized_score} {user_id}` — **randomized score, not a strict timestamp**. This is the lottery (more in §8.1).
5. Returns `queue_token` + a polling cadence. Client now polls `/queue/{token}/status` every 2–3s.
6. Behind the scenes, Waiting Room admits N users/min by `ZPOPMIN`-ing from the ZSET and minting **signed admit tokens** (JWT-like, bound to user_id+event_id+ttl).

**Phase 2 — admitted, pick seats.**
7. Client receives `{status: admitted, admit_token}`. Redirects to seat-pick page.
8. Client `GET /v1/events/{id}/availability?section=A` — returns Redis seat map (slightly stale, that's fine).
9. User clicks seats. Client `POST /v1/events/{id}/holds {seat_ids: [...], idempotency_key}` with `Admit-Token` header.
10. Inventory Service runs an **atomic Lua script** in Redis:
    - For each seat: check status == OPEN; if all OPEN, set to HELD, write `held_by` and `expires_at`.
    - Either all-or-nothing.
11. On success: emit `hold.created` to Kafka, write hold record to Redis with TTL, return 201.
12. On conflict: return 409 with which seats are gone, client re-renders, user picks again.

**Phase 3 — checkout.**
13. User clicks "Checkout". Client `POST /v1/holds/{hold_id}/checkout`.
14. Checkout Service:
    - Validates hold still alive (Redis `EXISTS` + TTL > 0).
    - Creates `orders` row in PG with status=`pending`, idempotency_key.
    - Calls Payment Service → Stripe (3–5s typical).
15. On payment success:
    - Atomically: orders.status → `paid`, seats Lua-script transition HELD → SOLD, hold removed.
    - Emit `order.confirmed` to Kafka. Tickets generated downstream.
16. On payment failure:
    - orders.status → `failed`, seats released back to OPEN, hold removed.
    - 402 to client; user can retry.

**Phase 4 — hold expiry (background).**
17. Hold Service watches for expired holds (Redis keyspace notifications OR a periodic scan).
18. Expired hold → seats transitioned HELD → OPEN via the same atomic Lua script.
19. Emit `hold.expired` to Kafka.

### Why this architecture

- **Edge does most of the work** during pre-sale. The CDN is the first line of defense against the read storm.
- **Waiting room is the airlock**. We intentionally throttle the rate at which traffic reaches Inventory Service. Without it, 14M users would simultaneously POST /holds and melt the system. With it, only a controlled trickle (5K–10K admitted users/min) ever reaches the inner services.
- **Redis is the inventory hot path.** Single-threaded → naturally atomic. Lua scripts give us multi-key transactions. PG is the ledger, written async.
- **Kafka decouples** the audit/reconciliation/analytics consumers from the hot path. They lag; that's fine.
- **PG remains the source of truth for orders.** Money requires ACID. Inventory is "fast and atomic" in Redis; settled in PG.

---

## 8. Deep dives

### Deep dive 1: Inventory contention — the heart of the problem

> **Interviewer:** "Walk me through what happens at the exact moment the sale starts. 14 million people, 50 thousand seats."

**Candidate:**

This is the single hardest part of the system. Let me lay out the options and pick.

**Option A — Pessimistic row lock in PostgreSQL**
- `SELECT ... FOR UPDATE` on the seat row, then UPDATE.
- Works at small scale: a single PG primary handles ~5K writes/sec, and even less with row-level lock contention.
- **Falls over here**: a single hot venue's inventory is a single PG instance. 50K contention attempts/sec serialize through one row lock per seat. We'd see lock-wait timeouts cascading. **This does not scale.**

**Option B — Optimistic concurrency (CAS / version compare-and-swap)**
- Read seat with version, attempt UPDATE with `WHERE version = old_version`. If 0 rows updated, retry.
- Better at moderate contention because there's no blocking — losers fail fast and retry on a different seat.
- **Falls over**: under 100K-attempts/sec on the same hot section, retry storms make it pathological. Every loser retries; many lose again. CPU goes to retry, not progress.

**Option C — Redis atomic decrement / Lua script**
- Pre-load all seats into Redis at T-5 min as a hash: `seats:{event_id}` → seat_id → status.
- A hold attempt is a Lua script: read N seats, check all OPEN, set all to HELD with `held_by` + `expires_at`, return success.
- Redis is **single-threaded** — Lua scripts run atomically. No locks, no contention; serialization is implicit and O(1).
- 100K ops/sec/instance comfortable. Even a single Redis can absorb the **successful** write rate (500/sec) with 200× headroom.
- The contention is moved to "who gets the seat"; users who lose see 409 immediately and retry on a different seat.
- **PG is the ledger**, written async via the inv.events Kafka topic.

**Option D — Sharded inventory (virtual sub-sections)**
- Split a section into K virtual sub-sections (say, 100 sub-shards of 5 seats each). Users assigned to a random sub-shard at admission. Each sub-shard has its own counter.
- Reduces contention K-fold.
- **Cost**: a user's sub-shard might sell out before other sub-shards are touched. They'd see "no seats left" while seats objectively exist. **Starvation / fairness regression.**
- Useful for **GA tickets** (one big counter), less so for assigned seats where users want to *see* and *pick*.

**My pick: C (Redis Lua) for assigned seats, with D (sharded counters) for GA-only tiers.**

Defense:
- Redis single-threading is exactly what we need: serialization without coordination. Lua atomicity gives us multi-seat all-or-nothing.
- Successful write rate (500/sec) is well under Redis's capacity. Failed write rate (~50K/sec under contention) is also fine — Lua scripts are short.
- PG as ledger: asynchronously catches up via Kafka consumer. We tolerate a few seconds of Redis-PG drift.
- Failure mode: Redis crash → AOF + replica → fail over in seconds. Worst case, some pending state lost; reconciler catches it.

**The Lua script (sketch):**

```lua
-- KEYS = [seats_key], ARGV = [user_id, expires_at, seat_id1, seat_id2, ...]
local seats_key = KEYS[1]
local user_id = ARGV[1]
local expires_at = ARGV[2]
local seat_ids = {unpack(ARGV, 3)}

-- Pass 1: check all open
for i, sid in ipairs(seat_ids) do
  if redis.call('HGET', seats_key, sid) ~= 'O' then
    return {err = 'CONFLICT', seat = sid}
  end
end

-- Pass 2: hold all
for i, sid in ipairs(seat_ids) do
  redis.call('HSET', seats_key, sid, 'H')
  redis.call('HSET', seats_key..':meta:'..sid, 'held_by', user_id, 'expires', expires_at)
end

return {ok = true}
```

The Lua execution is atomic from Redis's perspective. Either all set, or none.

> **Interviewer:** "What if Redis crashes mid-sale?"

**Candidate:**
- AOF set to `everysec` — at most 1s of writes lost on crash.
- Redis replica in another AZ; failover ~10s with Sentinel/Cluster.
- During failover, hold attempts get 503; clients retry with backoff.
- Worst case: a few hundred holds lost. Reconciler compares the last-flushed Kafka inventory log against PG — any seats marked HELD with no corresponding Kafka event get released.
- Critical: the **Kafka log** is durable, replicated, and is what we use for reconciliation. Redis is the fast hot path, not the source of truth.

> **Interviewer:** "How do you prevent overselling under all of this?"

**Candidate:**
- Overselling can only happen if two separate writers both believe a seat is OPEN when it isn't.
- In the Redis-Lua design, **there is exactly one writer at any instant** because Redis is single-threaded. So overselling is structurally impossible at the Redis layer.
- The danger is at the **PG-Redis boundary**: if reconciler sees Redis says SOLD and PG says OPEN, we trust Redis (Redis is online truth; PG is ledger, async behind).
- For belt-and-suspenders: PG enforces a UNIQUE constraint on `(event_id, seat_id, status='sold')` — if reconciliation ever tried to sell a seat already sold, the PG insert fails and we alert.
- We monitor `oversell_attempts` as a gauge metric. **Should always be zero.** Non-zero is a Sev-1.

---

### Deep dive 2: Virtual waiting room — the airlock

> **Interviewer:** "How does the queue work when 14 million people show up at the same instant?"

**Candidate:**

The waiting room exists for one reason: **decouple the rate of incoming users from the rate the inner system can handle.** Inventory Service can absorb 5K–10K admissions/min comfortably. We can't let 14M users hit it in 30 seconds. The queue is the buffer.

**Three things the queue must do:**

1. **Hold position durably** for users who showed up early.
2. **Admit at a controlled, operator-tunable rate.**
3. **Not melt down itself** under the same load it's protecting.

**Implementation: per-event Redis sorted set (ZSET).**

- Key: `queue:{event_id}` → ZSET of (user_id, score).
- Join queue: `ZADD queue:{event_id} {score} {user_id}`.
- Admit: a worker periodically `ZPOPMIN` the next batch and mints admit tokens.
- Position: `ZRANK queue:{event_id} {user_id}`.

**Score = randomized lottery, not strict timestamp** (more in §8.4).

**Why ZSET specifically:**
- O(log N) for insert and rank.
- Score-based ordering lets us swap policy (FIFO vs lottery) without changing the data structure.
- Atomic POPMIN.
- Per-event isolation: Coachella's queue is a different key from Taylor Swift's, so a slow Coachella admission doesn't block Taylor.

**Won't a single ZSET per event saturate?**

- Single Redis: 100K ops/sec. We have 14M users polling every 3s = 5M polls/sec. **This breaks a single Redis.**
- **Mitigation 1**: poll endpoint is mostly read, not write. We answer position from a *read replica* (`ZRANK` is read-only) or, better, from a position cache.
- **Mitigation 2**: **shard the queue itself.** Split `queue:{event_id}` into K shards, each with its own ZSET. User's `user_id` hashes to a shard. Each shard does its own POPMIN at 1/K the rate. Position numbering is approximate ("you are around #X"), which users tolerate.
- **Mitigation 3**: don't return exact position; return a **bucket**: "you are in bucket 4 of 100". Bucket is computed once at queue-join and cached. Polls just refetch admission-status, not recompute rank. Cuts ZRANK load to near zero.

I'd ship Mitigation 3 (bucketed position) plus a sharded ZSET for very large events.

**Admission policy.**
- A control-plane service (Admission Controller) issues admit tokens at rate R/min, where R is per-event configurable.
- Default: 10K admits/min for a hot event. Operator can dial up/down live in war-room.
- Each admit token is signed (HMAC over user_id + event_id + admit_time + ttl=15 min). Inventory Service rejects requests without a valid admit token.
- This means the **only way to reach Inventory Service is through the queue.** No back-door.

**Server-driven sale start (no client clocks).**
- Clients don't know the exact sale-start to the millisecond. They know "around 10:00 AM". The server admits the first batch out of the queue *whenever the server says*. This avoids a thundering herd at exactly 10:00:00.
- All inbound `queue/join` requests before T-0 get queued with future-dated scores. They become eligible for admission only at T-0+. This way, joining the queue 5 seconds early gives no advantage.

**Heartbeat / abandonment.**
- Users who close the tab should leave the queue. Client sends a heartbeat every 30s. If no heartbeat for 90s, queue entry is GC'd.
- Without this, dead tabs occupy queue slots indefinitely → ghost queue.

---

### Deep dive 3: Hold timeouts and consistent expiration across DCs

> **Interviewer:** "A user has held seats but is taking forever in checkout. When and how do those seats get released?"

**Candidate:**

**Policy.** Hold TTL is 7–10 min. Industry standard. Long enough for credit-card fumble, short enough that inventory churn is acceptable.

**Implementation options.**

**Option A — Redis TTL + keyspace notifications.**
- Hold key has TTL. When it expires, Redis emits a keyspace notification.
- A Hold Service consumer listens, reads the hold metadata, releases the seats via the inventory Lua script.
- Pro: precise expiry, low overhead.
- Con: keyspace notifications are fire-and-forget; if the consumer is down, you miss expirations. Need a backup sweep.

**Option B — TTL + periodic sweep.**
- Every 30 seconds, scan for holds where `expires_at < now`. Release.
- Pro: simple, deterministic, replayable.
- Con: up to 30s late on release (acceptable for a 10-min hold).

**Option C — At time of hold attempt, opportunistic release.**
- When a new hold attempt arrives for a seat that's HELD but expired, the Lua script lazily releases it and re-acquires.
- Pro: zero background process needed.
- Con: doesn't release seats that no one tries to claim again. Inventory looks "more sold" than it really is.

**My pick: B + C combined.** Background sweep at 30s cadence as the durable mechanism; opportunistic release on the hot path so freshly-needed seats become available immediately.

**Cross-DC consistency.**
- All hold state lives in the per-event Redis cluster, which is regionally pinned (more in §13.1). So expiration is per-region authoritative. Cross-region replication of hold state is **not** required because each event sells from a single primary cell.
- The clock used for `expires_at` is **server time**, not client time. Client-issued timestamps are explicitly ignored.
- NTP-synced server clocks are sufficient. We don't need TrueTime; we're not doing global linearizability across DCs for holds.

**Edge case: hold expires right as user clicks Checkout.**
- Race condition: Hold Service releases at T+7:00. User submits checkout at T+7:00.001.
- Resolution: Checkout Service does an atomic re-acquire via Lua: "for these seat_ids, if status == HELD by this user, transition to SOLD." If the seats have already been released, the script fails atomically and we return 410 Gone with "your hold expired, please retry."
- This means we never charge a user whose hold expired.

**Edge case: payment is in flight when hold expires.**
- Don't allow this. Before calling Stripe, **extend the hold** (set `expires_at = now + 5min`) or transition seats to a special `LOCKED_FOR_PAYMENT` state. This protects the user during the slow payment leg.
- The state machine: `OPEN → HELD → LOCKED_FOR_PAYMENT → SOLD` (success) or `→ OPEN` (failure / timeout).

---

### Deep dive 4: Fairness — FIFO, lottery, or tiered

> **Interviewer:** "How is it fair? The fastest connection wins?"

**Candidate:**

This is partly a UX/legal question, not just a technical one. Three policies in industry use:

**Policy 1 — Strict FIFO (first-click-wins).**
- Score = arrival timestamp.
- "Fairest" by the literal definition of first-come-first-served.
- **Problem**: arrival time is dominated by network latency, browser warmup, button-mashing. A user 50ms closer to the LB wins over a user with a flaky connection. People with worse connectivity systematically lose. Bots also benefit massively (low-latency colocation).
- **Verdict**: not perceived as fair, and is exploitable.

**Policy 2 — Uniform lottery.**
- Score = `random(0, 1)` at queue-join time.
- Joining 5 seconds early vs 5 seconds late doesn't matter — your score is random.
- Mathematically uniform across all entrants in the join window.
- **Problem**: users who clicked first feel cheated when they end up at position #800K while a user who joined 30 seconds later is at position #5K. Public perception suffers.
- **Verdict**: technically fair, optically unpopular.

**Policy 3 — Tiered presale (Verified Fan, etc.).**
- Pre-register days/weeks in advance. Verified Fan account gets a pre-issued lottery code that admits them into a **separate, smaller queue** before the public sale.
- The general public sale is a smaller residual.
- **Bot defense baked in**: getting Verified Fan status requires phone verification, account age, etc. Hard for bots to bulk-acquire.
- **Verdict**: best blend of fairness and bot defense. Standard for mega-events.

**My pick: tiered. Verified Fan presale (lottery within tier), then a public on-sale (lottery within public tier).**

Implementation:
- Tier is encoded in the `queue` ZSET score: `score = (tier_priority * 10^9) + random_within_tier`. So tier-1 always sorts before tier-2 regardless of randomness.
- This is one ZSET per event with implicit tier ordering, not a queue per tier.
- For purely public sales (no presale): single tier, uniform lottery.

**Justifying lottery over FIFO to the interviewer.**
- "If we're going to throttle admission to 10K/min anyway, the user with the 50ms-slower connection lands at position 1,234,567 instead of 1,200,000 — both are below the threshold of getting in at all. So FIFO doesn't actually help them. It just makes us *feel* like we're rewarding fast clickers, which is illusory at this scale and benefits bots most."

---

### Deep dive 5: Bot defense

> **Interviewer:** "At $5K/seat on the resale market, every bot operator on Earth is hitting your system. How do you stop them?"

**Candidate:**

This is a layered problem. Single defense is insufficient. The economic reality: bots will spend $1K to bypass us if the expected return is $10K. We make their EV negative.

**Layer 1 — WAF / edge (Cloudflare/Akamai).**
- IP reputation (residential vs colocation; known proxy nets).
- TLS fingerprinting (JA3/JA4): real browsers have specific TLS handshake signatures; HTTP libraries don't match.
- HTTP/2 fingerprinting (frame ordering).
- Rate limiting per IP, per ASN, per /24.
- Geographic limits (this concert is North America-only? block other continents at the edge).

**Layer 2 — Bot management.**
- Cloudflare Bot Management or Akamai Bot Manager scoring on every request.
- Behavioral signals: mouse movement, page-load timing, click cadence.
- Challenge response: invisible CAPTCHA → managed CAPTCHA (hCaptcha/reCAPTCHA) → forced interactive challenge.

**Layer 3 — Account-level constraints.**
- Account age: if you registered 5 minutes ago, you can't queue for the Taylor Swift presale.
- Phone-verified accounts.
- Limit one queue entry per account per event.
- Limit one device fingerprint per account per event.
- Card-on-file requirement: ties purchase to a real payment instrument.

**Layer 4 — Verified Fan.**
- Pre-registration with manual + automated review.
- Behavioral history: prior purchases, account engagement.
- Phone-verified, identity-tied.
- Outputs a signed presale code that's the only way into the presale tier.

**Layer 5 — Per-purchase limits.**
- Max 8 tickets per account per event.
- Max 1 transaction per credit card per event.
- Address verification (fraud signals from card processor).

**Layer 6 — Behavioral / ML detection during the sale.**
- Anomaly model on (time-to-click, mouse path, IP cluster, device fingerprint cluster).
- Realtime: high-confidence bot → silently shadow-route to a "tar pit" that returns slow, eventually-failing responses. Bot sees no signal that it's been detected.
- Post-sale: review high-purchase clusters, void orders, refund.

**Layer 7 — Honeypots.**
- Hidden form fields a real browser ignores; bots fill them. Easy detection.
- Fake seat IDs that don't exist; bots that try to claim them are flagged.

**Limitations to admit honestly.**
- Bots still get through. Industry data: 5–20% of high-demand on-sale traffic is bot, and a non-trivial fraction succeeds.
- The goal isn't 100% bot blocking. The goal is: **make bot-purchased tickets a small fraction; raise the bot's per-ticket cost above the resale spread**; provide tools for promoter-led void/refund post-sale.

---

### Deep dive 6: Read amplification before sale start

> **Interviewer:** "What about the millions of users hammering F5 on the event page in the 60 seconds before sale opens?"

**Candidate:**

Pre-sale page traffic is a **pure read storm on a static-ish artifact**. Solve this entirely at the edge.

**The event page becomes static + countdown.**
- HTML: 100% CDN-served. No origin hits.
- Cache key: `(event_id)` — no per-user variation on the public-facing page.
- TTL: 60s normally; **drop to 5s in the hour before sale start** so the "Buy" button can transition reliably from disabled → enabled.
- Stale-while-revalidate so the CDN keeps serving even if origin is slow.

**The seatmap is pre-baked.**
- Static SVG/JSON of the venue layout, served from CDN.
- Immutable per event (the venue layout doesn't change).
- Compressed, ~50–200 KB.

**Availability data is the only dynamic piece.**
- Pre-sale: every seat is OPEN. Hardcode `all_open` in the response. Don't even hit the inventory layer.
- Post-sale-start: per-section availability summary updated every few seconds, served via short-TTL CDN or a fast read-only API behind a CDN.

**The "Buy Tickets" button transition.**
- Before T-0: button is `disabled`, with a server-rendered countdown.
- The page polls a *small, fast endpoint* `GET /v1/events/{id}/sale-status` every 3–5 seconds in the last minute. This endpoint is also CDN-cached at 1–2s TTL.
- At T-0, the endpoint flips status, the CDN cache rolls over within 1–2s, button enables.

**Sale-start synchronization.**
- The flip is server-driven, not client-clock-driven.
- Avoids the case where some users' clocks are 30s off and they spam "Buy" early or late.
- Avoids the thundering herd at exactly T-0:00.000 — the cache rollover spreads the surge over 1–2 seconds organically.

**Capacity implication.**
- At peak, edge serves ~14M users × ~5 KB × 1/3s polls = ~23 GB/s. CDNs eat this without breaking a sweat.
- Origin serves only the cache fills: at 1s TTL × ~10 PoPs × tiny payload = a handful of QPS. Origin is calm.

---

### Deep dive 7: Payment integration and idempotency

> **Interviewer:** "Payment can fail or time out. How do you keep inventory consistent with what was actually charged?"

**Candidate:**

Payment is the slowest, least-reliable hop in the entire system. 3–5 second latency, ~1% nominal failure rate, occasional timeouts.

**State machine for an order:**

```
       hold_id created (Redis)
              │
              ▼
   ┌─→ pending (PG order row)
   │          │
   │          │ call Stripe(payment_intent)
   │          ▼
   │       ┌──── 200 success ────→ paid → seats SOLD → tickets issued
   │       │
   │       ├──── 4xx decline ────→ failed → seats released
   │       │
   │       └──── timeout / 5xx ──→ ??? (this is the hard case)
   │
   └─ retry-with-same-idempotency-key
```

**Timeout case (the hard one).**
- Stripe might have charged the card but our connection died. We don't know.
- Solution: **idempotency key per checkout attempt.** Pass the same key on retry. Stripe deduplicates server-side and returns the same result.
- If we still don't get an answer after retries: leave the order in `pending` with a flag, and let an async **reconciler** poll Stripe with the idempotency key periodically (every 30s, then exponential backoff up to 1 hour). When reconciler gets a definitive answer, finalize the order.
- Hold remains alive (or in `LOCKED_FOR_PAYMENT`) during reconciliation up to a max of, say, 30 min. After that, we accept the loss: cancel the order, refund if Stripe says it was charged.

**Success-but-seats-went-bad case.**
- Stripe says paid, but we crashed before transitioning seats to SOLD in Redis.
- On restart, the Reconciler reads `orders` rows in `paid` status and ensures seats are SOLD. If seats were re-released (somehow) and resold, we have an oversell — must refund and apologize. Monitor as `payment_succeeded_but_no_seats`. **Should always be zero.**

**Idempotency keys, end to end:**
- Client generates an idempotency key per user-intent (one click on Checkout). Sends it with every retry.
- Server stores `idempotency_key → order_id` mapping for 24h. Duplicate keys return the original result, not a new order.
- Pass the same key down to Stripe. Stripe also dedupes.

**Don't over-engineer the success path.**
- 99% of the time, payment succeeds and we transition cleanly.
- The 1% is what kills you in a flash sale: 50K orders × 1% = 500 needing reconciliation. Designed for, not panicked over.

---

## 9. Tradeoffs & alternatives

### Decision: Redis Lua for inventory hot path

| Option | Pros | Cons | When to pick |
|---|---|---|---|
| **PG row lock (pessimistic)** | Simple, ACID, familiar | Doesn't scale past ~5K writes/sec on a hot row | Small venues (< 5K seats) without flash demand |
| **PG optimistic (CAS)** | Scales better than pessimistic | Retry storms under heavy contention | Moderate-contention systems |
| **✅ Redis Lua atomic** | Single-threaded → naturally serial; 100K ops/sec; O(1) per attempt | Volatile; needs AOF + replica + reconciler | **Flash sales with high contention on a single venue** |
| **Sharded inventory (sub-sections)** | Reduces contention K-fold | Starvation: one user's shard sells out while others have seats | GA tickets (1 SKU); not assigned-seat |
| **DynamoDB conditional writes** | Managed, scales | Higher latency than Redis; per-write cost adds up at this scale | AWS-native team without Redis ops capacity |

Defense for Redis: "single-threading is a feature here, not a bug. We get strict serialization without coordination, multi-key atomicity via Lua, and 200× headroom on the successful-write rate. PG remains the durable ledger written async; reconciliation handles drift. The cost is operating Redis as a primary, which we mitigate with AOF + replica + a Kafka audit log."

### Decision: Per-event cell isolation

| Option | Pros | Cons | When to pick |
|---|---|---|---|
| **Single shared system for all events** | Simple, cost-efficient | A Taylor Swift on-sale outage takes down a Coachella sale concurrently | Smaller scale, no mega-events |
| **Per-region cells** | Compliance, latency, blast-radius | More infra, cross-region sync for cross-border events | Most production systems |
| **✅ Per-event cells for top events** | Maximum blast-radius isolation; pre-warmable | Manual cell-warming process; per-cell cost | **Mega-events** (top 1% of on-sales) |
| **Per-promoter cells** | Cost attribution | Some promoters have many events of varying sizes | If the promoter relationship is the contractual unit |

We do all three: per-region default; per-event for Taylor Swift class; shared regional cell for everything else. More in §13.

### Decision: Lottery vs FIFO admission

| Option | Pros | Cons | When to pick |
|---|---|---|---|
| **Strict FIFO** | "First-come" intuitive | Bots win; users with bad connections systematically lose | When the event is small enough that everyone gets in |
| **✅ Uniform lottery** | Mathematically fair within a tier | Optics: feels random | When demand >> supply |
| **Tiered presale + lottery** | Bot-resistant; fairer; rewards engaged fans | Requires Verified Fan infrastructure | **Mega-events** |

### Decision: Hold TTL = 8 minutes

- Industry standard 7–10 minutes.
- Shorter (3 min): higher inventory churn, more failed checkouts, more frustration.
- Longer (15 min): fewer failed checkouts, but inventory is locked up longer, fewer total tickets sold per hour for sold-out events (less of a concern when supply >> demand isn't the case).
- **Pick 8 min as default; per-event configurable.**

### Decision: Kafka for audit

- Alternative: write everything to PG synchronously. Kills hot-path latency.
- Alternative: write to Kinesis. Vendor lock; otherwise equivalent.
- Kafka gives us replayable, ordered, durable audit log with 30-day retention. Reconciler, fraud detection, analytics, and downstream notification all consume from it.

### Decision: Eventual consistency for seat-map UI; linearizable for the seat object

| Scenario | Consistency model | Mechanism |
|---|---|---|
| User browses seat map | Eventual (1–2s lag) | CDN + Redis read replica |
| User submits a hold | Linearizable on the seat row | Redis Lua script atomicity |
| User sees their own held seats turn yellow | Read-your-writes | Optimistic UI update + server confirmation |
| Order durability post-payment | Strong (ACID) | PG transaction |

The eventual consistency on the *map* is what makes the UI fast; the linearizability on the *hold attempt* is what prevents oversell. Both at the same time.

---

## 10. Defending architectural choices — interviewer attacks

> **Interviewer:** "Why not just use DynamoDB conditional writes for inventory? It's managed, it scales linearly."

**Candidate:** Three reasons. (1) Per-write latency: Dynamo conditional write is ~5–10ms; Redis Lua is ~0.1ms. At 50K contention attempts/sec we're spending an order of magnitude more on every retry. (2) Cost: conditional writes against a hot partition get throttled and cost more than provisioned, and our access pattern is exactly a hot partition (one venue's inventory). (3) Operational: we already need Redis for the queue and the seat-map cache. Adding Dynamo for inventory is a third datastore for a problem that fits one. Where Dynamo wins: if we wanted full multi-region active-active with low ops burden, Dynamo Global Tables would be more turnkey. We don't need that for inventory because we keep one event in one region.

> **Interviewer:** "Why is Redis OK as the source of truth for inventory? Isn't that scary?"

**Candidate:** Redis isn't the source of truth. **Kafka is.** Every inventory state transition is emitted as an event before the response. Redis is a fast cache of the *current* state. PG is the materialized ledger. If Redis is wiped, we replay from Kafka and rebuild Redis. The only real risk is the gap between a Lua script committing in Redis and the Kafka emit succeeding — we close that gap by emitting from inside Redis (Redis Streams) or doing a strict write-then-emit-or-rollback pattern, and we accept that during failover we may need to replay the last few seconds of Kafka to be safe.

> **Interviewer:** "Why is the queue a single ZSET? Won't that ZSET itself be a bottleneck?"

**Candidate:** It will be, on a single Redis instance, when we hit several million entries with millions of polls. So we shard it. The user_id hashes to one of K shards, each its own ZSET. K = 16–64 typical for a mega-event. Position numbering becomes approximate ("you are in bucket 4 of 100") rather than exact, which is what users see anyway. Admission is the round-robin POPMIN across shards. We lose strict global ordering (which we don't want — we want lottery anyway), and gain horizontal scalability.

> **Interviewer:** "What if the waiting room itself goes down?"

**Candidate:** That's an interesting failure mode because the queue is the airlock — without it, traffic floods inward. Our defense is layered: (1) the queue is the *only* path to a valid admit token, and Inventory Service rejects requests without one, so even if the WR layer is down, no one can buy. So an outage means "no sales for the duration", not "uncontrolled surge". (2) The queue is regionally redundant: per-event cells have HA Redis. (3) If the queue is genuinely lost (cluster wipe), we have a "queue replay" mode where users are admitted to a freshly-built queue in lottery order from a Kafka log of join events. This takes a few minutes and is a Sev-1, but it doesn't oversell.

> **Interviewer:** "Why per-event cells for the top events instead of just over-provisioning the shared system?"

**Candidate:** Three reasons. (1) Blast radius: if Taylor's sale meltdown affects Coachella's sale next door, that's two outages in a row. Per-event cells make events independent. (2) Pre-warming: dedicated capacity can be brought up *just for that event* (Redis pre-loaded with inventory; queue ZSETs pre-created; ML bot models tuned to the expected attack profile). Shared infra can't pre-warm without affecting others. (3) Cost: paying for 100× over-provisioning shared 24/7 is wasteful. Per-event cells are spun up T-2 hours, torn down T+24 hours. Pay only when needed. The cost: per-event cell warming runbook, deployment automation. Worth it for the top 1% of events; not worth it for the long tail.

> **Interviewer:** "Why not let any user click Buy directly without a queue? Just rate-limit the inventory service and 503 the rest."

**Candidate:** Two problems. (1) Without a queue, "503 the rest" means *random* users get in — fastest-network wins. The queue lets us implement deliberate fairness (lottery, tier). (2) 14M users 503-retrying every second is itself a self-DDoS — the retries hit the inventory service, which is exactly the load we're trying to throttle. The queue moves the buffering to a layer that's designed for it (a sorted set of pending entries) rather than a layer that's not (the inventory hot path).

---

## 11. Failure modes & mitigations

| Failure | Detection | Blast radius | Mitigation | Recovery |
|---|---|---|---|---|
| Redis (inventory) primary crashes | Sentinel/Cluster health; latency spike | All hold attempts for the affected event(s) 503 | Failover to AOF replica; ~10s outage during which clients see 503 | Replica promoted; clients retry; reconciler validates Redis ↔ Kafka |
| Inventory drift (Redis says SOLD, PG says OPEN) | Reconciler diff | Inventory accounting wrong | Trust Redis as online truth; PG catches up | Alert if drift > N seats; investigate manually |
| Payment timeout / Stripe outage | 5xx rate from Payment Service | Checkouts stuck in `pending` | Retry with idempotency key; reconciler polls Stripe async; hold extended to LOCKED_FOR_PAYMENT | Auto-reconciles; Sev-2 if Stripe extended outage |
| Bot bypass of waiting room | Per-user request rate; admit-token signature mismatches | Bots reach Inventory Service | Signed admit tokens (HMAC-validated); rate limit per user even with valid token; behavioral ML | Tighten admission rate; void offending orders post-sale |
| Queue ZSET shard hot/melted | Per-shard latency; memory pressure | Subset of users see slow queue updates | Shard count was sized for 2× expected; auto-rebalance into more shards if observed | Operator scale command; war-room |
| Hold-Service crashes (no expirations) | Hold queue depth; expired holds count growing | Inventory frozen in HELD state; new buyers see fewer seats | Periodic sweep is durable (option B); opportunistic release on hot path catches in-demand seats | Restart Hold-Service; sweep catches up |
| Sale-start clock skew | Synthetic monitor on `/sale-status` | Some users see Buy enabled early/late | Server-driven flip via cache rollover; ignore client clocks | None needed; CDN propagation drives this |
| CDN outage at sale-start | Edge error rate; origin spike | Pre-sale read storm hits origin | Multi-CDN failover (Cloudflare + Akamai + CloudFront); origin can absorb if WAITING_ROOM is intact | Failover to secondary CDN; queue still gates inventory |
| Per-event cell cross-talk (cell A bug taking down cell B) | Cross-cell error correlation | Should be zero by design | Cells are network-isolated; no shared mutable state; shared services are read-only | Postmortem any cross-cell incident; design review |
| Order DB primary failure (PG) | Write error rate, replication lag | Checkouts cannot finalize | Stream replication to standby; failover ~30s; in-flight orders retried by client | Auto-failover; reconciler validates orders post-recovery |
| Queue replay needed (queue cluster lost) | Operator-initiated | Sale paused for ~minutes | Replay from Kafka `queue.join` log; rebuild ZSETs; resume admission | Sev-1 runbook |
| Mass refund event (event cancelled) | Operator-initiated | All orders for event refunded | Async refund pipeline against Stripe; idempotent | Hours to clear; communicated proactively |
| Oversell detected (PG UNIQUE violation) | Alert | A user pays for a seat already sold | Should be impossible by design; if it happens, comp the latecomer with same-or-better seat or refund | **Sev-1, root-cause to design flaw** |

---

## 12. Productionization

### Pre-launch (every major on-sale)

- [ ] **Cell warming** at T-2 hours: Redis pre-loaded with inventory, queue ZSETs pre-created, downstream consumers scaled, ML bot models pre-fetched.
- [ ] **Load test at 10× expected**: in a staging cell with synthetic traffic. Even after passing, run scared.
- [ ] **Game day**: chaos-test killing inventory Redis primary mid-sale; verify failover and zero-oversell.
- [ ] **War room** staffed during the on-sale: oncall engineers from inventory, queue, payments, edge, plus comms/PR.
- [ ] **Kill switch** rehearsed: operator can disable the queue (block new joins), pause admission (stop POPMIN), force-release expired holds, or emergency-cancel the event.
- [ ] **Monitoring dashboard** preloaded with the right panels (admission rate, queue depth, hold attempts, oversell counter, payment success rate).
- [ ] **Communicate sale start** clearly: server-driven, not "10:00 AM exactly" — say "approximately 10:00 AM" so we control the actual flip.

### Rollout (system-level changes, not per-sale)

- [ ] Feature-flag any change to the inventory hot path. Roll to 1% of events first.
- [ ] Dark-launch new code paths for 2 weeks: shadow traffic, compare results.
- [ ] Per-region rollout: smallest market first (e.g., Australia) to limit blast radius if something is wrong.
- [ ] Auto-rollback if `oversell_attempts > 0` over 5 minutes.
- [ ] Per-region capacity check: each region must have headroom for its largest scheduled on-sale that month.

### Capacity

- [ ] **Pre-provisioned cells** for known mega-events (top 20 events of the year, planned 6 months out).
- [ ] **On-demand cell-spin-up** runbook for surprise demand (3–7 day lead time).
- [ ] Auto-scale Inventory Service on QPS, target 50% CPU (we want headroom for spike).
- [ ] Redis memory headroom 60% (needs room for hold metadata growth during peak).
- [ ] Kafka partition count = 5× current — repartitioning is expensive.

### Migration / replacement

- [ ] Dual-write to old + new inventory backends for 30 days.
- [ ] Reconciliation job comparing both; alert on any divergence.
- [ ] Cutover one event at a time, lowest-demand first.
- [ ] Old system stays read-only for 90 days post-cutover (rollback insurance).

### Cost

- Estimated cost per hot on-sale: dominated by **pre-warmed cell capacity that idles 99% of the time** if we keep cells permanently up.
- Mitigation: ephemeral cells, spun up T-2h, torn down T+24h. ~60% cost reduction on cold-spin-up.
- Bigger cost lever: **CDN egress on the pre-sale read storm.** Negotiate multi-CDN contracts; aggressive cache tuning.
- Per-DAU is misleading here — costs are dominated by the few mega-events, not the long tail.

### Compliance

- PCI-DSS for payment data (which is why Stripe/Braintree handle the card; we never see PAN).
- Data residency: EU on-sales must be served from EU cells; US from US cells; etc.
- Anti-scalping legislation in some jurisdictions (e.g., NY, ON): per-event purchase caps, identity verification.

### Post-event

- **Postmortem after every major on-sale**, regardless of whether anything broke. What was the queue depth? Admission rate? Bot-block rate? Any user-reported issue?
- Tune knobs for next event: admission rate, hold TTL, bot-defense aggression.
- Feed learnings back into runbook.

---

## 13. Monitoring & metrics

### Business KPIs

- Tickets sold per on-sale (vs supply)
- Time-to-sellout
- % of buyers who completed checkout vs abandoned
- Revenue per event
- Customer-reported issues per 1K transactions

### Service-level (SLIs)

- **Hot path SLO during on-sale**: 99.99% availability of `/holds` and `/checkout` (no more than 4 min downtime per major sale).
- **Hold latency**: p99 < 200ms.
- **Admission rate**: tracked vs target (e.g., 10K/min ± 5%).
- **End-to-end checkout**: p99 < 8s (payment dominates).
- **CDN hit rate on event page**: ≥ 99%.

### Component metrics

#### Waiting Room Service
- Queue depth per event (alert if ≥ 5M)
- Polls/sec per shard; alert on shard imbalance
- Admission rate (target vs actual)
- Heartbeat dropoff rate (ghost queue size)
- Admit-token mint rate

#### Inventory Service
- Hold attempts/sec, success rate
- Hold contention rate (409 responses / total)
- p99 Lua script execution time (alert if > 10ms)
- Redis memory utilization (alert at 75%)
- Reconciler drift size (alert if > N seats)
- **Oversell counter — must be 0**

#### Hold Service
- Active holds (current count)
- Hold expiration rate
- Hold-to-checkout conversion rate
- Sweep latency (lag from `expires_at` to actual release)

#### Checkout Service
- Checkout requests/sec
- Payment success rate (alert if drops below 95%)
- Payment p99 latency
- Reconciliation queue depth
- `payment_succeeded_but_no_seats` counter (must be 0)

#### Edge / WAF
- Bot-block rate
- CAPTCHA challenge rate
- Top blocked ASNs / IP ranges
- Per-region request rate

#### Kafka (audit)
- Producer error rate (alert if non-zero)
- Consumer lag (alert > 30s)
- Partition skew

#### PostgreSQL (orders)
- Write rate, p99 latency
- Replication lag
- Connection pool saturation
- Slow queries (any > 100ms during sale)

### Alert routing

- **Page (24/7)**: oversell > 0; payment failure rate > 5%; admission rate ≠ target ± 20% during a hot sale; Redis primary unreachable; queue depth growing without bound after sale start.
- **Ticket**: minor drift in reconciler; CDN hit rate below 95%; bot block rate spike.
- **Dashboard-only**: cost, throughput trends, capacity headroom.

### Dashboards

1. **On-sale war room** (single screen during a hot sale): admission rate, queue depth, hold success/fail, payment success, oversell counter, top-level error rate.
2. **Inventory health**: per-event Redis, reconciler drift, Lua p99, hold expiry queue depth.
3. **Waiting Room**: queue depth, admission rate per event, ghost queue.
4. **Edge / Bot**: blocks, CAPTCHA, top abusing networks.
5. **Business**: tickets sold, revenue, time-to-sellout.

---

## 14. Security & privacy

- All admit tokens are HMAC-signed (rotating key per event-day). Validating tokens at Inventory Service blocks queue bypass.
- Idempotency keys are hashed before storage (avoid leaking client-controlled values).
- Per-account limits enforced server-side; client-side limits are advisory.
- PCI-DSS: card data never touches our servers — Stripe/Braintree iframe takes the card; we get a `payment_method_token`.
- Credentials: secrets in Vault, rotated 90-day; signing keys rotated per major event.
- DDoS: WAF + per-IP rate-limits + geographic blocks for events with regional restrictions.
- PII: emails, addresses, phone numbers. Encrypted at rest. Access-logged. GDPR delete path.
- Audit logs for every admin action (operator changing admission rate, force-cancelling an order).

---

## 15. Cost analysis

Order-of-magnitude:

| Component | Rough cost driver | Optimization lever |
|---|---|---|
| Per-event cell capacity | Pre-warmed Redis + service compute, ~100× normal | Ephemeral cells (T-2h to T+24h); auto-tear-down |
| CDN egress (pre-sale storm) | Bytes served at edge | Aggressive caching; smallest-possible event-page payload |
| Stripe fees | % of transaction value | Negotiated rate at scale |
| PostgreSQL (orders) | Storage + replication | Cold-tier orders > 1 year old |
| Redis (inventory + queue) | RAM | Pre-allocate per-event capacity; release post-sale |
| Kafka | Disk + retention | 30-day audit retention enough; archive to S3 for longer |
| Bot-defense (Cloudflare/Akamai bot mgmt) | Per-request | High-confidence requests bypass deeper inspection |

**Dominant cost driver**: spike capacity. We must provision for 100× normal during big sales. Pre-provisioning 24/7 is wasteful; on-demand scaling is too slow (Redis cold-load takes minutes; queue must exist *before* user clicks). Compromise: dedicated cells for top-tier events spun up T-2h, plus auto-scaled shared cell for the long tail.

**Cost per ticket**: ranges from $0.05 for a long-tail event to $5+ for a Taylor Swift–class event when you amortize the pre-warmed cell, multi-CDN, and war-room labor. Charged back to the promoter as part of the platform fee.

---

## 16. Open questions / what I'd validate with PM

1. What are the contractual SLAs with promoters for hot on-sales? (Drives our SLO targets and per-cell investment.)
2. Anti-scalping legal regime per jurisdiction — we have to enforce these (NY, Ontario, parts of EU).
3. Is the Verified Fan registration system part of this design, or upstream? (Affects bot-defense surface.)
4. What's the policy on extending hold time during slow payment legs vs aborting? (Affects user experience and conversion.)
5. Is multi-event purchase in one cart in scope? (E.g., a fan buying both Friday and Saturday shows simultaneously — affects state machine.)
6. Refund / cancellation policy: who initiates, what's the SLA, do we void seats back to inventory or write off? (Affects reconciliation paths.)

---

## 17. Staff-level scorecard for this problem

| Signal | Did the candidate... |
|---|---|
| ✅ Identified inventory contention as the central insight | Within first 10 min, not as an afterthought |
| ✅ Quantified the read amplification | "14M:50K = 280:1" calculation made explicit |
| ✅ Picked Redis + Lua with a defended reason | Not "because it's fast"; because of single-threaded atomicity |
| ✅ Designed the waiting room as the airlock, not an afterthought | Explained throughput-decoupling rationale; noted the queue itself is a scale problem |
| ✅ Volunteered fairness as a real design dimension | FIFO vs lottery vs tiered; chose tiered + lottery with reasoning |
| ✅ Designed for zero-oversell, not "low-oversell" | Linearizable on the seat object; structural impossibility argument |
| ✅ Discussed bot defense as a layered system, not "we'd add a CAPTCHA" | Six layers minimum; admitted limits |
| ✅ Talked about cost in concrete terms | Pre-warmed cell idle cost, CDN egress, ephemeral cells |
| ✅ Volunteered failure modes before being asked | Redis crash, payment timeout, queue meltdown, oversell as Sev-1 |
| ✅ Designed for graceful degradation | Failover behavior, Kafka audit replay, reconciler |
| ✅ Closed with a wrap-up speech | Restated dominant constraints and the chosen levers |

### What separates Staff from Senior on *this* problem

A Senior candidate will:
- Correctly identify that you need a queue
- Pick Redis for inventory
- Mention CAPTCHA for bots

A Staff candidate will additionally:
- Recognize the queue *itself* is a hot-key scaling problem and shard it
- Articulate **why** Redis single-threading is the win (not just "Redis is fast")
- Make the admission rate **operator-tunable live** during the sale
- Treat fairness as a design dimension with explicit tradeoffs (lottery vs FIFO vs tiered)
- Identify per-event cell isolation as the answer to spike-capacity-vs-cost, not generic auto-scaling
- Volunteer the reconciliation story for Redis ↔ PG drift, not hand-wave consistency
- Discuss the payment-timeout idempotency story end-to-end, including the "stuck pending" reconciler
- Bring up server-driven sale-start to defeat client-clock skew, before being asked

---

## 18. Wrap-up speech (template)

> "To wrap up: the dominant constraints here are extreme read amplification at sale-start (280:1, 14M users for 50K seats) and high-contention writes on a single venue's inventory inside a 10-minute window. The architecture answers both: a virtual waiting room at the front (sharded Redis ZSET, lottery + tiered admission, server-driven sale-start) acts as the airlock that throttles inbound traffic to a rate the inner system can handle; the inner inventory layer is Redis with Lua scripts for atomic multi-seat holds, with PostgreSQL as the durable order ledger and Kafka as the audit log driving reconciliation. The two pieces I'd worry about in production are (1) Redis as the hot path — mitigated by AOF + replica + per-event cells with a Kafka replay path, and (2) bots — mitigated by a six-layer defense from edge fingerprinting through Verified Fan, with the explicit acknowledgment that we won't hit zero. Productionization: per-event cell warming for top-tier events, war-room staffing during on-sale, kill switches for queue and admission, postmortem after every major sale. SLO-alerted on `oversell_attempts == 0` (the only bright-line metric), admission-rate-vs-target, and payment success rate. If we had another 30 minutes, I'd deep-dive (a) the reconciliation pipeline between Redis, PG, and Kafka, or (b) cross-cell coordination for events where one venue is replicated across regions."

---

## 19. Cellular / per-event isolation — the high-leverage architectural story

> Bring this up explicitly — it's the difference between "Senior who designed Ticketmaster" and "Staff who has run mega-events". Frame it as: *"The architecture above is necessary but not sufficient. The next layer is cellular isolation at two granularities — per-region for compliance and blast-radius, and per-event for hot on-sales."*

### What & why

Two cell granularities:

1. **Per-region cell** — like the news-feed model: US, EU, India, etc. Same drivers (compliance, data residency, blast radius, latency).
2. **Per-event cell** — *new in this domain*. For a Taylor Swift class on-sale, we spin up a **dedicated stack** for that event: its own Redis (inventory + queue), its own Inventory Service deployment, its own Hold Service, its own Kafka topics. Smaller events share a regional cell.

Drivers for **per-event cells** specifically:

1. **Blast-radius isolation** — a Taylor Swift on-sale meltdown does not affect the simultaneous Coachella sale. Cell-level outages are 1 event, not all events.
2. **Pre-warming capability** — we know months in advance that Taylor Swift on-sale is on date X. We can spin up a dedicated cell on date X-1, pre-load Redis, pre-create queue ZSETs, pre-fetch ML bot models, run final load tests. **Pre-warming is impossible on shared infra without affecting other events.**
3. **Tail-latency isolation** — a hot event's contention doesn't hit the Lua-script latency for unrelated events sharing the same Redis.
4. **Capacity attribution** — promoter pays for the cell; cost is direct, not amortized.
5. **Different ML models** — bot defense for a mega-event might use a stricter model than a club show. Per-cell config.

### Architecture

```
        ┌──────────────── Global Edge ─────────────────────┐
        │  CDN + Anycast + GeoDNS + WAF + Bot Mgmt         │
        └────────┬───────────────┬─────────────────┬───────┘
                 │               │                 │
       ┌─────────▼─────────┐ ┌───▼────────────┐ ┌─▼───────────────────┐
       │  Per-Event Cell   │ │  Per-Event     │ │  Regional Shared    │
       │  (Taylor Swift)   │ │  Cell (Super   │ │  Cell (US-East —    │
       │                   │ │  Bowl)         │ │  long tail of       │
       │ ┌───────────────┐ │ │ ┌────────────┐ │ │  smaller events)    │
       │ │ WaitingRoom   │ │ │ │WaitingRoom │ │ │                     │
       │ │ Inventory     │ │ │ │Inventory   │ │ │ ┌─────────────────┐ │
       │ │ Hold          │ │ │ │Hold        │ │ │ │ WaitingRoom Svc │ │
       │ │ Checkout      │ │ │ │Checkout    │ │ │ │ Inventory Svc   │ │
       │ │ Redis (inv,   │ │ │ │Redis       │ │ │ │ Hold Svc        │ │
       │ │   queue)      │ │ │ │Kafka       │ │ │ │ Checkout Svc    │ │
       │ │ Kafka         │ │ │ └────────────┘ │ │ │ Redis cluster   │ │
       │ └───────────────┘ │ └────────────────┘ │ │ Kafka           │ │
       └─────────┬─────────┘                    │ └─────────────────┘ │
                 │                              └──────────┬──────────┘
                 │                                         │
                 └──────────────────┬──────────────────────┘
                                    │
                       ┌────────────▼─────────────┐
                       │   Shared Services         │
                       │   - Catalog Svc (read)    │
                       │   - Orders DB (PG, sharded│
                       │     by event_id; orders   │
                       │     can stay in shared    │
                       │     ledger because they   │
                       │     are write-once-       │
                       │     append after payment) │
                       │   - Payment Svc           │
                       │   - User Directory        │
                       │   - Audit / Reconciler    │
                       └───────────────────────────┘
```

### Routing

- DNS / API Gateway has a per-event routing table: `event_id → cell_id`.
- Routing table entries written by an "Event Onboarding" workflow when a promoter schedules a hot event.
- Default: events route to the regional shared cell. Promoter-or-platform-flagged hot events route to a dedicated cell.

### Cell lifecycle for a per-event cell

```
T-30 days:  Event flagged as hot. Cell capacity provisioned.
T-7 days:   Cell deployed. Synthetic load test on the cell.
T-2 hours:  Inventory Loader runs PG → Redis warm-up. Queue ZSETs pre-created.
            ML bot models loaded. War room staffed.
T-0:        Sale starts. Cell handles the spike.
T+sale-end: Cell continues serving any in-flight checkouts and refunds.
T+24 hours: Cell drained, retired. State persisted to shared services for analytics.
```

### Cross-cell interactions

Mostly absent, by design:
- Orders go to the shared Orders DB (sharded by event, but one centralized table family). Per-cell Kafka writes orders to the shared DB via a consumer. Per-cell does not own its own orders ledger.
- Catalog (event metadata) is shared — read-only for cells.
- Payment Service is shared — cells call into it.
- Reconciler is global — it consumes Kafka audit topics from all cells.

### Per-region cells — the boring layer

- US, EU, IN, etc. cells for compliance and latency.
- A per-event cell lives **inside** one region cell.
- Cross-region: a global event (international tour, multiple venues across regions) is replicated *catalog-only*; each venue's on-sale is its own cell in its own region. Users buy in their home cell.

### When NOT to do per-event cells

- Pre-scale: under 10 hot on-sales/year, the operational overhead of per-event cells exceeds benefit.
- Long-tail events (most): too small to warrant. Shared regional cell is fine.
- Unpredictable spikes: by definition we can't pre-warm a cell for an event we didn't know was hot. Mitigate with a "surprise hot" runbook: shared cell can be force-failed-over to a dedicated cell on 30-min notice if traffic indicates.

### What "Staff signal" sounds like here

> "The on-sale path is not just a service — it's a *deployment unit*. Hot events get their own cell, pre-warmed two hours before sale-start with inventory loaded, queue created, bot models tuned to expected attack profile. The regional shared cell handles the long tail. The orders ledger and payment integration stay shared because they're write-once-append and can absorb the multiplexed write rate. The cost is real — per-event cell warming costs roughly $X per event of dedicated capacity that idles 99% of the time — but it's the only architecture that survives a Taylor Swift on-sale and a Coachella on-sale on the same day without one cannibalizing the other. The hard operational question isn't 'can we build cells'; it's 'who owns the per-event cell warming runbook, and what's the SLA for spinning up a surprise hot event'."

