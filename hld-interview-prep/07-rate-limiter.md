# 07 — Design a Distributed Rate Limiter

> Infrastructure problem with a small feature surface. Staff signal here is depth on algorithm tradeoffs, distributed counter consistency, hot-key handling, and the *fail-open vs fail-closed* decision. The interviewer doesn't care if you can recite token-bucket pseudocode — they care whether you can defend why you'd use it over sliding-window-counter at 10M QPS, and what happens when Redis dies.

---

## 0. Problem statement (as the interviewer would drop it)

> **Interviewer:** "Design a distributed rate limiter that sits in front of a large set of APIs. It needs to enforce per-user, per-API, and per-tenant limits. Walk me through the architecture."

That's it. The problem is small in surface but deep in tradeoffs. Staff candidates don't rush to "Redis with INCR" — they scope the algorithm choice, the consistency model, and the failure behavior first.

---

## 1. Clarifying questions

Pick 5–6. The right ones here cluster around: **what's a "limit", what does the caller experience on a deny, and what's the latency budget**.

> **Candidate:** "Before I draw anything, a few questions to scope this."
>
> 1. **"What's the QPS through the limiter, and how many distinct rate-limit keys?"** — *Expect: 10M+ QPS aggregate (it sits in front of every API call), 100M+ keys (per-user × per-API × per-tenant combinations).*
> 2. **"What's the latency budget the limiter is allowed to add to a request?"** — *Expect: < 5ms p99. A limiter that adds 10ms to every request has just doubled the latency of half the APIs it protects.*
> 3. **"On a Redis or backend outage, do we fail open (allow the request) or fail closed (reject)?"** — *This is THE question. Almost always fail-open for user-facing APIs; fail-closed only for billing-critical paths. Force the answer.*
> 4. **"What's the granularity of the limits — per-second, per-minute, per-hour, per-day? All of the above?"** — *Drives algorithm choice. Sliding windows are fine for short windows; long windows (per-day) require more memory.*
> 5. **"Are limits global across regions, or per-region/per-cell?"** — *Massive design impact. Cross-region counter sync is expensive. Most production systems run per-cell counters and accept ~5–10% over-allowance globally.*
> 6. **"Is exact accuracy required, or is ~5% over-allowance acceptable?"** — *Drives the strong-vs-eventual consistency call. Almost always eventual is fine; exact accuracy is a feature flag for billing/quota-critical paths.*
> 7. **"Who configures limits, and how often do they change?"** — *Drives the config plane. Static limits = simpler. Dynamic limits per-tenant = needs a control plane.*
>
> **Out of scope (state explicitly):**
> - L3/L4 DDoS mitigation (delegate to edge — Cloudflare, AWS Shield)
> - WAF rules / pattern-based blocking
> - Captcha / bot detection
> - Authentication / API key issuance (assumed done upstream)
> - Billing / usage metering (related, but separate system)
>
> **Candidate:** "I'll design the synchronous in-line check that an API gateway calls, the distributed counter store, and the algorithm. I'll skip DDoS and WAF — those belong at the edge layer above this. Sound right?"

> **Interviewer:** "Yes."

---

## 2. Functional requirements

| Priority | Requirement |
|---|---|
| P0 | Given a rate-limit key, return ALLOW or DENY in <5ms p99 |
| P0 | Support multiple limit dimensions: per-user, per-API, per-tenant, per-IP, and composite keys |
| P0 | Support multiple time windows on the same key (e.g., 100/sec AND 10K/hour AND 1M/day) |
| P0 | Allow bursts up to a configured burst size (don't reject the first request after a quiet period) |
| P0 | Configurable limits per-key — control plane to update without redeploys |
| P1 | "Why was I rate limited?" explain endpoint returning the offending key + window + current/max counts |
| P1 | Shadow mode: count and report but do not actually reject (for productionizing new limits) |
| P1 | Per-tenant kill switch (override: always allow, always deny, or use limit) |
| P2 | Cross-region quota aggregation for global limits (eventually consistent) |
| Out | DDoS mitigation, WAF, captcha, billing metering |

---

## 3. Non-functional requirements

| Dimension | Target | Justification |
|---|---|---|
| Availability | 99.99%+ for the limiter; system fails OPEN on limiter outage | The limiter must not become a SPOF for the APIs it protects |
| Latency added | p99 < 5ms, p50 < 1ms | A 5ms-slow limiter on a 50ms API doubles the experienced tail |
| Throughput | 10M+ QPS aggregate, scalable horizontally | Sits in front of every API call |
| Accuracy | ~5% over-allowance acceptable; ~0% under-allowance (don't deny legit traffic) | Over-allowance is a small spend; under-allowance breaks the product |
| Key cardinality | 100M+ active keys | Per-user × per-API × per-tenant combinations |
| Consistency | Eventual across regions; per-region best-effort strong | Global exact-match counters are infeasible at this scale and not needed |
| Durability | Counters are ephemeral — losing them = brief over-allowance, not data loss | Tradeoff: durable Redis is slower; we don't need durability |
| Geography | Per-cell by default; cross-cell aggregation for global quotas | Cellular isolation, market data residency, blast radius |

---

## 4. Capacity estimation

> **Candidate:** "Let me work through the numbers. The interesting outputs are: how big is Redis, how many shards, and what's the bandwidth between gateway and limiter."

### QPS
- 10M QPS through the limiter (every API call hits it once minimum)
- A single API call may consult multiple limit keys (user + tenant + API = 3 lookups). Realistic load: 30M check-and-increment ops/sec on the counter store.
- Peak (3×): ~90M counter ops/sec.

### Keys
- 100M active rate-limit keys
- Each key: ~50 bytes (counter value + window timestamp + TTL metadata) = **~5 GB** of working set. Trivially fits in a small Redis cluster — the bottleneck is **ops/sec**, not size.

### Per-Redis-node capacity
- Single Redis instance: ~100K ops/sec sustained for atomic Lua-script ops (lower for complex Lua).
- 90M ops/sec ÷ 100K/node = **~900 nodes** if every op hits Redis.
- **This is the punchline of the estimation:** unless we cut Redis QPS, we're in the absurd-cost regime. Two levers:
  - Local approximation per gateway host (sloppy counter), syncing every 100ms. Cuts Redis QPS by 10–100×. Brings node count to 50–100.
  - Coalesce multi-key checks into one Lua script (3 lookups = 1 round trip). Saves 3× round trips.

### Latency budget (5ms total)
| Hop | Budget |
|---|---|
| Gateway → limiter sidecar (UDP/IPC) | < 0.5 ms |
| Sidecar → Redis (intra-DC RTT) | ~0.5 ms (one round trip) |
| Redis Lua script execution | < 1 ms |
| Sidecar → gateway response | < 0.5 ms |
| Margin / GC / scheduler jitter | ~2 ms |

If Redis is across regions, that's already 80–150ms RTT. **The limiter MUST be co-located with the gateway it protects.**

### Bandwidth
- 30M ops/sec × ~200 bytes (request + response wire) = ~6 GB/s
- Sharded across 50–100 limiter hosts: ~60–120 MB/s/host. Trivial.

### Storage (counter retention)
- 100M keys × 50 bytes × replication 2× = **~10 GB** for current state
- Plus 24h history for "explain" endpoints + audit: 100M × ~5 events/day × 100 bytes = **~50 GB/day** for audit logs. Lives in cold-tier object storage, not in Redis.

---

## 5. API design

> **Candidate:** "There are two surfaces here: the data plane (gateway → limiter, called millions of times per second) and the control plane (operator → config, called rarely)."

### Data plane (gRPC; latency-critical)

```protobuf
service RateLimiter {
  // Synchronous check + decrement. Returns ALLOW / DENY.
  rpc Check(CheckRequest) returns (CheckResponse);

  // Async report-only (post-hoc penalty for systems that can't block).
  rpc Report(ReportRequest) returns (Ack);
}

message CheckRequest {
  // List of dimension descriptors to check together.
  // Allows one round trip for "user + tenant + API" composite limits.
  repeated Descriptor descriptors = 1;
  uint32 hits = 2; // usually 1; can be N for batched calls
}

message Descriptor {
  // Hierarchical key: [("tenant", "acme"), ("api", "POST /orders"), ("user", "u123")]
  repeated KV entries = 1;
}

message CheckResponse {
  Status overall = 1; // ALLOW | OVER_LIMIT | UNKNOWN (fail-open path)
  repeated DescriptorStatus per_descriptor = 2;
}

message DescriptorStatus {
  Status status = 1;
  uint32 limit = 2;
  uint32 current = 3;
  uint32 remaining = 4;
  uint32 retry_after_ms = 5; // hint for clients; goes into 429 header
}
```

**Why one batched call and not N parallel calls?** N parallel calls = N round trips of network jitter. One Lua script doing N checks atomically = one RTT. Saves 3× latency and ensures all-or-nothing semantics (don't decrement user counter if tenant counter denies).

### Control plane (REST; consistency-critical, low-QPS)

```
GET    /v1/limits                      → list all configured limits
PUT    /v1/limits/{descriptor_match}   → upsert a limit rule
DELETE /v1/limits/{descriptor_match}   → remove
GET    /v1/explain?descriptor={...}    → why was this denied? returns last N events
PUT    /v1/overrides/{tenant_id}       → kill switch: { mode: ALLOW_ALL | DENY_ALL | NORMAL }
PUT    /v1/limits/{id}/shadow          → toggle shadow mode (count, don't deny)
```

### HTTP semantics for callers (gateway → upstream)

When the limiter says DENY, the gateway returns:
```
HTTP 429 Too Many Requests
Retry-After: 5
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 1730000000
X-RateLimit-Bucket: tenant=acme;api=POST /orders
```

The `X-RateLimit-Bucket` header is **the** debugging affordance. Without it, customers paged at 3am don't know which limit they tripped.

---

## 6. Data model

The rate limiter has very little state. What it has, lives in three places:

### 6.1 Counter store (Redis cluster — the hot path)

Per-key data structure depends on the algorithm. For **token bucket** (our choice — see deep dive 1):

```
key   = "rl:tenant=acme:api=POST_/orders:user=u123"
value = HASH {
  tokens:    93.4         (float — fractional tokens for sub-second refill)
  last_ts:   1730000000123 (ms epoch of last refill)
}
TTL = max_window_seconds × 2  (e.g., 2 hours for an hourly limit)
```

For **sliding window counter**:
```
key   = "rl:tenant=acme:api=POST_/orders:user=u123"
value = HASH {
  curr_window_count: 42
  curr_window_start: 1730000000000
  prev_window_count: 87
}
TTL = 2 × window_size
```

### 6.2 Limit rules (config plane — Postgres / etcd)

```
table: rate_limit_rules
| id  | descriptor_match               | algorithm     | limit | window_sec | burst | priority |
|-----|--------------------------------|---------------|-------|------------|-------|----------|
| 1   | {tenant="acme",api="*"}        | token_bucket  | 1000  | 1          | 1500  | 100      |
| 2   | {api="POST /orders"}           | sliding_count | 10000 | 60         | -     | 50       |
| 3   | {user="*"}                     | token_bucket  | 100   | 1          | 200   | 10       |
```

Rules are a **set** of patterns; a request may match multiple. Highest-priority match wins per-dimension; aggregated across dimensions, ALL must allow.

### 6.3 Audit / explain log (Kafka → cold storage)

Every DENY event is sampled (1% by default, 100% for high-value tenants) and written to Kafka. Hot tail (last 1 hour) goes to a per-tenant Redis stream, queryable by the `/v1/explain` endpoint. Long tail goes to S3/Snowflake for analysis.

```
{
  ts: 1730000000123,
  tenant: "acme",
  user: "u123",
  api: "POST /orders",
  matched_rule: 1,
  algorithm: "token_bucket",
  current: 1500,
  limit: 1000,
  decision: "DENY",
  retry_after_ms: 200
}
```

---

## 7. High-level architecture

```
                   ┌─────────────────────────────────────────────┐
                   │              Edge / CDN / WAF               │
                   │   (DDoS L3/L4, IP reputation — out of scope)│
                   └────────────────────┬────────────────────────┘
                                        │
                              ┌─────────▼──────────┐
                              │  API Gateway       │
                              │  (Envoy / nginx)   │
                              └────────┬───────────┘
                                       │ rate-limit filter
                                       │ (gRPC, ~0.5ms RTT)
                              ┌────────▼───────────┐
                              │  Limiter Sidecar   │   ← co-located, same host or pod
                              │  - local cache     │     (sloppy counter — fast path)
                              │  - sync coordinator│
                              └────────┬───────────┘
                                       │ on cache miss / sync
                                       │ (or always, for strict mode)
                            ┌──────────▼────────────┐
                            │   Redis Cluster       │   ← sharded, hash-slot routed
                            │   (counters, Lua      │     consistent hashing → stable
                            │    script storage)    │     resharding behavior
                            └──────────┬────────────┘
                                       │
                ┌──────────────────────┼──────────────────────┐
                │                      │                      │
                ▼                      ▼                      ▼
      ┌───────────────────┐  ┌───────────────────┐  ┌────────────────┐
      │  Control Plane    │  │  Audit Pipeline   │  │ Cross-Cell     │
      │  - Limit rules    │  │  - Kafka topic    │  │ Aggregator     │
      │    (Postgres)     │  │  - Sampled deny   │  │ (eventual,     │
      │  - Rule push to   │  │    events         │  │  ~10s lag, for │
      │    sidecars (etcd)│  │  - S3 archive     │  │  global quotas)│
      │  - /explain API   │  │  - Hot-key alarm  │  └────────────────┘
      └───────────────────┘  └───────────────────┘

                ┌──────────────────────────────────┐
                │  Hot-Key Detector (background)   │
                │  - Top-N keys by req rate        │
                │  - Auto-shards a hot key by      │
                │    appending a random suffix     │
                └──────────────────────────────────┘
```

### Request walkthrough — the synchronous check path

1. Client request arrives at API Gateway.
2. Gateway calls the **limiter sidecar** on the same pod via UDP or unix-domain socket (~50µs overhead).
3. Sidecar:
   - **Fast path:** consults its in-memory sloppy-counter for the key.
     - If clearly under limit (>10% headroom): ALLOW immediately. No Redis call.
     - If clearly over limit (locally tracked over): DENY immediately. No Redis call.
     - If borderline: fall through to Redis.
   - **Slow path:** issues a Lua script to Redis with the descriptors. Atomic check-and-decrement returns ALLOW/DENY + remaining.
4. Sidecar returns to gateway.
5. Gateway either forwards request upstream (ALLOW) or returns 429 (DENY).

### Why this layering

- **Sidecar (not separate fleet):** zero network hop for the common case. ~95% of requests should be served from local cache, bypassing Redis entirely.
- **Lua scripts in Redis:** atomic check-and-decrement. Without atomicity, two concurrent decrements on the same key both succeed at the limit boundary → over-allowance. Redis Lua execution is single-threaded per shard, gives us atomicity for free.
- **Per-cell Redis cluster (not global):** counter sync across regions is too expensive for hot path. Per-cell counters + async aggregation = "good enough" for ~95% of limits.
- **Control plane separate:** rule changes are infrequent (~10s/day) and consistency-critical. Don't put them on the hot path.

### Where the latency comes from

In the common case (sloppy counter says clearly under limit):
```
Gateway → Sidecar (IPC):       ~50 µs
Sidecar local check:           ~10 µs
Sidecar → Gateway:             ~50 µs
Total:                         ~110 µs (well under budget)
```

In the slow case (sidecar consults Redis):
```
Gateway → Sidecar:             ~50 µs
Sidecar local check:           ~10 µs
Sidecar → Redis (intra-DC):    ~500 µs
Redis Lua execution:           ~200 µs
Redis → Sidecar:               ~500 µs
Sidecar → Gateway:             ~50 µs
Total:                         ~1.3 ms (still well under 5ms)
```

p99 inflated by GC / scheduler / network jitter ≈ 2–3 ms.

---

## 8. Deep dives

### Deep dive 1: Algorithm selection

> **Interviewer:** "Walk me through the algorithm options and pick one."

**Candidate:**

There are five canonical algorithms. I'll compare against the requirements: 10M QPS, ~5% over-allowance acceptable, must handle bursts, must be cheap in memory at 100M keys.

#### Fixed window
```
At each second boundary, reset counter.
counter[key, current_second]++
if counter > limit: DENY
```
- **Pros:** dead simple, O(1) memory per key.
- **Cons:** **boundary spike problem**. If limit = 100/sec and a client sends 100 at t=0.99s and 100 at t=1.01s, that's 200 requests in 0.02s — 2× the intended limit. Real attackers exploit this.
- **When to use:** never for adversarial traffic. OK for soft limits where 2× burst is acceptable.

#### Sliding window log
```
For each request, append timestamp to a sorted set.
On each request, drop entries older than window, count remaining.
```
- **Pros:** exact, no boundary problem.
- **Cons:** O(N) memory where N = requests in window. For a 10K/min limit that means 10K timestamps per key. At 100M keys × 10K × 8 bytes = **8 TB**. Killer.
- **When to use:** small N (e.g., per-IP limits with low cap). Not for our scale.

#### Sliding window counter (approximation)
```
Keep counts for current window AND previous window.
Estimated rate = curr_count + prev_count × (1 - elapsed/window)
if estimated > limit: DENY
```
- **Pros:** O(1) memory per key (3 fields). Smooths the boundary spike. ~5% inaccuracy under uniform traffic, more under bursty.
- **Cons:** approximation; real rate can deviate up to ~10% from estimated under non-uniform traffic.
- **When to use:** **default for high-cardinality limits where memory matters**. ~10× cheaper than log.

#### Leaky bucket (queue model)
```
Requests enter a FIFO queue of fixed capacity.
Queue drains at rate R requests/sec.
If queue is full, DENY.
```
- **Pros:** smooths output traffic to a constant rate. Good when downstream has a hard rate cap (e.g., third-party API).
- **Cons:** doesn't allow bursts even when there's headroom. A user idle for an hour can't suddenly send 100 requests at once.
- **When to use:** traffic shaping for downstream services that cannot tolerate bursts.

#### Token bucket
```
Bucket has capacity B, refilled at rate R tokens/sec.
Each request consumes 1 token. If bucket has 0 tokens, DENY.
On request:
  tokens = min(B, tokens + (now - last_refill) * R)
  last_refill = now
  if tokens >= 1: tokens -= 1; ALLOW
  else: DENY
```
- **Pros:** O(1) memory (2 fields). Allows bursts up to B. Simple, well-understood. **The industry default.**
- **Cons:** burst sized B can hit downstreams hard if B is large.
- **When to use:** **default for user-facing APIs**. Burst handling matches user expectations (idle then click → works).

#### Comparison table

| Algorithm | Memory/key | Accuracy | Burst handling | Boundary problem? | Best for |
|---|---|---|---|---|---|
| Fixed window | O(1), 2 fields | Bad at boundaries | Fixed | Yes (2× spike) | Soft limits |
| Sliding window log | O(N), N=req/window | Exact | Configurable | No | Low-cardinality, low-cap |
| Sliding window counter | O(1), 3 fields | ±5% | Smoothed | No | High-cardinality default |
| Leaky bucket | O(1) (queue ptrs) | Exact | None | No | Downstream traffic shaping |
| Token bucket | O(1), 2 fields | Exact (in-window) | Configurable burst | No | **User-facing APIs (default)** |

#### My pick

**Token bucket** for user-facing limits (allows bursts, matches user mental model: "I haven't called in a while, now I can").

**Sliding window counter** for very-high-cardinality short-window limits (per-IP, where 100M+ keys × 3 fields needs to be tight).

**Leaky bucket** specifically when shaping outbound calls to a third-party API with a hard ceiling.

I'd let the rule config specify the algorithm per-rule.

> **Interviewer:** "Why not sliding window log? You said it's exact."

**Candidate:** Memory. We have 100M+ active keys. A 10K/min limit means 10K timestamps per key (each 8 bytes) = 80KB per key × 100M keys = **8 TB of Redis**. We'd need 200+ Redis nodes just for memory, before even considering ops/sec. Token bucket needs 2 fields per key (~50 bytes incl. overhead) = **5 GB**. Three orders of magnitude difference. The exactness gain is not worth 1000× the cost.

> **Interviewer:** "What about clock skew?"

**Candidate:** Token bucket is sensitive to wall-clock drift. If sidecar A and sidecar B have clocks 100ms apart and both touch the same key, refill calculations diverge → small over-allowance. Two mitigations:
1. Use the **Redis server clock** as the single source of truth for `last_refill` — Lua script reads `redis.call('TIME')` not the client clock.
2. NTP sync across the fleet — keeps drift under ~10ms in practice. We accept that 10ms × R refill rate of error.

For sliding window counter, same fix — anchor windows to Redis-server time.

---

### Deep dive 2: Distributed counter consistency — single Redis vs cluster vs local approximation

> **Interviewer:** "You're at 30M counter ops/sec. How do you scale Redis behind that?"

**Candidate:**

Three architectural options. I'll walk each.

#### Option A: Single Redis primary

- One Redis instance, ~100K ops/sec ceiling.
- 30M ops/sec ÷ 100K = **300× short**. Doesn't fly.
- Plus: single point of failure. If Redis dies, the entire limiter is offline.
- **Reject.**

#### Option B: Sharded Redis cluster (consistent hashing)

- Hash the rate-limit key, route to one of N Redis shards.
- 100 shards × 100K ops/sec = 10M ops/sec. With 200 shards we're comfortable at 20M ops/sec headroom for 30M peak.
- Each shard is independent — no cross-shard atomicity needed because each rate-limit key lives entirely on one shard.
- **Hash slots:** Redis Cluster uses 16,384 hash slots; we use consistent hashing client-side with virtual nodes for smoother resharding.
- **Failover:** Redis Sentinel or Redis Cluster's built-in primary/replica failover. Lose a primary → replica promoted in ~5–10s.

This is the common production answer. But the latency budget pressures us further.

#### Option C: Local approximation + periodic sync (sloppy counter)

The insight: we don't need exact counts in real-time. We accept ~5% over-allowance.

- **Each gateway host** keeps a local counter for keys it has recently seen.
- Local counter increments on every request **without going to Redis**.
- Every 100ms (or every 100 hits, whichever first), the host pushes its delta to Redis and pulls the global state back.
- If the local counter alone is already over the limit: DENY immediately.
- If the global counter (last sync) is over: DENY.
- Otherwise: ALLOW (and increment locally).

**Math on accuracy:**
- Sync interval: 100ms.
- Number of gateway hosts touching a key: K.
- Worst case over-allowance per sync window = K × (limit per 100ms slice).
- For most keys, K ≈ 1 (key is per-user, hashes to one host via session affinity). Over-allowance is negligible.
- For hot keys (celebrity, large tenant), K could be 1000+. Over-allowance = 1000 × tiny = could be ~5–10% of the limit.
- **Tunable knob:** shorten sync interval (10ms → 5× more Redis QPS but tighter accuracy). Lengthen for cheaper.

**Why this is the right answer at our scale:**
- Cuts Redis QPS by ~10–100× (from 30M to ~300K–3M ops/sec).
- Cuts node count from ~300 to ~50.
- Brings p99 latency from ~1.3ms to ~110µs for the 95% of requests served from local cache.

#### Option D: Token-bucket-in-client-SDK with periodic reconciliation

- Service mesh: every service knows its own per-call quota and tracks it locally. Periodically reconciles with a central authority for over/under shoot.
- Best for east-west service-to-service traffic (microservice mesh) where the caller is trusted (i.e., not a malicious user).
- Less appropriate for user-facing APIs where the client is untrusted — a malicious client could lie about its local count.

#### My pick

**Sharded Redis cluster (B) with sloppy counters (C) layered on top.** The sidecar holds local approximations and only consults Redis when borderline or on sync. This gets the best of both: Redis is the global source of truth, but most requests never hit it.

For service-mesh internal calls (D) is layered on the same primitive — same Redis, but we add a per-pod token bucket as a first-line check.

#### Visual: the three "tiers" of consistency

```
                                                strict ↑
                                                       │
  ┌──────────────────────────────────────────────┐    │
  │  Redis Lua (atomic check-and-decrement)      │   strict
  │  + global aggregation across cells           │    │
  │  ~3ms, exact within cell, eventual cross-cell│    │
  └──────────────────────────────────────────────┘    │
                                                       │
  ┌──────────────────────────────────────────────┐    │
  │  Redis Lua, single-cell (default)            │    │
  │  ~1ms, exact within cell                     │    │
  └──────────────────────────────────────────────┘    │
                                                       │
  ┌──────────────────────────────────────────────┐    │
  │  Sloppy counter (sidecar local + 100ms sync) │    │
  │  ~100µs, ±5% over-allowance                  │   loose
  │                                              │    │
  └──────────────────────────────────────────────┘    │
                                                eventual ↓
```

The rule config picks which tier to use per-limit. Critical billing-quota limit? Tier 2. Per-API soft limit? Tier 3.

---

### Deep dive 3: Hot keys

> **Interviewer:** "What happens when one tenant has 100× the traffic of all others?"

**Candidate:**

Hot keys are the most underestimated problem in distributed rate limiters. They show up as:

1. **Celebrity user** posting an API key on a public site.
2. **Large enterprise tenant** doing 1M QPS to the same endpoint.
3. **Shared-IP NAT** — thousands of corporate users behind one egress IP.
4. **Per-API limit** that all users contend on (e.g., "global cap on POST /search = 10K/sec").

In all cases, traffic concentrates on one Redis key → single shard → single CPU core. Redis is single-threaded per shard, so one hot key bottlenecks everything.

#### Mitigation 1: Pre-shard hot keys

When we detect a hot key (top-N detection — see Deep Dive 6), we **sub-shard** it by appending a random suffix:

```
Original key: "rl:tenant=acme:api=POST_/search"
Becomes:      "rl:tenant=acme:api=POST_/search:#0"
              "rl:tenant=acme:api=POST_/search:#1"
              ...
              "rl:tenant=acme:api=POST_/search:#99"
```

Each request is randomly routed to one of N sub-keys. Each sub-key has limit/N. So if `acme` had a 10K/sec limit, each sub-key is 100/sec, summed = 10K.

**Read coalescing:** for the "explain" endpoint or for global state queries, we need to read all N sub-keys and sum. Lua `EVAL` over a key set can do this in one round trip. Cost: 100× Redis ops for one read query, but reads are rare relative to writes.

**Tradeoff:** sub-sharding only works for limits where sum is acceptable. Doesn't work for "exact 10K/sec per second across all sub-keys" if the requests are bursty enough that sub-key #5 gets 200 in one slice while #95 gets 0 — we still allow 200 even though the global limit is 100. The classic hash-bucket bias problem.

**Mitigation:** keep the random distribution roughly uniform; in practice ±5% over-allowance from sub-sharding is acceptable.

#### Mitigation 2: Local per-host approximate limiter

For really hot keys, every gateway host keeps its OWN local token bucket sized for its expected share of the global limit. No Redis call.

- 100 gateway hosts, global limit = 10K/sec → each host gets 100/sec local bucket.
- Hosts that are quiet leave tokens unused (slight under-utilization). Hosts that are slammed deny — slight under-allowance for those hosts but global is correct.
- Cheaper than sub-sharding but loose: assumes uniform traffic across hosts.

**Refinement:** dynamic re-allocation. If host A is hitting its local limit while host B is idle, A "borrows" tokens from B via a coordinator. This is the **distributed leaky bucket** pattern.

#### Mitigation 3: Detection-driven

The Hot-Key Detector (separate service, see architecture) consumes a sample of all rate-limit checks and emits the top-N keys by request rate. When a key crosses 100K req/sec, it's auto-promoted to "hot key" treatment (sub-sharded). This is a 30-second feedback loop, not real-time, but catches the long-running hot keys.

For burst-style hot keys (sudden viral event), the gateway has a circuit-breaker pattern: if a single key exceeds 10× expected rate, the gateway temporarily applies host-local rate limiting until the global state catches up.

> **Interviewer:** "What if a malicious actor specifically targets the limiter to overload it?"

**Candidate:** That's a denial-of-service attack on the limiter itself. Mitigations:
- **L3/L4 protection at edge** (out of scope but assumed) blocks volumetric attack.
- **Per-IP cap on number of distinct keys touched per second** — prevents an attacker from generating 1M unique synthetic keys per second.
- **Limit the number of distinct keys we'll create per-tenant per-second** — if `acme` suddenly has 1M new user IDs in their requests, that's suspect.
- **Failover to local-only mode** — if Redis is overloaded by attack, sidecars fail over to host-local approximate limits. Worst case: ~5% over-allowance, but we don't go down.

---

### Deep dive 4: Failure modes — fail open vs fail closed

> **Interviewer:** "Redis goes down. What happens to traffic?"

**Candidate:**

This is the single most consequential design decision, and the answer depends on which limit we're enforcing.

#### Fail-open (default)
- Redis unreachable → limiter returns ALLOW for all checks.
- All upstream APIs receive their full traffic, no rate limiting.
- **Why default:** the limiter is a *protection* layer, not a *correctness* layer. If Redis is down, we'd rather over-serve users than break the product. A 5-minute outage of the limiter shouldn't cause a 5-minute outage of the entire API surface.
- **Risk:** during the outage, abusive clients can hammer the API. But the API has its own degradation (autoscaling, circuit breakers); the loss of rate limiting alone is rarely catastrophic.

#### Fail-closed
- Redis unreachable → limiter returns DENY for all checks.
- Only used for specific high-stakes limits: **billing quotas**, **compliance-driven limits** (e.g., "free tier max 1000 calls/month — don't allow exceeding even on outage"), **financial transaction caps**.
- **Why:** for these limits, over-allowance has direct financial / legal cost > the value of availability.

#### Implementation

Each rule has a `fail_mode` config:
```yaml
- match: { tenant: "*", api: "*" }
  algorithm: token_bucket
  limit: 1000
  window_sec: 1
  fail_mode: OPEN          # default
- match: { tenant: "*", api: "POST /charge" }
  algorithm: token_bucket
  limit: 100
  window_sec: 1
  fail_mode: CLOSED        # billing-critical
```

The sidecar's behavior on Redis failure:
- For OPEN rules: log error, return ALLOW.
- For CLOSED rules: log error, return DENY.
- For ALL rules during a brownout (high error rate, not full outage): exponential backoff retries with jitter; circuit-break to fail-mode after 50% error rate over 5s.

#### Other failure modes

| Failure | Detection | Mitigation |
|---|---|---|
| Redis primary down | Sentinel / cluster failure detection (~5s) | Promote replica; sidecar reconnects |
| Redis brownout (slow) | Sidecar p99 latency alarm | Circuit-break to local-only sloppy counter; alert on-call |
| Sidecar OOM / crash | Pod liveness check; gateway calls fail | Gateway falls back to fail_mode default; pod restarts |
| Gateway → sidecar IPC fails | gRPC health check | Same as sidecar crash |
| Network partition (multi-AZ) | Per-AZ Redis replica latency | Sidecar prefers same-AZ Redis; falls back cross-AZ |
| Clock skew between sidecars | NTP drift monitor | Anchor clock to Redis server time inside Lua |
| Limit rule push fails | etcd watch error | Sidecar uses last-known-good config; alerts on-call |
| Hot-key detection lags | Top-N detector is 30s window | Manual override via control-plane API to mark a key hot |
| Cold start (new pod, empty cache) | First N requests have no local state | Sidecar marks self as "warming", uses Redis for everything for first 10s |

#### The "what's the worst case" answer

> **Interviewer:** "Walk me through what happens when the entire Redis cluster goes down for 5 minutes."

**Candidate:**
1. **t+0s:** Redis becomes unreachable. Sidecars detect via health-check + connection errors.
2. **t+1s:** Sidecars switch to fail-mode per-rule. OPEN rules: all ALLOW. CLOSED rules: all DENY.
3. **t+1s:** PagerDuty alarm: limiter Redis unreachable. On-call paged.
4. **t+1s to t+5min:** Traffic is unrestricted on OPEN rules. Upstream APIs see ~5% increase in traffic from formerly-rate-limited clients (most don't even notice, since most never hit limits). Some abusive clients could spike, but autoscaling absorbs.
5. **t+1min:** SRE confirms Redis cluster issue, initiates failover to standby cluster.
6. **t+3min:** Standby promoted, sidecars reconnect. Counters are reset (lost ephemeral state) — this means a brief window where every user has fresh quota. Acceptable.
7. **t+5min:** Limiter back to normal. Audit log shows ~$X in over-allowance traffic served. Postmortem.

We do **not** go down. We do **not** start denying user traffic because our limiter is sad. This is THE design tenet.

---

### Deep dive 5: Sidecar pattern vs dedicated fleet

> **Interviewer:** "Why a sidecar and not a dedicated rate-limiter service fleet?"

**Candidate:**

Three deployment models. Each has tradeoffs.

#### Model A: Inline middleware (in-process)
- Limiter logic compiled into the gateway binary itself.
- **Pros:** zero RPC overhead, sub-microsecond.
- **Cons:** couples deployment cycles (limiter bugs require gateway redeploy); language-bound (gateway in Go → limiter in Go); shared memory (limiter OOM crashes gateway).
- **When:** very small systems, latency-allergic.

#### Model B: Sidecar (this design)
- Separate process on the same pod / host as the gateway. Communication via UDS or localhost gRPC.
- **Pros:** independent deploy; language-flexible; isolation (limiter crash doesn't take gateway with it); shares host resources (very low RPC cost).
- **Cons:** doubles container count; per-pod resource overhead.
- **When:** **default for service-mesh-style deployments**; what Envoy + ratelimit do.

#### Model C: Dedicated fleet
- Separate set of hosts running limiter; gateway calls over network.
- **Pros:** centralized; easier capacity tuning; no per-pod overhead.
- **Cons:** network hop adds latency (1–2ms intra-DC); if fleet is unreachable, every gateway is affected; cross-AZ traffic costs.
- **When:** when you have very few gateways; or when limiter has heavy state (large in-memory caches) you can't replicate per-pod.

#### My pick: Sidecar

At our scale (10M QPS), every microsecond matters. Sidecar gets us to ~110µs in the common case. A dedicated fleet would add ~1ms RTT minimum — that's ~10× the budget for the common case.

#### Envoy + ratelimit reference

The industry-standard implementation:
- Envoy is the gateway.
- Envoy's `ratelimit` filter calls a separate `ratelimit` service (gRPC) per request.
- The `ratelimit` service is typically deployed as a sidecar to Envoy.
- Counter state lives in Redis.

We follow this pattern. Specifically:
- Envoy's HTTP filter chain includes the ratelimit filter.
- The filter calls our sidecar with a `RateLimitRequest` (descriptors).
- The sidecar replies `OK` or `OVER_LIMIT`.
- Envoy adds standard `X-RateLimit-*` headers automatically.

This buys us a battle-tested integration; we don't build the gateway-side glue.

---

### Deep dive 6: Hot-key detection & dynamic shard mitigation

> **Interviewer:** "How do you actually detect that a key is hot?"

**Candidate:**

Hot-key detection is a streaming top-N problem. Three approaches:

#### Approach 1: Sample-based at sidecar
- Each sidecar samples 1/1000 of its check operations and emits to a Kafka topic.
- A streaming aggregator (Flink / Kinesis Analytics) maintains a top-N count over a 30s window.
- Top-N keys with rate > threshold are emitted as "hot" events.
- These events are pushed to all sidecars via the control plane (etcd watch).
- Sidecars start sub-sharding those keys.

**Cost:** 30M ops/sec × 1/1000 = 30K events/sec into Kafka. Trivial.

**Latency:** ~30s from key getting hot to mitigation kicking in. For most production workloads this is fine.

#### Approach 2: Count-Min Sketch in sidecar
- Each sidecar maintains a CMS (count-min sketch) of recent keys.
- Periodically (every 10s) flushes top-N candidates to a central aggregator.
- Aggregator merges sketches, identifies global hot keys.
- Faster (~10s) detection, slightly more sidecar memory (1–10 MB per sidecar for CMS).

#### Approach 3: Threshold-based at Redis
- A separate Lua script is called on every check: if the per-key counter exceeds a threshold in a 1s window, write that key into a hot-key list.
- Hot-key list is read by the control plane.
- **Problem:** adds Redis ops on the hot path. We rejected.

#### My pick

Approach 1 (sampled to Kafka) for simplicity and no hot-path cost. CMS (Approach 2) as a future optimization if 30s latency is too slow.

#### What "mitigation kicking in" looks like

When the control plane emits `hot_key: rl:tenant=acme:api=POST_/search`:

1. Control plane writes a "hot key entry" to etcd with TTL of 5 minutes.
2. Sidecars watching etcd pick it up within ~1s.
3. From that point, sidecars route requests for that key to one of N sub-keys (e.g., N=100).
4. Original Redis shard for that key cools down within seconds.
5. After 5 minutes of no hot signal, etcd entry expires and we revert to single-key mode.

**Operator visibility:** dashboard shows current hot keys, when they were promoted, observed rate. On-call can manually promote/demote via control plane API.

---

## 9. Tradeoffs & alternatives

### Decision: Token bucket as default algorithm

| Option | Pros | Cons | When to pick |
|---|---|---|---|
| **Fixed window** | Simplest impl | 2× boundary spike | Soft limits where 2× burst tolerated |
| **Sliding window log** | Exact | O(N) memory — 8 TB at our scale | Low-cardinality, low-cap limits |
| **Sliding window counter** | O(1), ±5% accurate, smoothed | Approximate; harder to explain to users | High-cardinality default |
| **Leaky bucket** | Smooth output rate | No bursts allowed | Shaping outbound to a third-party hard cap |
| **✅ Token bucket** | O(1), bursts configurable, well-known | Burst can hit downstream hard if B large | **User-facing API limits** |

### Decision: Sloppy counter + Redis cluster

| Option | Pros | Cons | When to pick |
|---|---|---|---|
| Single Redis | Simple | 100K ops/sec ceiling, SPOF | Small scale (< 1M QPS) |
| Redis cluster (sharded) | Scales | Still ~1ms per check | Default for medium scale |
| **✅ Sharded Redis + per-host sloppy counter** | Cuts Redis QPS 10–100×; sub-millisecond common case | ±5% over-allowance | **Our scale (10M+ QPS, latency-sensitive)** |
| DynamoDB atomic counters | Managed, durable | Expensive ($$$); 5–10ms latency | When you need durability and don't have ops team |
| Custom gossip-based | Theoretically scales | Complex; convergence under attack hard | Research / future-proofing |

### Decision: Redis with Lua scripts, not Memcached

| Choice | Why |
|---|---|
| **✅ Redis** | Atomic check-and-decrement via Lua. Sorted sets if needed for sliding window log. Pub/sub for control-plane fanout. |
| Memcached | No atomic ops beyond CAS — would need round trips; race conditions at boundary. **Bad fit.** |
| DynamoDB | Atomic counters work, but cost at 30M ops/sec is prohibitive. Latency 2–5× Redis. |
| Cassandra | Counter columns work but eventually consistent — over-allowance under contention. Overkill for ephemeral state. |

### Decision: Per-cell counters, eventual cross-cell aggregation

| Option | Pros | Cons | When to pick |
|---|---|---|---|
| Single global Redis | Exact | Cross-region RTT blows latency budget | Never at our scale |
| **✅ Per-cell counters + async aggregation** | Latency budget met; cellular blast radius | ~5% over-allowance globally for global quotas | **Default** |
| Synchronous cross-cell quorum | Exact | Multi-region RTT in hot path | If exact global enforcement is regulatory |

### Decision: Fail-open default, fail-closed configurable

| Choice | Why |
|---|---|
| **✅ Fail-open default** | Limiter is a protection, not a correctness layer. Over-serving > breaking the product on a limiter outage. |
| Fail-closed always | Better safety, but turns a limiter outage into a product outage — cure worse than disease. |
| Per-rule configurable | Allows fail-closed for billing/compliance, fail-open for soft limits. |

---

## 10. Defending architectural choices — interviewer attacks

> **Interviewer:** "Why not just use the cloud provider's rate limiter (e.g., AWS API Gateway throttling)?"

**Candidate:** Two reasons. First, those products have very limited configurability — coarse-grained per-API-key throttling, not the composite (per-user × per-tenant × per-API) keys we need. Second, vendor lock-in: a multi-cloud or on-prem deployment can't depend on it. AWS API Gateway throttling is appropriate as a *defense-in-depth layer* upstream of our own limiter — but not a replacement.

> **Interviewer:** "Why a sidecar, not in-process?"

**Candidate:** Independent deploy cycle is the big one. The limiter is configured by a different team than the gateway. Putting the limiter in-process couples those release pipelines, and a limiter bug requires a gateway emergency redeploy. Sidecar isolates that. Latency cost of UDS IPC is ~50µs — within budget.

> **Interviewer:** "Sloppy counters mean over-allowance. What if a customer has paid for exactly 1000 calls/hour and we let them do 1100?"

**Candidate:** That's the billing-critical path I called out — those use fail-closed semantics AND strict-mode counters (every check goes to Redis, no local approximation). The 5% over-allowance is for *soft* limits where over-allowance is a small operational cost, not a billing fraud. The rule config picks the tier per-rule.

> **Interviewer:** "Why not use a database with atomic counters like DynamoDB?"

**Candidate:** Cost and latency. At 30M ops/sec, DynamoDB write throughput would cost millions per month. Latency is 2–5× Redis intra-DC. The durability DynamoDB gives us, we don't need — counters are ephemeral; losing them on a Redis crash means brief over-allowance, which is fine.

> **Interviewer:** "What if a single tenant has millions of API keys, each with its own limit?"

**Candidate:** Two effects. (1) Key cardinality grows — 100M total keys is fine; if one tenant has 100M+ unique keys, we'd start to worry about Redis memory in that tenant's shard. We'd cap keys-per-tenant in the rule schema. (2) The sloppy-counter local cache size grows — sidecars may not have all keys cached, missing more often → more Redis traffic. We monitor cache hit rate per-tenant; if a noisy tenant degrades hit rate globally, we'd shard that tenant onto its own limiter cluster.

> **Interviewer:** "What's the consistency model for the rule config? If I update a rule, when does it take effect?"

**Candidate:** Rules are propagated via etcd → sidecar watch → in-memory swap. Propagation latency is ~1–5s across the fleet. We accept that during a rule change, different sidecars may briefly enforce different limits. For non-emergency changes, this is fine. For an emergency tightening (e.g., kill-switch a runaway tenant), we have a "hard apply" path that bypasses the gradual rollout — pushes synchronously to all sidecars, blocks until 95% confirm. Adds 5s of operational latency but provides a strong guarantee.

> **Interviewer:** "Suppose a customer claims they were rate limited but shouldn't have been. How do you debug?"

**Candidate:** That's the `/explain` endpoint. The limiter samples DENY events to Kafka with full context: tenant, user, API, matched rule, current count, limit, timestamp. Hot tail (last 1 hour) is in Redis Streams, queryable by tenant. Long tail in S3 / Snowflake. The `X-RateLimit-Bucket` header on the 429 response includes the matched rule descriptor, so the customer can grep for it. This is THE customer-visibility feature; without it, every limiter incident takes hours to diagnose.

> **Interviewer:** "What if the descriptor is malformed or unknown?"

**Candidate:** The sidecar treats unknown descriptors as ALLOW (fail-safe for unknown patterns — same philosophy as fail-open). Logs a warning. The control plane should have surfaced this as a config error before deploy. We monitor `unknown_descriptor_count` — if it spikes, someone deployed a service emitting descriptors the limiter doesn't recognize.

> **Interviewer:** "Could you use the gateway's local cache to remember recent ALLOWs and skip the limiter?"

**Candidate:** No — that defeats the purpose. Every request needs to be counted; caching ALLOW means the next request "for free" doesn't decrement, and we under-count. The sloppy counter in the sidecar IS the local cache, but it counts every hit.

---

## 11. Failure modes & mitigations

| Failure | Detection | Blast radius | Mitigation | Recovery |
|---|---|---|---|---|
| Redis primary down | Connection error / Sentinel | All limiter traffic for keys on that shard | Replica promoted (~5s); sidecar reconnects | Auto |
| Redis cluster fully down | Health-check failure on all nodes | All limiter traffic | Sidecars switch to fail_mode (OPEN by default) | Restore Redis cluster; counters reset (acceptable ephemeral loss) |
| Redis brownout (high latency, not down) | p99 sidecar→Redis latency > 50ms | Limiter latency degrades | Circuit-break to local-only sloppy mode | Redis recovers; sidecar reconnects |
| Sidecar crash | Pod liveness | One pod's traffic | Pod restarts; gateway calls fail in interim | k8s restart (~30s) |
| Gateway → sidecar IPC fails | gRPC health | One pod | Gateway uses fail_mode default | Sidecar recovers |
| Hot key (single key over 10× expected rate) | Top-N detector | One Redis shard CPU pegs | Auto-shard the key (sub-key suffix); ~30s detection lag | Auto |
| Adversarial key explosion (1M new keys/sec) | Key creation rate per tenant | Sidecar memory bloat | Per-tenant cap on new-keys-per-second; rate-limit the rate-limiter (recursive) | Throttle attacker |
| Clock skew across hosts | NTP drift monitor | Slight over/under-allowance on token bucket | Anchor to Redis server clock in Lua | NTP convergence |
| Cold start (new pod, empty cache) | Pod age < 30s | First requests through that pod hit Redis 100% | Sidecar uses Redis on cold start; warms cache as it observes traffic | Auto, ~10s |
| Rule config push fails | etcd watch error | New rules don't apply; old rules still enforce | Sidecar uses last-known-good; alerts on-call | Manually republish |
| Audit pipeline (Kafka) backed up | Consumer lag alarm | "Explain" endpoint returns stale data | DENY decisions are still made correctly; audit lag is informational only | Drain Kafka; not user-impacting |
| Cross-cell aggregator down | Replication lag alarm | Global quotas may over-allow per-cell | Per-cell limits still enforced; global is approximate | Aggregator restart; sync resumes |
| Schema mismatch on rule (added a new field) | Sidecar unmarshalling error | Affected rules treated as missing; default = ALLOW | Schema is forward-compatible by convention; reviewer must check | Deploy rollback if needed |
| Sidecar OOM (cache too big) | Memory pressure | Pod evicted, gateway falls back | Cache size cap; LRU eviction of cold keys | Auto-restart |
| Network partition between sidecars and Redis | Per-AZ alarm | Affected AZ's limiter degrades | Sidecars prefer same-AZ Redis replica; fall back to local-only | Network heal |

---

## 12. Productionization

### Pre-launch
- [ ] **Shadow mode** for 2 weeks: limiter computes decisions but never enforces. Gateway proceeds as if everything is ALLOW, but logs what *would* have been denied. Compare against production for false-positive / false-negative rate.
- [ ] **Load test to 5× peak QPS** (50M check ops/sec). Verify p99 latency holds, Redis cluster doesn't OOM, no sidecar crashes.
- [ ] **Chaos test:** kill random Redis nodes during load test. Verify failover is < 10s, no data loss user-visible (counters reset is OK).
- [ ] **Game day:** simulate full Redis outage, verify fail-open behavior, verify alarms fire, verify customer-visible APIs stay up.
- [ ] **Algorithm A/B:** run two algorithms (token bucket vs sliding window counter) on shadow traffic, measure deny-rate divergence. Decide per-rule.

### Rollout
- [ ] Start with **very high limits** — set every rule's threshold at 10× expected peak. Watch for any DENY events: those are bugs in our threshold-setting, not in customer behavior.
- [ ] **Per-rule rollout:** turn on enforcement one rule at a time, observe for 24h. P0 limits first, P1 last.
- [ ] **Per-tenant rollout:** for any tenant on the deny list (more than 0.01% deny rate during shadow), reach out to PM/customer-success before enforcing.
- [ ] **Gradual tightening:** ramp limit values over weeks. e.g., week 1 = 10× target, week 4 = target. Customers adjust without surprise.
- [ ] **Auto-rollback:** if deny rate exceeds 5× baseline in any 5-minute window, auto-revert to previous rule version.

### Capacity
- [ ] Reserve **50% headroom** on Redis cluster — node addition is slow.
- [ ] Sidecar memory cap with LRU; alert at 80% utilization.
- [ ] Per-region capacity sized for **2× peak** to absorb regional failover.
- [ ] Auto-scale gateway pods (and thus sidecars) on QPS, target 60% CPU.

### Migration (if replacing existing system)
- [ ] **Dual-evaluate:** old and new limiter both compute; gateway uses old's decision. Compare daily.
- [ ] **Cutover by tenant:** small tenants first, 1% of traffic. Bake 24h.
- [ ] **Old limiter stays for 2 weeks** as rollback insurance.
- [ ] Decommission old.

### Per-customer kill switch
- [ ] Operator can flip a tenant to ALLOW_ALL (debug-bypass) or DENY_ALL (block) instantly via control plane.
- [ ] Audit-logged, requires two-person approval for DENY_ALL on prod-tenant.

### Cost
- Estimated steady state: ~50 Redis nodes × ~$200/month = **$10K/month** for the limiter Redis tier. Sidecars are co-located so marginal CPU cost.
- Largest variable cost: cross-region replication for global-quota rules. Disable if not needed per-rule.
- Cost levers: shrink local cache (smaller pods); longer sloppy-sync interval (fewer Redis ops); cold-tier audit logs instead of Redis Streams.

### Compliance
- [ ] Audit log of all DENY events retained 90 days for billing-quota rules.
- [ ] Per-region data residency: limit rules and counter state stay in-region.
- [ ] Customer can request "rate-limit transparency report" — counts of their DENY events over a period.

---

## 13. Monitoring & metrics

### Business KPIs
- % of API requests denied (overall and per-tenant)
- Customer-reported false-positive rate (rate-limited when shouldn't have been)
- Cost per million checks ($/M ops)

### Service-level (SLIs)
- **Limiter check latency:** p50, p95, p99, p99.9. Alert: p99 > 5ms for 5 min.
- **Limiter availability:** % of checks returning ALLOW or DENY (vs UNKNOWN). Alert: availability < 99.99% for 5 min.
- **SLO:** 99.99% of checks resolve in <5ms p99; 99.999% return a non-UNKNOWN result.

### Key data-plane metrics
- **ALLOW rate** per rule
- **DENY rate** per rule (alert on sudden 10× spike — likely runaway client OR limiter bug)
- **Per-tenant DENY rate** — surfaced to customer-success dashboard
- **UNKNOWN rate** — should be near zero; spike means limiter unhealthy
- **Local cache hit rate** in sidecar (target ≥95%)
- **Redis call rate** per sidecar — should track with cache miss
- **Redis Lua exec latency** (p50/p99)
- **Sloppy-counter sync lag** — time since last sync per-key. Alert if median > 200ms.
- **Hot-key detector top-N output** — leaderboard dashboard
- **Sub-shard expansion count** — how many keys are in hot mode

### Component metrics

#### Sidecar
- IPC RPC latency (gateway ↔ sidecar)
- Local cache size, hit rate, eviction rate
- Goroutine count, GC pause time
- Outbound Redis QPS, error rate

#### Redis
- Per-shard ops/sec, latency p99
- Per-shard memory utilization
- Lua script error count
- Replication lag
- Hot-key detection (Redis-side: top keys by hit count)

#### Control plane
- Rule push latency (etcd write → sidecar apply)
- Rule push success rate
- Override count (kill-switches active)

#### Audit pipeline
- Kafka producer error rate
- Consumer lag
- Sample rate (% of denials sampled)

### Alert routing
- **Page (24/7):** Redis cluster unreachable, limiter availability < 99.99%, p99 > 10ms for 10 min
- **Ticket (next business day):** single Redis shard hot, sloppy-sync lag > 1s sustained, rule push failures
- **Dashboard-only:** cost trends, deny-rate trends per tenant, hot-key history

### Dashboards
1. **Limiter health** — ALLOW/DENY/UNKNOWN rates, p50/p99 latency, regional split
2. **Redis cluster** — per-shard ops, memory, replication lag, hot keys
3. **Per-tenant denial** — top-10 tenants by deny rate (operator visibility)
4. **Rule effectiveness** — deny rate per rule (highlights ineffective or runaway rules)
5. **Cost** — Redis op cost, $/M checks, trend over weeks

---

## 14. Security & privacy

- Rate-limit keys often contain PII (user IDs, IPs). Counter store does not retain content beyond the key itself; audit logs scrub IP after 30 days for GDPR.
- The `/explain` endpoint must require tenant-scoped auth — a tenant can only see their own denial events.
- Control plane mutations (rule changes, overrides) require two-person approval for production.
- mTLS between gateway, sidecar, Redis, control plane.
- Rate-limit bypass: an attacker who can forge descriptors could starve a specific user's quota. Mitigate by requiring auth context (signed) for descriptor fields like `user_id`.
- Audit trail of all kill-switch and override actions.

---

## 15. Cost analysis

| Component | Rough cost driver | Optimization lever |
|---|---|---|
| Redis cluster | ops/sec × node count | Sloppy counter cuts ops by 10–100×; sub-shard hot keys to spread load |
| Sidecar CPU | inline with gateway | Marginal — already running gateway pod |
| Cross-region replication (for global quotas) | egress bytes × keys synced | Per-rule opt-in only; disable for cell-local rules |
| Audit / Kafka | events × sample rate | 1% sample default; 100% only for high-value tenants |
| Cold storage (S3) for audit | bytes × retention | TTL 90 days; downsample older |
| Control plane | low QPS, cheap | Negligible |

**Largest cost: Redis ops at scale.** This is the primary optimization target, and it's why sloppy counters are a 10–100× cost lever, not just a latency lever. A naive "every check hits Redis" design at 30M QPS would need ~300 Redis nodes; with sloppy counters we get to ~50 nodes — about $40K/month savings in compute alone.

---

## 16. Open questions / what I'd validate with PM and infra

1. Is exact accuracy required for any specific limit class (billing, compliance, regulatory)? Drives strict-mode rule count.
2. What's the cross-region traffic pattern? If users frequently cross cells (e.g., traveling), per-cell counters under-count.
3. What's the upper bound on key cardinality per tenant? Needed to size shards.
4. Are there limits we'd want users to override locally (e.g., paid customers can request higher quotas)? Drives the override / per-customer config plane.
5. What's the SLA we promise customers when their rate limit is wrong (false positive)? Drives the customer-comms playbook.
6. Should the limiter be the source of truth for billing? (Almost certainly no — billing should observe API logs separately. But worth confirming.)

---

## 17. Staff-level scorecard for this problem

| Signal | Did the candidate... |
|---|---|
| Identified algorithm tradeoffs explicitly | Compared 5 algorithms with memory/accuracy axes, picked one with reasoning |
| Quantified Redis QPS and node count | Showed why naive design needs 300 nodes, sloppy counter cuts to 50 |
| Volunteered fail-open vs fail-closed | Brought it up before being asked; tied it to per-rule config |
| Discussed hot keys with concrete mitigation | Sub-sharding via random suffix, with read-coalescing and bias acknowledgment |
| Treated latency budget rigorously | Walked the 5ms budget hop-by-hop; identified Redis RTT as the dominant cost |
| Picked sidecar architecture with reason | Compared inline / sidecar / dedicated fleet, justified sidecar at this scale |
| Identified counter consistency tiers | Three tiers: strict Redis, single-cell Redis, sloppy local |
| Named specific algorithms | Token bucket vs sliding window counter, not "we'll use Redis" |
| Designed for graceful degradation | Sidecar local-only mode on Redis brownout |
| Discussed productionization unprompted | Shadow mode, gradual ramp, kill switch, explain endpoint |
| Closed with wrap-up | See section 18 |

### What separates Staff from Senior on *this* problem

A Senior candidate will:
- Pick token bucket (or one algorithm) and implement it on Redis.
- Mention sharding.
- Note that Redis can fail.

A Staff candidate will additionally:
- Pick algorithm **per-rule type**, not globally.
- Identify that **the sloppy counter pattern is the cost lever**, not just a latency lever (10–100× Redis QPS reduction).
- Volunteer the **fail-open vs fail-closed** decision *and* tie it to per-rule semantics.
- Identify hot keys as a separate first-class problem, not "Redis will handle it".
- Insist on the `/explain` endpoint as a customer-debugging primitive, not an afterthought.
- Distinguish sidecar from dedicated fleet and defend the choice with latency numbers.
- Note clock skew as an actual risk (token bucket is wall-clock-sensitive).
- Discuss the cellular boundary: per-cell vs global limits, with eventual consistency for global.
- Mention shadow mode + gradual ramp + per-customer kill switch as productionization defaults.

---

## 18. Wrap-up speech (template)

> "To wrap up: the dominant constraints here are 10M QPS through the limiter at <5ms p99, with 100M+ active keys, and the requirement that the limiter NEVER becomes the reason the product is down. The architecture answers all three: token bucket as the default algorithm because it gives O(1) memory and matches user burst expectations; sloppy counters in a per-host sidecar that cut Redis QPS by 10–100× and bring the latency budget under 200µs in the common case; sharded Redis cluster for the source of truth, fronted by Lua scripts for atomic check-and-decrement; fail-open by default per-rule with fail-closed available for billing-critical paths. The two pieces I'd worry about in production are (1) hot keys — a celebrity tenant or shared-IP NAT can pin one Redis shard's CPU, mitigated by detection-driven sub-sharding with random suffixes, and (2) the cross-cell consistency story for global quotas — we accept ~5% over-allowance globally to avoid putting cross-region RTT on the hot path. To productionize: 2-week shadow mode, gradual ramp from 10× target to target over 4 weeks, per-tenant kill switch, customer-visible `/explain` endpoint with the matched rule descriptor in the 429 header. If we had another 30 minutes, I'd deep-dive (a) the cross-cell aggregation protocol for global quotas, or (b) the algorithm-per-rule selection logic and how it interacts with the rule-priority resolver."

---

## 19. Cellular / market isolation

> Bring this up if there's time, or if the interviewer pushes on multi-region. For an infrastructure component like the rate limiter, cellular is naturally simpler than for a stateful product system — but the cross-cell quota story is the interesting tradeoff.

### What & why

The rate limiter is **per-cell by default**. Each cell (US, EU, IN, BR, JP) runs its own:
- Limiter sidecars (co-located with each cell's gateway fleet)
- Redis cluster (counter store, in-region)
- Control plane (rule store, in-region)
- Hot-key detector
- Audit pipeline

Drivers:

1. **Latency** — the limiter must be on the hot path; no cross-region RTT allowed. Per-cell is the only way to keep p99 < 5ms.
2. **Blast radius** — a Redis outage in EU does not affect US traffic.
3. **Compliance** — GDPR / DPDP / LGPD: rate-limit state for EU users stays in EU. Even ephemeral counter state.
4. **Cost** — per-cell capacity sized to per-cell traffic, no over-provisioning to handle cross-region failover.
5. **Operational independence** — each cell's limiter team can deploy, monitor, scale independently.

### Cross-cell quota: the interesting design problem

What if a tenant has a **global quota** — "acme can do 1M API calls/hour across all our regions combined"? The rate limit doesn't fit cleanly into a single cell.

Three approaches:

| Approach | How | Pros | Cons |
|---|---|---|---|
| **Synchronous global quorum** | Every check consults a global counter (e.g., a single global Redis or a CRDT-replicated counter) | Exact | Cross-region RTT (80–150ms) on every check — blows the latency budget |
| **Per-cell budgets, equal split** | Each cell pre-allocated 1M/N where N = cell count. Cell enforces independently. | No cross-region calls; latency met | If usage is uneven across cells, one cell denies while another has headroom. ~30–50% under-utilization |
| **Per-cell budgets, dynamic re-allocation (✅)** | Each cell starts with equal share. A central aggregator reads consumption from every cell every 5–10s and re-allocates unused budget to busy cells. | Latency met; ~5% over-allowance globally; near-full utilization | Complex; ~10s reaction time on demand spikes |

**My pick:** dynamic re-allocation. Cells run independently; an out-of-band aggregator rebalances the budget every ~10s based on observed consumption per-cell. Slight over-allowance during the rebalance interval is acceptable.

### Cellular routing for rate limit decisions

- Most rate limits are **cell-local** by definition: per-user, per-IP, per-API-within-cell. No coordination needed. ~95% of rules.
- A small number of rules are **global**: per-tenant total quota, per-customer monthly cap. These use the dynamic re-allocation pattern. ~5% of rules.
- A tenant is assigned a **home cell** — most of their traffic lands there. Cross-cell traffic from that tenant is observed by the aggregator.

### Cross-cell aggregator architecture

```
              Each cell publishes per-rule counters to a shared bus
              (Kafka with topic-per-rule, or Pub/Sub)
                              │
                              ▼
              ┌────────────────────────────────────┐
              │  Cross-Cell Aggregator             │
              │  - Sums per-rule consumption       │
              │  - Allocates remaining budget to   │
              │    each cell every 5–10s           │
              │  - Pushes new per-cell budgets via │
              │    control-plane fanout            │
              └────────────────────────────────────┘
                              │
              ┌───────────────┼───────────────┐
              ▼               ▼               ▼
          US cell         EU cell          IN cell
          gets 400k       gets 350k        gets 250k
          this 10s        this 10s         this 10s
```

Failure mode: aggregator dies → cells freeze with their last-known budget. They keep enforcing per-cell, but no rebalancing happens. Acceptable degradation for ~10–30 min while we recover; at that point we're back to ~30% under-utilization but not over-limit.

### Per-cell rollout

Limit rule changes are rolled out per-cell. We can launch a new rule in IN first, observe for 24h, then EU/US. Same gradual-ramp pattern as the news-feed case study.

### When NOT to do cellular for the limiter

- Single-region product: just use one Redis cluster.
- Pre-scale: under ~1M aggregate QPS the operational tax of 5 cells exceeds the benefit.
- All limits are per-user / per-tenant (no globals): no cross-cell coordination needed; cellular adds no complexity.

### What "Staff signal" sounds like here

> "I'd run the limiter per-cell from day one if we have multiple regions, because the latency budget makes cross-region calls in the hot path infeasible. The interesting design problem isn't the cells themselves — it's the global-quota rules. I'd default to per-cell budgets with dynamic re-allocation every ~10 seconds, accepting ~5% over-allowance during rebalancing intervals. The exact-global-counter alternative isn't worth the latency, and the static-equal-split alternative wastes 30–50% of capacity. The aggregator is the new failure mode introduced by cellular — it's worth investing in its observability because when it dies, we silently underutilize."
