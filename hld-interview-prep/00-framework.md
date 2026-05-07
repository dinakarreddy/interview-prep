# 00 — HLD Interview Framework (Staff SWE @ FAANG, product-heavy)

> Before reading any specific problem, internalize this framework. Every problem in this folder follows the same shape, so the only thing that varies between interviews is the domain — not how you attack it.

---

## 1. The 60-minute clock

A typical FAANG HLD round runs 45–60 minutes. Most candidates blow the budget in scoping/estimation and then hand-wave the deep dives. Staff candidates ruthlessly time-box the early phases.

| Phase | Time | What "good" looks like |
|---|---|---|
| Clarifying scope + functional req | 5–7 min | 4–6 sharp questions that converge fast on a tractable scope. Not a checklist. |
| Non-functional requirements | 2–3 min | Numbers, not adjectives. "p99 < 200ms read", not "fast". |
| Capacity estimation | 3–5 min | DAU → QPS → storage → bandwidth. Show arithmetic, round aggressively. |
| API + data model | 5–7 min | Public surface only. Pick 1 primary table/collection's partition key intentionally. |
| High-level architecture | 8–10 min | Boxes + arrows. Walk the request path end-to-end for the *primary* flow. |
| Deep dive(s) | 15–20 min | This is where Staff is decided. Pick the highest-leverage 1–2 components. |
| Wrap: failures, scale, ops | 5 min | Volunteer this — don't wait to be asked. |

**Rule of thumb:** if you're 20 minutes in and haven't drawn a box, you're losing. If you're 35 minutes in and still drawing boxes (no deep dive), you're losing.

---

## 2. The RESHADED template (memorize)

I use a personalized version of the popular **RESHADED** mnemonic:

- **R**equirements (functional + non-functional)
- **E**stimation (back-of-envelope)
- **S**torage / data model
- **H**igh-level architecture
- **A**PI design
- **D**eep dives
- **E**dge cases & failure modes
- **D**eployment, monitoring, ops

You can re-order S/H/A based on the problem (some are API-first, some are storage-first), but cover all 8.

---

## 3. Senior vs Staff signal — the calibration delta

This is the most important section in this document. Most candidates fail Staff not because they're wrong, but because they sound like a strong Senior.

| Dimension | Senior signal | Staff signal |
|---|---|---|
| **Scoping** | Asks clarifying questions, scopes well | Picks scope **strategically** — explicitly excludes things and justifies why ("I'll skip live video b/c it doubles the design surface and the core feed is the harder problem") |
| **Estimation** | Computes correct numbers | Uses numbers to **drive design choices** ("8 PB total → can't fit on a single shard, can't afford SSD-only → tiered storage") |
| **Architecture** | Draws a correct system | Identifies the **2 hardest sub-problems** before drawing, and explicitly tells the interviewer they're the focus |
| **Tradeoffs** | Mentions tradeoffs when asked | Volunteers them **with the chosen alternative**, not just the rejected ones. "I picked Cassandra. The price is harder secondary indexing — I'll address that with..." |
| **Scale story** | Handles 10x | Handles 10x AND has a story for **what breaks at 100x** ("at 100x we'd need to shard the shard router") |
| **Failure modes** | Knows what can fail | Knows blast radius, **MTTR**, and which failures are user-visible vs absorbed |
| **Ops** | Mentions monitoring | Names specific SLOs, error budgets, alert routing, runbook ownership |
| **Cost** | Doesn't mention | **Quantifies** ($/req or $/DAU) and uses it as a tradeoff lever |
| **Interviewer interaction** | Answers questions | **Asks the interviewer** when stuck; flags assumptions explicitly; checks in at phase boundaries ("does this scope sound right before I dive in?") |
| **Disagreement** | Defers to interviewer | Defends the choice with reasoning, then asks "what am I missing?" — never just folds |

**Rule:** if the interviewer can't tell you what your highest-conviction architectural decision was at the end of the round, you scored Senior.

---

## 4. Clarifying questions — the master list

You don't ask all of these. You pick 4–6 based on the problem. The categories:

### Users & scope
- Who's the user? (consumer? B2B? internal?)
- Read-heavy or write-heavy?
- Global or regional? (drives multi-region question)
- Mobile-first, web, or both?
- What features are **in scope** for this discussion? (force the interviewer to commit)

### Scale
- DAU / MAU?
- Peak QPS, or peak/avg ratio?
- Median object size?
- Retention period?

### Quality of service
- Latency target (p50, p99)?
- Availability target (3 nines? 4? 5?)
- Consistency: strong, eventual, read-your-writes?
- Durability: can we lose any data? Under what failure scope?

### Constraints
- Existing infra to integrate with? (Kafka already? Cassandra fleet?)
- Greenfield or replacing something?
- Compliance: GDPR, HIPAA, PCI?
- Cost ceiling / cost per user target?

### Anti-questions (DO NOT ask)
- "What database should I use?" (you should propose, then defend)
- "Do you want me to talk about X?" (decide and inform)
- "Is this big enough scale to need sharding?" (estimate, then conclude)

---

## 5. Estimation cheatsheet

Memorize these. You should never compute them in-interview from scratch.

### Time
- 1 day ≈ 86,400 seconds ≈ 10⁵ s
- 1 month ≈ 2.5M s
- 1 year ≈ 32M s ≈ 3 × 10⁷ s

### Storage units (powers of 10 for back-of-envelope, though real life is 2¹⁰)
- 1 KB = 10³ B
- 1 MB = 10⁶ B
- 1 GB = 10⁹ B
- 1 TB = 10¹² B
- 1 PB = 10¹⁵ B

### Latency (Jeff Dean's numbers — internalize the orders)
| Operation | Time |
|---|---|
| L1 cache | 0.5 ns |
| L2 cache | 7 ns |
| Mutex lock/unlock | 25 ns |
| Main memory ref | 100 ns |
| Compress 1 KB w/ Zippy | 3,000 ns (3 µs) |
| Send 1 KB over 1 Gbps net | 10,000 ns (10 µs) |
| Read 4 KB from SSD | 150,000 ns (150 µs) |
| Read 1 MB sequentially from memory | 250,000 ns (250 µs) |
| Round trip in same datacenter | 500,000 ns (500 µs) |
| Read 1 MB sequentially from SSD | 1,000,000 ns (1 ms) |
| Disk seek (HDD) | 10 ms |
| Read 1 MB sequentially from disk | 20 ms |
| **Round trip CA → Netherlands** | **150 ms** |

### Throughput rules of thumb
- Single host SSD random read IOPS: ~100K
- Single host network: 1–10 Gbps (~125 MB/s – 1.25 GB/s)
- Single Kafka broker: ~100K msg/s sustained
- Single MySQL primary: ~5K writes/s, ~50K reads/s (without sharding)
- Single Redis: ~100K ops/s/instance
- Single Cassandra node: ~10K writes/s, ~5K reads/s
- A "good" service host handles ~10K QPS for typical request shapes

### Common derivations
- **DAU → peak QPS**: `(DAU × actions/user/day) / 86,400 × peak_factor` (peak_factor usually 3–5×)
- **Storage**: `writes/sec × avg_write_size × seconds × replication_factor × retention`
- **Bandwidth out**: `reads/sec × avg_response_size`

---

## 6. The pattern library

These are the building blocks. Every system you design is a composition of these. Know them cold.

### 6.1 Sharding strategies
| Strategy | When to use | Pitfall |
|---|---|---|
| Hash on key | Even distribution, no range queries | Resharding is painful (consistent hashing helps) |
| Range | You need range scans (time-series) | Hot ranges (today's data) |
| Geographic | Locality matters (users in EU stay in EU) | Cross-region operations |
| Lookup table (directory) | Tenants of vastly different sizes | The lookup itself becomes a bottleneck |
| Consistent hashing | Need elastic resharding | Virtual nodes required for balance |

### 6.2 Caching patterns
| Pattern | Read | Write | Use case |
|---|---|---|---|
| Cache-aside | App reads cache, falls back to DB on miss | App writes DB only; invalidates cache | Default choice |
| Read-through | Cache fetches from DB on miss | App writes DB, cache invalidated | Hides cache logic from app |
| Write-through | App reads cache | App writes cache, cache writes DB synchronously | Read-after-write consistency |
| Write-behind | App reads cache | App writes cache, cache async-writes DB | High write throughput, can lose writes |
| Refresh-ahead | Cache proactively refreshes hot keys | — | Hot, predictable keys |

**Cache invalidation policies:** TTL, LRU, LFU, FIFO. For news-feed-style hot keys: LFU or LFU-with-decay.

### 6.3 Queueing & async patterns
| Tool | Semantics | When |
|---|---|---|
| Kafka | Persistent log, partition-ordered, replayable | Event streaming, fanout to many consumers |
| RabbitMQ | Push-based broker, flexible routing | Complex routing, smaller scale |
| SQS | Managed, at-least-once, no ordering (FIFO variant exists) | Simple decoupling, AWS-native |
| Kinesis | AWS Kafka equivalent | AWS-native streaming |
| Redis Streams | Lightweight, in-memory | Low-latency, smaller volumes |
| Pub/Sub (GCP) | Managed, at-least-once | GCP-native |

**Delivery semantics:**
- At-most-once → fire and forget; safe to lose
- At-least-once → most common; consumer must be idempotent
- Exactly-once → expensive; usually really "effectively-once" via dedup keys + transactional writes

### 6.4 Consistency models
- **Strong** — every read sees the latest write (e.g., Spanner, single-leader RDBMS)
- **Linearizable** — strong + real-time ordering across keys
- **Sequential** — order respected per-client but not globally
- **Causal** — happens-before relationships preserved
- **Read-your-writes** — a client sees its own writes (often achieved with sticky sessions)
- **Monotonic reads** — never go backwards in time
- **Eventual** — converges, no time bound

When pushed on consistency: name the model, name the price, name the win.

### 6.5 Database choice cheatsheet
| Need | Pick | Why |
|---|---|---|
| Relational, transactions, complex queries | PostgreSQL, MySQL | ACID, mature, joins |
| Globally distributed, strong consistency | Spanner, CockroachDB, YugabyteDB | TrueTime/HLC + Paxos/Raft |
| Wide-column, write-heavy, big | Cassandra, ScyllaDB, BigTable | Tunable consistency, linear scale |
| KV, low latency | DynamoDB, Redis | O(1) lookup |
| Document | MongoDB, DocumentDB | Flexible schema |
| Time-series | InfluxDB, TimescaleDB, M3 | Compression, time-windowed queries |
| Search | Elasticsearch, OpenSearch | Inverted index, ranking |
| Graph | Neo4j, JanusGraph, Neptune | Multi-hop queries |
| Geo | PostGIS, S2-indexed Cassandra | Spatial indexes |
| Analytics / OLAP | Snowflake, BigQuery, Druid, Pinot, ClickHouse | Columnar, vectorized |

### 6.6 Geo-indexing tools
- **Geohash** — string prefix encodes proximity. Easy. Has corner-case issues at boundaries.
- **S2 (Google)** — hierarchical sphere covering. Best for global apps.
- **H3 (Uber)** — hex grid. Better adjacency than square grids.
- **Quadtree** — recursive subdivision. Good for non-uniform density.

### 6.7 Real-time delivery
- **Polling** — simplest, wasteful
- **Long polling** — better, hold connection open
- **SSE** — server → client, one direction, HTTP-friendly
- **WebSockets** — bidirectional, sticky connection
- **MQTT** — IoT, low-power, tiny header
- **gRPC streaming** — typed, multiplexed

### 6.8 Identity / dedup tricks
- **Snowflake IDs** (Twitter): `[timestamp | machine | seq]` — sortable, no central allocator
- **UUIDv7** — timestamp-prefixed, sortable, standard
- **Idempotency keys** — client-generated; server stores `{key → result}` for N hours

---

## 7. Defending architectural choices — common interviewer attacks

Staff signal is when you've already pre-played the rebuttal in your head. Memorize these dialogues.

### "Why X over Y?"
**Pattern:** state the dominant axis, name what Y is better at, name when you'd switch.

> Interviewer: "Why Cassandra over DynamoDB?"
>
> Candidate: "Three reasons: (1) we're multi-cloud and don't want vendor lock-in for a 10PB store, (2) we want tunable consistency per-query and Dynamo's strong/eventual is binary, (3) we already operate Cassandra. DynamoDB beats us on operational simplicity — if we were a 5-engineer team with no Cassandra muscle, I'd flip. At our scale the per-RCU/WCU pricing also makes Dynamo expensive."

### "What if X goes down?"
Always answer in this order: **(1) detection time, (2) blast radius, (3) automatic mitigation, (4) manual recovery, (5) data loss bound**.

### "Why is this fast enough?"
Walk the latency budget: client → LB → service → cache check → DB → response. Add up the components, compare to SLO, identify which hop dominates, name what you'd do if the SLO tightened.

### "Why this consistency level?"
Tie it to **a user-facing scenario**, not abstractly. "If two users like a post simultaneously and we lose one, the like count is off by one for a few seconds — users won't notice. So eventual is fine. If it were a payment, it wouldn't be."

### "What about cost?"
Have a $/DAU number in your head. Know which line item dominates (usually egress bandwidth or DB storage). Know one cost lever (tiered storage, sampling, batching, compression).

---

## 8. Productionization checklist (use this in every wrap-up)

You should always volunteer this in the last 5 minutes. It separates Staff candidates immediately.

### Rollout
- Feature flag with kill switch
- Dark launch / shadow traffic for 1–2 weeks
- Gradual rollout: 1% → 5% → 25% → 100%, with bake time at each step
- Per-region rollout if multi-region (start in lowest-traffic region)
- Auto-rollback on SLO breach

### Capacity
- Load test to 2× projected peak before launch
- Auto-scaling policy (target CPU/QPS/queue-depth)
- Reserved vs on-demand split
- Capacity headroom: 30–50% above peak by default

### Dependencies
- All upstream/downstream services notified
- SLAs negotiated with upstreams
- Circuit breakers + retries with jittered exponential backoff
- Bulkheads / per-dependency thread pools

### Data
- Schema migration plan (forward+backward compatible during rollout)
- Backfill plan if launching with historical data
- Backup + restore tested
- Retention policy + GDPR delete path

### Security
- Authn (mTLS service-to-service, OAuth/OIDC user-facing)
- Authz (per-route, per-resource)
- Secrets in vault, rotated
- PII tagging + access logging
- Rate limiting + DDoS protection at edge

### Observability
- See section 9.

### Operations
- Runbook for top 5 alerts
- On-call rotation defined
- SLO + error budget agreed with stakeholders
- Postmortem template ready

---

## 9. Monitoring & metrics — the 4 layers

Mention all four. Most candidates only mention layer 2.

### Layer 1: Business metrics
- DAU, MAU, retention, conversion
- The thing the PM cares about. Tie your system's health to a business KPI.

### Layer 2: Service-level indicators (SLIs)
- **RED** (for request-driven services): **Rate**, **Errors**, **Duration**
- **USE** (for resources): **Utilization**, **Saturation**, **Errors**
- p50/p95/p99/p99.9 — never just averages
- Per-endpoint, per-region, per-tenant if multi-tenant

### Layer 3: Component metrics
- DB: query latency, replication lag, connection pool saturation, slow queries
- Cache: hit rate, eviction rate, memory usage, key cardinality
- Queue: depth, age of oldest message, consumer lag
- LB: connection count, 5xx rate by upstream
- Per-service: GC time, thread pool saturation, FD count

### Layer 4: Infrastructure
- CPU, memory, disk, network — per host
- K8s: pod restarts, OOMKills, scheduling failures
- Network: packet loss, retransmits, p99 RTT

### Alerting hygiene
- **SLO-based alerts**, not threshold-based ("error budget burn rate of 14.4× over 1h" beats "errors > 100/min")
- Multi-window, multi-burn-rate (Google SRE workbook ch. 5)
- Page only on actionable, user-impacting issues
- Symptom-based at the top of the funnel; cause-based for diagnostic dashboards

### Dashboards
- One **service health** dashboard (golden signals only)
- One **deep-dive** dashboard per major component
- One **business** dashboard (DAU, conversion, etc.)
- Keep them under ~12 panels each — denser than that and nobody reads them

---

## 10. Communication tactics

### Drawing
- Top-left → bottom-right, request flow
- Use consistent shapes: rectangle = service, cylinder = DB, queue = parallelogram, client = stick figure
- Number arrows in the order they fire
- Don't draw before you've stated the scope

### When stuck
- Say "let me think for 10 seconds" — silence beats waffling
- Restate the question if confused
- Ask the interviewer "is the question X or Y?"

### Disagreeing with the interviewer
- Never fold immediately. Say "let me think about that — my reasoning was X, what's the failure mode you're seeing?"
- If they're right, concede explicitly: "you're right, I missed Y. Let me update the design."
- If you still disagree after thinking, say so politely with reasoning.

### Checking in
- At every phase boundary: "Does this scope feel right before I move on?"
- Before deep dive: "I see two candidates for deep dive — A and B. A is the harder problem; B is more product-impactful. Which would you rather see?"

### Naming things
- Use industry-standard names where they exist (Kafka, not "the queue"; CDN, not "edge cache" if you mean an actual CDN)
- Make up names when needed: "I'll call this the FeedRanker service"
- Don't say "microservice" without saying which microservice

---

## 11. Anti-patterns (instant Staff downgrades)

- Drawing before scoping
- Picking a database without saying why
- Saying "we'd use Redis" without saying for what
- Mentioning Kafka because it sounds smart, when SQS would do
- Adding a service to the diagram and never explaining its job
- Hand-waving consistency ("we'll make it eventually consistent")
- Ignoring multi-region until asked
- Not volunteering failure modes
- Using "microservices" as a punctuation mark
- Forgetting the user (every system serves a user — keep them in the picture)
- Refusing to commit ("it depends" on every question)
- Committing too fast (no tradeoffs ever surfaced)
- Showing zero awareness of cost
- Ignoring the interviewer's hints

---

## 12. The wrap-up speech (memorize a version of this)

In the last 2–3 minutes, you should always close with something like:

> "To wrap up: the dominant constraints here were [X, Y]. The architectural choices I made to address those were [A, B, C]. The two pieces I'd worry about in production are [D, E] — I'd mitigate D with [...] and accept E as a known limitation that shows up at [scale Z]. To productionize I'd flag-gate the rollout, dark-launch for two weeks, and SLO-alert on [primary signal]. If we had another 30 minutes, the next thing I'd want to deep-dive is [F]."

This single paragraph reliably moves "borderline" calibrations to "hire."

---

## 13. How to use this folder

1. Read `00-framework.md` (this file) until it's instinctive.
2. Pick a problem (`01-news-feed.md` is the canonical warm-up).
3. **Don't read it cover-to-cover first.** Read only the problem statement, close the file, and try to design it yourself for 45 minutes.
4. Then open the file and read the rest. Compare your approach to the script. Note where the script went somewhere you didn't.
5. Re-do the problem 24 hours later from scratch.
6. Move to the next.

The goal is not to memorize 12 designs. The goal is to internalize the *shape* of how a Staff candidate attacks any HLD problem.
