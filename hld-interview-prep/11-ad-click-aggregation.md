# 11 — Design a Real-time Ad Click Aggregation System

> The streaming-pipeline canonical. The interviewer cares less about the boxes you draw and more about whether you understand that **exactly-once semantics** and **late-arriving events** are the entire problem. Get those right, and the rest of the system (Kafka, Flink, Druid) is just plumbing. Get them wrong, and the system silently double-bills advertisers — which is fraud risk and lost revenue.

---

## 0. Problem statement (as the interviewer would drop it)

> **Interviewer:** "Design a real-time ad click aggregation system. Every time a user clicks on an ad, that click needs to be counted, deduped, and made queryable both in real-time (advertisers want to see clicks-per-minute on their dashboard) and historically (advertisers want to see CTR for the last 30 days). Walk me through how you'd build it."

That's the prompt. No mention of fraud, no mention of late events, no mention of cross-region. The candidate's job: surface those concerns *before* drawing a box.

---

## 1. Clarifying questions

Pick 5–6. Interleave with the interviewer's responses, don't fire them as a checklist.

> **Candidate:** "Before I design, let me scope. A few questions:"
>
> 1. **"What's the scale — clicks per second, ads, advertisers?"** — *Expect: 10M impressions/sec, ~1% CTR → 100K clicks/sec peak, ~1B clicks/day, 10M advertisers, 100K publishers, 100M+ active ads.*
> 2. **"What are the consumers of this aggregation? Advertiser dashboards, billing, ML training?"** — *Drives consistency, freshness, and retention. Expect: advertiser dashboards (real-time, ≤1 min stale), billing (daily reconciled, exactly-once), ML training (raw events archived).*
> 3. **"How fresh does 'real-time' need to be — sub-second, 1 minute, 5 minutes?"** — *Expect: 1-min freshness on dashboards, sub-second NOT required.*
> 4. **"What's the dedup requirement — best-effort, at-least-once with downstream dedup, or exactly-once?"** — *This is THE question. Expect: exactly-once semantics on counted output (because clicks are revenue).*
> 5. **"What about late-arriving events — clicks that show up minutes or hours after they happened (mobile offline, network delays)?"** — *Expect: yes, real problem. Late window of ~1hr, then dropped (logged for audit).*
> 6. **"Is fraud / bot click detection in scope?"** — *Expect: yes, signal-level (flag, don't delete). ML scoring out of scope at deep level but the integration point matters.*
> 7. **"Are advertisers querying any dimension at any time-grain, or pre-defined cube slices?"** — *Drives OLAP store choice. Expect: pre-defined dimensions (ad/advertiser/publisher/region × 1min/1hr/1day) with ad-hoc on raw for power users.*
>
> **Out of scope (state explicitly):**
> - Ad serving / auction (separate problem)
> - Bidding logic
> - Creative/asset storage and CDN
> - Billing / payment systems (we feed them, we don't run them)
> - User/advertiser authentication
> - Targeting and personalization
>
> **Candidate:** "I'm going to design the **click ingestion + dedup + aggregation pipeline**, the **real-time and historical query API**, and the **fraud signal pipeline** as a parallel consumer. Ad serving I'll treat as an upstream that emits clicks; billing I'll treat as a downstream that consumes our exactly-once aggregates. Sound right?"

> **Interviewer:** "Yes — and I want to spend most of the time on the dedup and late-event handling."

> **Candidate:** "Got it. I'll plan to deep-dive there."

---

## 2. Functional requirements

| Priority | Requirement |
|---|---|
| P0 | Ingest click events from edge endpoints with <100ms latency |
| P0 | Deduplicate clicks by client-generated `click_id` (idempotency key) |
| P0 | Aggregate clicks in real-time by (ad_id, advertiser_id, publisher_id, region) × (1min, 5min, 1hr) windows |
| P0 | Serve advertiser dashboards: "clicks for ad X over the last 7 days, by hour" with p99 < 2sec |
| P0 | Persist raw events for ≥30 days for billing audit and reconciliation |
| P0 | Tolerate late-arriving events up to 1 hour past event timestamp |
| P1 | Fraud signal pipeline: flag suspicious clicks (don't delete) |
| P1 | Daily/hourly batch reconciliation: raw count vs aggregated count |
| P1 | Schema evolution: add new dimensions to events without breaking pipelines |
| P2 | Cross-region global aggregates (advertiser sees worldwide CTR) |
| Out | Ad serving, bidding, creative storage, billing payment, user auth |

---

## 3. Non-functional requirements

| Dimension | Target | Justification |
|---|---|---|
| Ingest availability | 99.99% (~4 min/month) | Lost click = lost revenue + advertiser trust |
| Ingest latency | p99 < 100ms (edge → Kafka ack) | Click impression budget; user has navigated away if we're slow |
| Dashboard query p99 | < 2 sec for 7-day window | Advertisers refresh dashboards constantly |
| End-to-end freshness | < 1 min from click → dashboard | Real-time advertiser optimization expectations |
| Dedup correctness | Exactly-once on counted output | Double-billing is unacceptable; under-counting breaks revenue |
| Durability | No raw event loss after edge ack | Source of truth for billing disputes |
| Late event window | 1 hour (configurable) | Mobile offline, network jitter; trade off vs window correctness |
| Retention (raw) | 30 days hot, 1 year cold | Billing dispute window + ML training |
| Retention (aggregates) | 5 years rolled up to daily | Year-over-year reporting |
| Scale | 10M impressions/sec peak, 100K clicks/sec peak, ~1B clicks/day | Industry FAANG-class scale |
| Geography | Multi-region active-active for ingest; per-region aggregation; global merge | Latency + GDPR |

---

## 4. Capacity estimation

> **Candidate:** "Let me estimate aggressively, round numbers. Stop me if you want me to dig deeper."

### Volume
- **Impressions:** 10M/sec peak → ~10B/day
- **Clicks:** 1% CTR → 100K/sec peak → ~1B/day
- **Active ads:** 100M concurrently in market
- **Advertisers:** 10M
- **Publishers:** 100K

### Per-event size
- `click_id` (UUID, 16B), `user_id` (8B), `ad_id` (8B), `advertiser_id` (8B), `publisher_id` (8B), `timestamp` (8B), `ip` (16B v6), `user_agent` (~100B), `geo` (~50B), `referrer` (~150B), `device_id` (16B), `context_json` (~100B)
- Total: **~500 bytes per click event**, including overhead/protobuf framing

### Throughput
- Ingest write: 100K events/sec × 500B = **50 MB/sec → 400 Mbps** sustained, ~1.2 Gbps peak with 3× burst headroom
- Kafka throughput (with replication 3×): 150 MB/sec network per broker for replication
- Single Kafka broker handles ~100 MB/sec sustained → need ~5 brokers for raw ingest with headroom; in practice run **~50 brokers** across the cluster for durability, partitioning, and reprocessing replay capacity

### Storage
- **Raw events:** 100K/sec × 500B × 86,400 sec = **4.3 TB/day raw**, 3× replication on Kafka = ~13 TB/day on Kafka
- **30-day raw retention on S3 (compressed Parquet ~5×):** 4.3 TB × 30 / 5 = **~26 TB on S3**
- **1-year cold retention (compressed):** ~300 TB
- **Aggregates (Druid/Pinot):** rollups of (ad × minute × dimensions) — say 100M ads × 1440 min/day × ~50B per row = **~7 TB/day pre-aggregated**, but most ads have zero clicks per minute; sparse representation cuts this 10–100×. Realistic: **~300 GB/day in OLAP**, **~100 TB for 1 year** at minute granularity, much less if rolled up

### Query load
- 10M advertisers, but maybe **100K active dashboard users** at peak
- Each dashboard refreshes every ~10 sec, runs ~5 queries → **50K queries/sec peak** on the OLAP store
- Druid/Pinot can do this with proper segment design

### Decision drivers from numbers
- 100K clicks/sec → **must use Kafka** (Postgres can't ingest this); single Kafka broker can handle but we need partitioning for replay and parallelism
- 4 TB/day raw → must archive to **S3** (cheap object storage), not keep all hot
- 50K dashboard QPS → must use **OLAP (Druid/Pinot/ClickHouse)**, not row store
- 10M advertisers, 100M ads → dimension cardinality matters, OLAP rollup cubes need careful design

---

## 5. API design

> **Candidate:** "Two API surfaces here — the **ingest API** that ad-serving / SDKs hit, and the **query API** that advertiser dashboards hit. They have very different shapes."

### Ingest API (edge-facing)

```
POST /v1/click
  headers: { x-click-id: <uuid_v4>, x-client-version: <ver> }
  body: {
    ad_id, advertiser_id, publisher_id,
    user_id (or anon_id), session_id,
    event_ts (ms epoch, client time),
    ip, user_agent, geo_hint, referrer, device_id,
    context: { placement, slot, page_url_hash, ... }
  }
→ 200 { server_ts, click_id }   # 200 even on dup, idempotent
→ 4xx for malformed
```

**Critical design choices:**
- `click_id` is **client-generated UUID v4 or v7**. The server treats it as the idempotency key. Dedup is keyed on this.
- Server returns 200 *even on duplicate*. The SDK retries on network failure → server must be idempotent.
- Server attaches its own `server_ts` (authoritative for windowing) but preserves `event_ts` (for late-arrival accounting).
- Body is **Protobuf or Avro**, not JSON. Saves ~30% bandwidth and gives schema versioning.

### Query API (dashboard-facing)

```
GET /v1/aggregates?
  metric=clicks|ctr|spend
  dimension=ad|advertiser|publisher|region
  filter={ad_id: ..., advertiser_id: ...}
  time_range=last_7d|last_24h|custom(start, end)
  granularity=minute|hour|day
  group_by=[dimension, ...]
→ 200 { series: [{ts, dim_values, metric_value}], data_freshness_ts }
```

- `data_freshness_ts` is **mandatory in every response**. Tells the client "data is current as of this timestamp." Hides ingestion lag from being an outage signal — instead it's a labeled "your data is 5 min stale" warning.
- Pagination + cursor for large result sets.

### Internal admin API
```
POST /v1/reconciliation/run     # trigger batch reconciliation
GET  /v1/reconciliation/results # raw vs aggregated diff
GET  /v1/fraud/scores?ad_id=    # fraud scores per ad
```

### Anti-patterns to avoid
- Don't make the ingest API blocking on aggregation — it acks after the Kafka write, period.
- Don't return raw events from the dashboard query API — that's a Trino/Athena job, not a hot-path API.
- Don't expose internal flink job state to clients.

---

## 6. Data model

> **Candidate:** "There are five distinct stores here, each with a specific job. I'll walk through each."

### 6.1 `clicks.raw` (Kafka topic)

- Partitioned by `hash(click_id)` → 1024 partitions for parallelism and ordered dedup per-partition
- Replication factor 3, ack=all
- Retention: 7 days (long enough for full reprocessing, short enough to control disk)
- Compaction: **disabled** (we want every event, including dupes, for audit)
- Schema: registered in **Confluent Schema Registry** as Protobuf v1; backward+forward compatible

### 6.2 Dedup state (RocksDB embedded in Flink, OR Redis cluster)

| Field | Type | |
|---|---|---|
| click_id | UUID | PK |
| first_seen_ts | int64 | when we first saw this click_id |

- TTL = **24 hours** (events older than this can't dup against new arrivals because they're outside the watermark window)
- Storage: at 100K clicks/sec × 86,400 sec × 16B = ~140 GB / day. With 24h TTL, steady state ~140 GB. **Fits in RocksDB on a single Flink task slot per partition** with SSDs.
- Trade-off: RocksDB embedded vs Redis external — see deep dive 1.

### 6.3 `clicks.deduped` (Kafka topic)

- Output of dedup operator
- Same schema as `clicks.raw`, same partitioning by `click_id` for downstream parallelism
- This is the **canonical input** for all downstream consumers (aggregation, fraud, archival)

### 6.4 OLAP store (Druid, Pinot, or ClickHouse) — `click_aggregates`

Pre-aggregated cubes with these dimensions:

| Dimension | Cardinality |
|---|---|
| time_bucket | 1min, 5min, 1hr granularity |
| ad_id | 100M |
| advertiser_id | 10M |
| publisher_id | 100K |
| region | ~20 |
| device_type | ~10 |

Metrics (additive, suitable for stream rollup):
- `click_count`, `unique_user_count` (HyperLogLog), `fraud_flagged_count`, `late_arrival_count`

Druid segment design: partitioned by `time_bucket` daily, secondary index on `advertiser_id` (the most common filter). Segments for "last 24h" stay in RAM; older segments tier to deep storage (S3) and load on query.

### 6.5 Raw event archive (S3 + Iceberg / Delta Lake)

- Output of an archival consumer reading from `clicks.deduped`
- Format: **Parquet, partitioned by `event_date / hour / region`**
- Columnar, compressed (Snappy or ZSTD), 5–10× compression
- Iceberg manifest enables: schema evolution, snapshot isolation, time-travel queries, batch reprocessing
- Queried by **Trino / Athena** for ad-hoc historical analysis

### 6.6 Fraud scores (low-volume, served from KV)

| Field | Type | |
|---|---|---|
| click_id | UUID | PK |
| score | float | 0.0 = clean, 1.0 = fraud |
| flagged_reasons | list<enum> | "high_velocity", "bot_ua", "ip_blacklist", ... |
| scored_at | timestamp | |

- Stored in DynamoDB or Cassandra, keyed by `click_id`
- Joined back during billing reconciliation to filter/discount fraud

### Why these choices

- **Kafka** for ingest: durable buffer, replayable, partitioned. Non-negotiable at 100K/sec sustained.
- **RocksDB embedded in Flink** for dedup state: keeps dedup hot-path local (no network hop), checkpointed to S3 for recovery. Redis works but adds a network call per event.
- **Druid** for OLAP: low-latency time-series, columnar, sub-second percentile queries on billion-row tables. Pinot is comparable; ClickHouse is faster ad-hoc but harder to operate. Snowflake is the wrong tool here (latency-bound to seconds-minutes, not sub-second).
- **S3 + Iceberg** for archive: cheap, durable, time-travelable. Trino over Iceberg gives flexible historical queries without paying Druid's RAM cost.

---

## 7. High-level architecture

```
                                 ┌──────────────────────────────────┐
                                 │       Edge / CDN (per-region)    │
                                 │    Cloudfront-class, anycast IP  │
                                 └──────────────┬───────────────────┘
                                                │
                                       ┌────────▼─────────┐
                                       │  Click Ingestor  │ (validates, enriches,
                                       │  Service         │  emits to Kafka)
                                       └────────┬─────────┘
                                                │
                                                ▼
                                       ┌──────────────────┐
                                       │ Kafka            │
                                       │ topic:           │
                                       │ clicks.raw       │
                                       │ (1024 partitions,│
                                       │  RF=3)           │
                                       └────────┬─────────┘
                                                │
                                                ▼
                                ┌─────────────────────────────────┐
                                │  Flink Dedup Operator           │
                                │  - keyed by click_id            │
                                │  - RocksDB state (24h TTL)      │
                                │  - emits new click_ids only     │
                                │  - checkpoint to S3 every 30s   │
                                └────────────────┬────────────────┘
                                                 │
                                                 ▼
                                       ┌──────────────────┐
                                       │ Kafka            │
                                       │ topic:           │
                                       │ clicks.deduped   │
                                       └─┬───────┬────┬───┘
                                         │       │    │
                  ┌──────────────────────┘       │    └──────────────────┐
                  │                              │                       │
                  ▼                              ▼                       ▼
         ┌────────────────┐         ┌─────────────────────┐    ┌────────────────────┐
         │ Stream         │         │ Fraud Detection     │    │ Raw Archival       │
         │ Aggregator     │         │ Consumer (Flink+ML) │    │ Consumer           │
         │ (Flink)        │         │ - velocity rules    │    │ - writes to S3     │
         │ - tumbling     │         │ - bot UA detector   │    │   Iceberg/Parquet  │
         │   windows      │         │ - ML scoring        │    │ - hourly partitions│
         │ - watermarked  │         └──────────┬──────────┘    └────────────────────┘
         │ - late-arrival │                    │
         │   side-output  │                    ▼
         └───────┬────────┘            ┌──────────────┐
                 │                     │ Fraud Scores │
                 ▼                     │ KV (Dynamo)  │
       ┌──────────────────┐            └──────────────┘
       │ Druid / Pinot    │
       │ (real-time +     │
       │  historical OLAP)│
       └────────┬─────────┘
                │
                ▼
       ┌────────────────┐                    ┌──────────────────┐
       │ Query API      │  ←  also reads ←   │ Trino on S3      │
       │ (Dashboard svc)│     (long-range)   │ Iceberg          │
       └────────┬───────┘                    └──────────────────┘
                │
                ▼
       ┌────────────────┐
       │ Advertiser     │
       │ Dashboard      │
       └────────────────┘

Daily batch (Spark) reads S3 archive → reconciliation diff → alerts
```

### Request walk-through

**Click write path (the critical 100ms budget):**

1. SDK on user's device captures click, generates `click_id` UUID, fires `POST /v1/click` to nearest edge.
2. Edge (Cloudfront-class) → regional Click Ingestor service (~10ms RTT).
3. Click Ingestor:
   - Validates payload (schema, required fields)
   - Enriches with `server_ts`, `geo` (from IP), and traffic-source attribution
   - Publishes to `clicks.raw` Kafka with `acks=all` (waits for in-sync replicas)
   - Returns 200 to SDK
4. End-to-end p99 ~80ms within budget.

**Dedup pipeline (asynchronous, sub-second):**

5. Flink dedup job consumes `clicks.raw`, keyed by `click_id`.
6. For each event: lookup `click_id` in RocksDB state. If exists → drop (and increment `dup_count` metric). If not → insert with TTL 24h, emit to `clicks.deduped`.
7. Checkpoint every 30s to S3. On failure, restore from last checkpoint and replay from last-committed Kafka offset.

**Aggregation pipeline:**

8. Stream Aggregator (Flink) consumes `clicks.deduped`.
9. Keyed by (ad_id, dimension_tuple, time_bucket). Tumbling windows: 1-min, 5-min, 1-hr (parallel windows).
10. **Watermark = max(event_ts) - 1 hour** allowed lateness. Events beyond watermark go to a side-output stream (`clicks.late`) for batch reconciliation.
11. Window closes → emit aggregate row to Druid via Tranquility / push API.

**Query path (sub-2sec budget):**

12. Dashboard hits `GET /v1/aggregates` with filter and time range.
13. Query API service:
    - For ranges ≤ 7 days: query Druid (serves from RAM-resident segments)
    - For ranges > 30 days: query Trino over S3 Iceberg (slower, OK for historical reporting)
    - Stamp `data_freshness_ts` in response from a watermark indicator service
14. Returns < 2sec p99.

---

## 8. Deep dives

### Deep dive 1: Exactly-once semantics & dedup

> **Interviewer:** "Walk me through how you guarantee exactly-once. Clicks are revenue — double-counting is fraud risk."

**Candidate:**

The honest answer is **we don't get true exactly-once; we get effectively-once via at-least-once + idempotent dedup.** Let me explain why and how.

**The four sources of duplication:**

1. **SDK retry** — network blip, SDK retries the same `click_id`. Handled by dedup keyed on `click_id`.
2. **Click Ingestor retry to Kafka** — Kafka producer retries on transient failures. Without idempotent producer, can write the message twice.
3. **Dedup operator restart** — Flink job restarts from checkpoint, replays Kafka messages already processed. Without exactly-once sink semantics, downstream sees duplicates.
4. **Downstream consumer reads same offset twice** — at-least-once consumer behavior.

**The mitigations:**

| Source | Mitigation |
|---|---|
| (1) SDK retry | `click_id` UUID v4 generated at click time, idempotent on server |
| (2) Kafka producer retry | **Idempotent Kafka producer** (`enable.idempotence=true`) — sequence numbers per partition prevent duplicates from retry |
| (3) Flink restart | **Two-phase commit (TwoPhaseCommitSinkFunction)** writing to Kafka with transactions; on replay, transactions roll back partial writes |
| (4) Downstream re-consumes | Consumers must be idempotent OR read with `read_committed` isolation level from Kafka transactions |

**My pick:** Use the dedup operator approach. It's simpler than full Kafka transactions and survives operator restart cleanly:

```
Event → keyed by click_id → RocksDB lookup → emit if new → Kafka clicks.deduped
```

Because dedup is keyed on `click_id` and RocksDB state is checkpointed atomically with Kafka offsets, a Flink job restart re-consumes the messages but the dedup state already has them — they're dropped silently. Result: **at-least-once Kafka + idempotent dedup = effectively exactly-once** for the dedup output.

> **Interviewer:** "What's the dedup window? What happens to a click that arrives 25 hours late?"

**Candidate:**

The dedup state has a **24-hour TTL**. Events arriving beyond that (which should be rare — clients don't queue clicks for 24h normally) are treated as new and may double-count. We accept this as a known limitation:

- 24h covers virtually all mobile-offline scenarios.
- Beyond 24h, the click_id would have aged out of state, so we'd count it again.
- Mitigation: route events older than 1h directly to the **late-arrival pipeline** (batch reconciliation only, never real-time), which has its own dedup logic over the S3 archive.

> **Interviewer:** "Why RocksDB embedded in Flink instead of Redis?"

**Candidate:**

| | RocksDB in Flink | Redis cluster |
|---|---|---|
| Per-event latency | Local disk, ~100µs | Network hop, ~1ms |
| State recovery | Checkpointed to S3 with Flink job; consistent with offsets | External; need to re-check on restart, can desync from offsets |
| State size | 140 GB on local SSD per task slot, fine | Same data, but Redis RAM cost is 10× SSD cost |
| Operational | Flink checkpoint mechanism owns it | Separate cluster to operate, scale, monitor |
| Failover | Auto via checkpoint replay | Manual; must coordinate Redis state with Kafka offsets |

I'd pick RocksDB. The Redis approach makes sense if you want dedup state visible to multiple jobs or external introspection, but for a single dedup pipeline, RocksDB is operationally cleaner.

> **Interviewer:** "What's the failure mode if RocksDB state is corrupted or lost?"

**Candidate:**

- Detection: Flink checkpoint restore fails; job won't start.
- Recovery: replay `clicks.raw` from last *committed* Kafka offset, rebuild state. We re-process up to 24h of events.
- During replay: `clicks.deduped` may receive duplicates because dedup state is empty until rebuilt.
- Mitigation: downstream aggregator detects this via a `pipeline_health` flag and **suppresses real-time output during state-rebuild**, falling back to "data 30 min stale" indicator on dashboards.
- After rebuild: trigger a reconciliation job that compares S3 archive count vs Druid aggregate count for the affected window; emit corrections to Druid.

The honest reality: there's a brief reconciliation window after a state-loss event. The system is designed to **detect, not pretend**.

### Deep dive 2: Late-arriving events & watermarking

> **Interviewer:** "Mobile users go offline, network is flaky, sometimes clicks arrive 30 minutes late. How do you handle that without rewriting your aggregations every time?"

**Candidate:**

This is the core streaming-semantics problem. Three concepts:

1. **Event time vs processing time** — every click has `event_ts` (when user clicked) and `server_ts` (when we received it). Aggregation must be on **event time**, otherwise the dashboard sees clicks "smeared" across whenever they arrived.
2. **Watermarks** — the streaming system's heuristic for "we believe we've seen all events with event_ts ≤ T." Watermark advances as we observe new event_ts values.
3. **Allowed lateness** — a tolerance window past the watermark. Events within it update an already-emitted window; events beyond it go to a side-output.

**My choices:**

- **Watermark strategy:** `BoundedOutOfOrdernessWatermark(15 min)`. We assume 99% of events are ≤ 15 min late; this is empirical, monitored.
- **Allowed lateness:** 1 hour total. Window closes for new updates 1h after event_ts.
- **Side output:** events beyond 1 hour go to `clicks.late` topic, archived to S3, processed by **batch reconciliation only**.

**The window lifecycle:**

```
event_ts: 10:00:00
window:   [10:00, 10:01)

10:01: window first closes — emit aggregate
10:00:30 event arrives at processing_time 10:01:30 → still within watermark → updates window
10:15:00: a 14-min-late event arrives → window updates again, re-emits aggregate (Druid handles upsert)
10:16:00: watermark = 10:16, closes window for new updates beyond grace
11:00:00: 1-hour grace expires
12:00:00: an event arrives 2 hours late → routed to clicks.late, processed by daily batch
```

**Druid (and Pinot) supports replacing segments**: when an updated aggregate arrives for a window, we emit a "delta" record. Druid's roll-up indexer merges. The dashboard sees the corrected number on next refresh.

> **Interviewer:** "How do dashboards handle the fact that 'last 1 minute clicks' might keep changing for an hour?"

**Candidate:**

Two parts:

1. **Make it explicit in the API.** The `data_freshness_ts` in every response includes a "watermark" field. The dashboard renders "data current to 10:14, late arrivals through 11:14 may update older windows." Most users don't see this; power users (analytics teams) do.
2. **Stable horizon.** For *finalized* numbers (e.g., daily billing), we use a 25-hour-old aggregate — past the late-arrival window — guaranteed stable.

> **Interviewer:** "What if you set the watermark too aggressive — say 1 minute — and 5% of events arrive late?"

**Candidate:**

- Aggregates emit too early; 5% of clicks miss the real-time aggregate.
- Late-arrival side-output gets 5% of total volume, not <1%.
- Dashboard shows undercounts that drift up over the next hour as late events update.
- Advertisers complain that "clicks went up" in their reports retroactively.
- Fix: empirically tune watermark based on observed P95 of (server_ts - event_ts). Auto-tune at deploy time. We monitor `late_event_rate` and alert if > 1%.

> **Interviewer:** "What's the cost of allowing 1-hour late events?"

**Candidate:**

- Window state retained 1 hour past close, ~60× more state than non-late.
- More frequent re-emits to Druid → more segment churn.
- Trade-off: shorter lateness → cheaper, but more events are dropped. Longer → expensive, but more accurate.
- 1 hour is a typical industry choice; mobile network outages rarely exceed it. Anything beyond is in batch territory.

### Deep dive 3: OLAP query latency

> **Interviewer:** "Advertisers refresh their dashboard every 10 seconds, querying the last 7 days hourly. How does that hit p99 < 2 sec?"

**Candidate:**

The constraints:
- 7 days × 24 hr = 168 buckets
- Filter: ad_id (high cardinality, 100M ads) and/or advertiser_id (10M)
- Group by hour × maybe device_type or region
- Result: 50K queries/sec at peak across the cluster

**Druid solves this with three things:**

1. **Roll-up at ingest.** Every minute, we emit one row per (ad_id, advertiser_id, publisher_id, device_type, region, time_bucket). Multiple clicks for the same key in the same minute are summed at ingest. So the 1B clicks/day become ~10M roll-up rows/day at minute granularity, ~500K at hour granularity.
2. **Time-partitioned segments.** Druid stores ~1-day segments. Each segment is a self-contained columnar file. A 7-day query touches 7 segments.
3. **Memory-mapped recent segments + bitmap indexes.** The last 7 days is RAM-resident; bitmap index on `advertiser_id` makes filtering O(matching-bitmap) not O(scan).

For a typical query ("advertiser X, last 7 days, by hour"):
- 7 segments × 1 ms scan each = 7 ms
- Filter cardinality reduction: 1 advertiser out of 10M → bitmap is sparse, 168 hourly rows returned
- Aggregation across nodes (Druid is shared-nothing): 5 ms
- Network + serialization: 20 ms
- **Total: ~50–100 ms p99** for typical queries

**For the worst case** ("show me all 10M advertisers' last 30 days, group by hour" — an admin/internal query):
- That's 100B cells. Won't fit p99 budget.
- Solution: **deny that query at the API layer.** Force aggregation: top-1000 advertisers, or pre-roll a different cube.

> **Interviewer:** "What if the query is an unusual filter — say, by city instead of region?"

**Candidate:**

If `city` isn't a pre-aggregated dimension, the query falls through to **Trino over the S3 Iceberg raw archive**. That's slower (10s–60s) but supports ad-hoc dimensions. The API distinguishes:

- **Fast path:** pre-aggregated dimensions → Druid → < 2s
- **Slow path:** ad-hoc dimensions → Trino → < 60s, async with a "your report is generating" UX
- Power users (data analysts) get the slow path explicitly via a "Custom Report" feature.

> **Interviewer:** "Druid is RAM-hungry. What's the cost story?"

**Candidate:**

- Hot data (last 7 days at minute granularity) in RAM: ~1 TB working set across cluster.
- Druid historical nodes: r5.4xlarge × ~30 nodes = ~$50K/month.
- Cold data tiering: segments older than 30 days move to S3 deep storage; reload-on-query (slower but cheap).
- Cost lever: reduce dimensionality. Rolling up minute-granularity to hour-granularity for data older than 7 days reduces RAM by 60×.
- Cost lever: sample non-critical dimensions. We don't need full geo cardinality for daily reports — country-level rollup is enough.

### Deep dive 4: Fraud detection pipeline

> **Interviewer:** "How do you detect bot clicks?"

**Candidate:**

Architecturally, fraud is a **separate consumer of `clicks.deduped`** — it doesn't sit in the dedup hot path because:
1. Fraud scoring is heavier (ML inference, IP reputation lookups)
2. We never *delete* fraudulent clicks — we *flag* them. Flagging happens out-of-band, doesn't block aggregation.

```
clicks.deduped
     │
     ├──→ Stream Aggregator (real-time aggregate, fraud-naive)
     │
     └──→ Fraud Detection Consumer (Flink)
                 │
                 ├── velocity check (>N clicks/sec from same user) [stateful, Flink keyed]
                 ├── IP reputation (Redis lookup against blocklist)
                 ├── User-agent classifier (regex / ML)
                 ├── Click-injection signature (header anomalies)
                 └── ML model (gradient-boosted trees, online inference)
                                      │
                                      ▼
                              Fraud Scores KV (Dynamo)
                              { click_id → {score, reasons} }
```

**Why flag, not delete:**
- Auditability — billing disputes require evidence.
- Tunable threshold — accountants may want different thresholds for different markets.
- Recoverability — if our fraud model is wrong, we can re-process; deletes are forever.

**How fraud scores are used:**
- **Billing:** at end-of-day reconciliation, billing service joins aggregates with fraud scores, deducts flagged clicks above a threshold.
- **Real-time dashboards:** show "raw clicks" by default; offer "filtered clicks" toggle that subtracts above-threshold fraud.
- **Advertiser disputes:** `click_id` traceable end-to-end, with full fraud signals attached.

> **Interviewer:** "What if your fraud model has high false-positive rate?"

**Candidate:**

- Flagged ≠ deleted, so no immediate revenue loss.
- Offline evaluation: precision/recall per model version, A/B tested.
- Override mechanism: advertiser-trust scores let us tune thresholds per advertiser tier.
- Manual review queue for borderline cases (top N by score below threshold).

### Deep dive 5: Reconciliation & audit

> **Interviewer:** "How do you know your aggregate counts are right? At billing time, who's the source of truth?"

**Candidate:**

**The S3 raw archive is the source of truth.** Always. Aggregates are derived; if they disagree with raw, raw wins.

**Daily reconciliation job (Spark on the data lake):**

1. Read S3 archive for previous day → count clicks per (ad_id, hour) — call this `raw_count`.
2. Read Druid aggregates for the same window → `agg_count`.
3. Compute diff per cell: `delta = raw_count - agg_count`.
4. **Tolerance threshold:** any cell with `|delta| > 0.1%` triggers an investigation alert.
5. **Auto-correction:** for cells within tolerance, emit corrective delta to Druid.
6. **Out-of-tolerance:** page on-call, halt billing for that day pending review.

**Sources of expected divergence:**
- Late events that arrived after Druid window close but before S3 archival
- Fraud-flagged clicks (excluded from billable count, included in raw)
- Pipeline state-rebuild gaps

**Audit trail:** every billed click is traceable from `click_id` → raw S3 partition → fraud score → aggregate row. Stored for ≥ 1 year.

---

## 9. Tradeoffs & alternatives

### Decision: Kappa architecture (single streaming pipeline) vs Lambda (streaming + batch)

| Option | Pros | Cons | When |
|---|---|---|---|
| **Lambda** (real-time stream + nightly batch reconciles) | Real-time path can be lossy; batch is authoritative | Two pipelines to maintain, double the code paths | Older systems, when stream tooling was weaker |
| **✅ Kappa** (single Flink pipeline, replayable) | One code path, replay = backfill | Requires durable Kafka, sophisticated state mgmt | Modern (Flink/Kafka mature) |

**My pick:** Kappa for the real-time pipeline; a parallel Spark batch job for **reconciliation only** (not as a separate aggregation source of truth). This is the modern pattern.

### Decision: Exactly-once via Kafka transactions vs at-least-once + idempotent dedup

| Option | Pros | Cons |
|---|---|---|
| **Kafka transactional producer + Flink TwoPhaseCommitSink** | True exactly-once end-to-end | Higher latency (~100ms for transaction commit), operational complexity, transaction coordinator becomes a critical dependency |
| **✅ At-least-once + idempotent dedup keyed on `click_id`** | Simpler, lower latency, well-understood | Brief duplicates during failover until dedup state catches up |

**My pick:** at-least-once + idempotent dedup. Effectively exactly-once at the boundaries that matter (the dedup output and downstream consumers).

### Decision: Druid vs Pinot vs ClickHouse vs Snowflake

| Option | Pros | Cons | When |
|---|---|---|---|
| **✅ Druid** | Real-time ingest, sub-second queries, mature, time-series native | RAM-hungry, schema relatively rigid | Hot OLAP at known cube dimensions |
| **Pinot** | Similar to Druid, slightly better star-schema | Operationally trickier | If LinkedIn-derived ecosystem |
| **ClickHouse** | Faster ad-hoc queries, more flexible schema, lower RAM | Harder real-time ingest at this scale, ops more manual | Smaller scale, ad-hoc heavy |
| **Snowflake** | Fully managed, infinite scale | Latency floor ~seconds, expensive | Reporting, not real-time |

Druid and Pinot are very similar; pick whichever your team has muscle in. I'd default to Druid.

### Decision: RocksDB vs Redis for dedup state

Discussed in Deep Dive 1. RocksDB embedded in Flink for hot path; Redis is a viable alternative if you want external visibility into dedup state.

### Decision: 1-hour vs 24-hour late event window

| Window | Pros | Cons |
|---|---|---|
| **1 hour (real-time)** | Bounded state, predictable cost | Drops mobile-offline scenarios > 1h |
| **6 hours** | Catches most stragglers | 6× more state |
| **24 hours** | Catches almost everything | Massive state, late updates feel stale |

**My pick:** 1 hour for real-time, with a 24-hour batch reconciliation pass. Best of both.

### Decision: Per-event vs batched edge ingest

| Option | Pros | Cons |
|---|---|---|
| **Per-event** | Lowest latency to Kafka | Higher request overhead, more connections |
| **✅ Batched (50–100 events per request)** | 5× throughput, lower edge cost | Up to ~500ms batching delay |

**My pick:** batched on the SDK side with a max-size or max-time trigger. SDK keeps a 100-event buffer and flushes on either threshold. The 500ms batching delay is invisible to user (no UI consequence).

---

## 10. Defending architectural choices — interviewer attacks

> **Interviewer:** "Why not just store every click in Postgres and aggregate on-demand?"

**Candidate:** At 100K writes/sec sustained and 1B rows/day, no Postgres single primary handles this. We'd need to shard, replicate, manage failover — and aggregations across shards become distributed joins. Postgres is the wrong tool. It's right for the *advertiser metadata* (who they are, what budgets) and arguably for the fraud scores, but not for the click stream.

> **Interviewer:** "Why Kafka and not Kinesis or Pulsar?"

**Candidate:** Kafka is the most mature, has the largest ecosystem (Flink, Druid, schema registry, Connect). Kinesis is a fine alternative if we're AWS-only and want managed; the throttling at high QPS and limited retention are real constraints. Pulsar's tiered storage is appealing for our case (we want long retention + S3 tiering), but the operational maturity isn't there yet for most teams. **Default Kafka, switch to Kinesis if managed-only is a hard requirement, evaluate Pulsar for the retention story but accept it's more operational risk.**

> **Interviewer:** "Why 1024 partitions for `clicks.raw`?"

**Candidate:** Partition count = (target throughput / per-partition throughput) × headroom. 100K events/sec / ~1K events/sec/partition = 100 minimum. We round up to 1024 because:
- Powers of 2 align to consistent-hashing nicely.
- More partitions = more consumer parallelism (each Flink task slot owns N partitions).
- 1024 lets us scale consumer count up to 1024 without repartitioning.
- Cost: more memory overhead per broker, but at this scale that's noise.

If we needed 1M events/sec, we'd go to 4096.

> **Interviewer:** "What if a single ad gets a viral spike — 10× normal click rate on one `ad_id`?"

**Candidate:** In the dedup pipeline this is fine — we're keyed by `click_id`, which is uniformly distributed (UUID), so partition load stays even.

The hot spot is in the **aggregator**, where we key by `ad_id`. If one `ad_id` becomes 10% of total traffic, one Flink task gets crushed. Mitigations:
- **Sub-key sharding:** key by `(ad_id, hash(click_id) % 16)` for the first aggregation step, then merge across the 16 sub-keys at the second step. Adds latency but flattens the hotspot.
- **Detect and isolate:** monitor per-key throughput; auto-promote hot keys to a dedicated parallel pipeline.
- **Backpressure tolerance:** Flink handles backpressure gracefully, lag spikes briefly during the surge.

> **Interviewer:** "What if Druid is down?"

**Candidate:** Real-time queries fall back to "stale" mode — we serve from a Druid query-result cache (Redis, last 5 min). After 5 min, dashboards show "Real-time data temporarily unavailable" with last-known values plus historical Trino queries. We don't fail closed (no error page); we degrade gracefully. Billing is unaffected because billing reads from S3 archive, not Druid.

> **Interviewer:** "How do you handle a malicious advertiser flooding clicks on competitors' ads?"

**Candidate:** That's a fraud detection problem. Velocity rules per (user_id, ip, device_id) catch the bulk. Coordinated attacks across many devices (click farms) need ML detection on patterns: similar UA fingerprints, anomalous geographic clustering, suspiciously regular timing. The fraud scores feed into billing — flagged clicks are deducted from chargeable clicks. We don't *block* ingestion (it's evidence), we *score* it.

> **Interviewer:** "What if an advertiser claims they're being charged for clicks that didn't happen?"

**Candidate:** Audit trail. Every billed click is traceable: `click_id` → S3 raw partition (with timestamp, IP, user_agent, all context) → fraud score (with reasons) → aggregate row. We hand the advertiser the raw events behind their bill. If our pipeline is correct, the evidence is there. If we have a bug, reconciliation catches it before billing finalizes.

---

## 11. Failure modes & mitigations

| Failure | Detection | Blast radius | Mitigation | Recovery |
|---|---|---|---|---|
| Click Ingestor down (regional) | 5xx rate, edge health probe | Ingest fails for that region | Edge anycast routes to next-nearest region (latency penalty) | Auto-failover; on-call investigates |
| Kafka broker loss | ISR shrink, under-replicated partitions | None (RF=3) | Other brokers take over; new replica streams in | Replace node, rebalance |
| Kafka cluster lost (catastrophe) | Producer error rate at edge | Ingest fully blocked | **Click Ingestor buffers in local WAL on disk for up to 1 hour**; replays on Kafka recovery | Multi-AZ Kafka prevents this; standby cluster failover < 10 min |
| Flink dedup job restart | Job manager metric | Brief duplicate output until state replays | Dedup state is checkpointed; replay from offset, dedup catches dups quickly | Auto-recovery from checkpoint, ~30s |
| Flink dedup state lost | Checkpoint restore failure | 24h of potential dup output | Replay clicks.raw from last committed offset; downstream tolerates briefly via reconciliation | Rebuild state, run reconciliation job |
| Late event beyond window | `late_event_count` metric | Event dropped from real-time aggregate | Routed to clicks.late, picked up by daily batch reconciliation | Reconciliation emits delta to Druid |
| Druid ingestion lag | Druid `realtime_lag` metric | Real-time dashboards stale | `data_freshness_ts` shows lag explicitly; queries still work | Auto-scale ingestion tasks; investigate root cause |
| Druid query overload | Query queue depth | Dashboards slow | Per-tenant rate limit; query result cache absorbs duplicates | Horizontal scale Druid brokers; throttle abusers |
| Schema mismatch (producer/consumer) | Schema registry rejects | Pipeline halts | Schema registry enforces backward compat at registration time; never deploy incompatible schema | Roll back producer; never roll out without compat check |
| Hot ad partition (viral content) | Per-key throughput skew | Aggregator slow for that ad | Sub-key sharding; auto-detect + isolate hot keys | Documented runbook |
| Fraud detection consumer down | Lag metric | Fraud scores stale; billing uses last-known | Aggregation independent; billing waits for fraud catch-up before finalizing | Replay from clicks.deduped |
| S3 archive write fails | Archive consumer error metric | No long-term raw retention for that period | Retry with exponential backoff; dead-letter to backup S3 path | Reconcile from Kafka if within 7-day retention |
| Cross-region replication lag | Replicator metric | Global aggregates incomplete | Per-region aggregates always available; global merges when caught up | Auto-recover when replication restored |
| DDOS / click bombing | Edge rate limit metric | Could overwhelm Kafka | WAF + per-IP rate limit at edge; per-`ad_id` rate limit at Click Ingestor | Auto-block; route attackers to tarpit |
| GDPR delete request | Compliance ticket | Must remove user data from raw + aggregates | Tombstone in user_id space; sweep S3 archive; aggregates are non-PII | SLA: 30 days |

---

## 12. Productionization

### Pre-launch
- [ ] Dual pipeline: run new pipeline in parallel with existing aggregation for 2 weeks. Daily diff report; investigate any divergence.
- [ ] Load test to 3× peak: 300K clicks/sec sustained, with periodic 5× bursts.
- [ ] Chaos test: kill random Flink task managers, Kafka brokers, Druid historical nodes during load.
- [ ] Schema migration drill: deploy a producer with new optional field, verify all consumers still work.
- [ ] Late-event injection test: replay events with synthetic delays 5min, 30min, 2hr; verify routing.
- [ ] Reconciliation test: inject known dups and known drops; verify reconciliation catches them.

### Rollout
- [ ] Feature flag at the Click Ingestor: route % of traffic to new pipeline.
- [ ] 1% → 5% → 25% → 100%, with 24h bake at each step.
- [ ] Per-region rollout: smallest region first.
- [ ] Auto-rollback if: ingest p99 > 200ms, dedup_dup_rate change > 0.5%, late_event_rate > 1%, Druid ingest lag > 5 min for 10 min.

### Capacity
- [ ] Kafka: 50 brokers, sized for 5× peak. Repartitioning is expensive — provision generously.
- [ ] Flink: 1000 task slots across cluster, target 50% CPU at peak (room for backfill burst).
- [ ] Druid: provision for 7-day hot working set + 30% headroom.
- [ ] S3: lifecycle policies — 30-day standard, then Glacier tier.
- [ ] Auto-scale Click Ingestor on QPS, target 60% CPU.

### Migration (replacing existing system)
- [ ] Dual-write to old + new for 30 days.
- [ ] Comparison job: hourly aggregates from both; investigate any divergence > 0.1%.
- [ ] Cut over read traffic on dashboards gradually (per-advertiser feature flag).
- [ ] Old system stays for 30 days post-cutover for rollback insurance.
- [ ] Decommission old; archive its data to S3 cold storage.

### Cost
- Estimated $/click: ~$0.0001/click (compute + storage amortized).
- Largest cost: Druid RAM and Kafka disk.
- Cost levers:
  - Roll up older data to coarser granularity.
  - Tier old segments to S3 deep storage.
  - Sample fraud-scored events at finer resolution; sample clean events more aggressively for analytics.
  - Compress on Kafka with ZSTD (5–10% savings).

### Compliance
- GDPR: user_id pseudonymized at edge; raw archive has no email/phone. Delete requests sweep tombstone-mark in user_id directory.
- Data residency: per-region cells (see section 19); EU data never leaves EU.
- PCI: no card data ever touches this pipeline (it's an ad event pipeline, not billing).
- Audit log: every admin action (schema change, threshold tweak, replay) logged.

---

## 13. Monitoring & metrics

### Business KPIs
- Total clicks / day, per advertiser, per publisher
- CTR (clicks / impressions)
- Fraud-flagged rate
- Advertiser dashboard sessions / day
- Billing reconciliation accuracy (target: 100% within tolerance)

### Service-level (SLIs)

- **Ingest** (the front door): p50, p99 latency; error rate; per-region.
- **End-to-end freshness:** click event_ts → visible on dashboard. p50, p99.
- **Dashboard query:** p50, p99, p99.9 latency.
- **SLO:**
  - 99.99% of click writes ack in < 200ms
  - 99% of clicks visible on dashboard in < 1 min
  - 99.9% of dashboard queries < 2 sec p99
  - 100% of daily reconciliation within 0.1% tolerance

### Component metrics

#### Click Ingestor
- Request rate, error rate, p99 latency
- Kafka publish latency, error rate
- Validation rejection rate (malformed, replay-attack)
- Per-region throughput, edge cache hit rate

#### Kafka
- Producer rate, error rate
- Consumer lag per group (alert if > 30s on hot consumers)
- Broker disk/network/CPU utilization
- ISR shrink/expand rate
- Per-partition throughput (detect skew)

#### Flink dedup
- Throughput in/out
- **Dedup ratio** (dups dropped / total events) — this is a key health metric; if it spikes, something is replaying
- RocksDB state size, compaction rate, get/put latency
- Checkpoint duration (alert if > 60s)
- Watermark progress (alert if stalled > 1 min)

#### Flink aggregator
- Throughput in/out
- Watermark lag (event time vs processing time)
- **Late event rate** (alert if > 1%)
- Window emit rate, window state size
- Per-key skew

#### Fraud Detection
- Throughput, scoring latency
- Score distribution (drift detection)
- False positive / negative rate (offline eval)
- ML feature staleness

#### Druid
- Real-time ingest lag (alert if > 5 min)
- Segment count, RAM utilization
- Query p50, p99 by query class (filter heavy vs scan heavy)
- Historical-vs-real-time split (load balance)
- Failed segment loads

#### S3 archive
- Write rate, error rate
- Partition count per day (sanity check)
- Reconciliation diff per day

#### Dashboard query API
- p50, p99 latency by query class
- Cache hit rate (Redis result cache)
- Query QPS per advertiser (rate limiting)

### Alert routing

- **Page** (24/7): ingest p99 > 500ms for 5 min; Kafka cluster ISR < 2 for any partition; Flink dedup job down; Druid no-ingest > 10 min; reconciliation > 1% diff.
- **Ticket** (next business day): late event rate creeping up; Druid query p99 degrading; cache hit rate drop.
- **Dashboard-only:** cost trends, throughput trends, capacity headroom.

### Dashboards
1. **Pipeline health** — RED metrics across ingest, dedup, aggregate, OLAP. End-to-end freshness front and center.
2. **Dedup deep-dive** — dup rate, RocksDB state, checkpoint duration, late events.
3. **OLAP health** — Druid ingest lag, query latency by class, segment counts, RAM.
4. **Reconciliation** — daily raw-vs-aggregate diff, fraud rates.
5. **Business** — clicks/day, CTR, advertiser activity.

---

## 14. Security & privacy

- **PII minimization:** user_id is pseudonymized at edge (one-way hash); raw archive contains no email, phone, address.
- **AuthN/Z:** advertiser dashboard requires OAuth + per-advertiser scope; query API enforces "you can only query ads you own."
- **Rate limiting:** at edge (per-IP, per-API-key) to prevent flood; per-advertiser query rate to prevent dashboard abuse.
- **Audit logging:** every admin action (schema change, threshold tweak, replay, reconciliation override) logged with actor, timestamp, before/after state. Retained 1 year.
- **PCI:** out of scope — billing systems handle card data, not us. We hand them aggregates only.
- **GDPR:** right-to-erasure path: tombstone user_id in directory; nightly sweep of S3 archive marks records; aggregates are non-PII (counts, no user identifiers exposed).
- **Encryption:** TLS in transit (mTLS service-to-service), at-rest on Kafka (LUKS) and S3 (SSE-KMS).
- **Bot detection:** WAF + edge bot detection rejects obvious bot traffic before Kafka. Sophisticated bot detection happens in the fraud pipeline.

---

## 15. Cost analysis

| Component | Cost driver | Optimization lever |
|---|---|---|
| Kafka brokers | Disk (RF=3 × 7-day retention) + network | Compression (ZSTD); shorter retention if reprocessing window can shrink |
| Flink compute | CPU-hours | Right-size operators; tune parallelism to lag, not headroom |
| RocksDB state | SSD | 24h TTL hard cap; partition state across many task slots |
| Druid | RAM + disk | Tier old segments to S3 deep storage; coarsen older data |
| S3 archive | Storage + egress | Lifecycle policies (Glacier after 30 days); columnar compression |
| Trino compute | CPU-hours when queried | On-demand (Athena-style); no cost when idle |
| Fraud ML inference | GPU/CPU | Batch where possible; quantize models; skip clean clicks (high-confidence non-fraud) |
| Egress (cross-region replication) | Bytes | Compress; only replicate aggregated data, not raw, for global views |

Largest cost driver at this scale: usually **Druid RAM** (hot OLAP) + **Kafka disk** (raw durability). Engineering lever: aggressive tiering and rollup. A 60-day-old click is fine to coarsen to hourly granularity; saves 60× space.

Rough $/click target: ~$0.0001/click amortized. At 1B clicks/day = ~$100K/day, ~$36M/year. Sounds high; for FAANG ad business pulling in tens of billions/year in ad revenue, infra is < 1% of revenue.

---

## 16. Open questions / what I'd validate with PM

1. **Is sub-second freshness ever actually required?** A power-user feature like real-time bid optimization might want 1-sec, but most advertisers tolerate 1-min. Drives whether we invest in sub-second OLAP path.
2. **What's the fraud P&L impact?** If fraud is 10% of clicks and we're under-counting, advertisers will sue. If it's 0.1%, the engineering investment threshold differs.
3. **Multi-region: are we required to keep EU data in EU even for global advertisers?** Affects cross-region pull architecture (see section 19).
4. **What's the schema-evolution velocity?** If we add 1 dimension/quarter, schema registry is enough. If 10/quarter, we need a more flexible model (event-as-JSON, schema-on-read).
5. **Are publishers also dashboard customers?** They have different access patterns (see *their* ad's clicks, not all ads). Affects data model and authz.
6. **What's the SLA on backfill?** "Replay last week's data after pipeline bug" — is that a 1-day operation or a 1-hour one? Drives Kafka retention and Flink savepoint design.
7. **Mobile SDK behavior on offline:** does the SDK queue clicks for hours or drop them? Drives our late-event window.

---

## 17. Staff-level scorecard for this problem

| Signal | Did the candidate... |
|---|---|
| ✅ Identified exactly-once semantics as central, not afterthought | Within first 10 min, named the dedup approach |
| ✅ Quantified late-event rate impact | Made the watermark/lateness tradeoff explicit with numbers |
| ✅ Picked tools with reasoned tradeoffs | Druid vs Pinot vs ClickHouse argued, not "we'd use Druid" |
| ✅ Treated fraud detection as separate consumer, not inline | Did not couple fraud to dedup hot path |
| ✅ Volunteered reconciliation / audit before being asked | "S3 raw is source of truth for billing" stated explicitly |
| ✅ Discussed schema evolution | Proto + schema registry + backward-compat |
| ✅ Cost-aware | Mentioned $/click, identified Druid RAM as dominator |
| ✅ Productionization plan | Dual pipeline, gradual rollout, reconciliation diff |
| ✅ Failure modes proactive | Kafka loss, dedup state loss, hot keys, schema mismatch |
| ✅ Asked the interviewer at decision points | "Should I deep-dive dedup or late events first?" |
| ✅ Closed with a wrap-up speech | Restated the constraints and choices |

### What separates Staff from Senior on *this* problem

A Senior candidate will:
- Correctly draw Kafka + Flink + Druid + S3
- Mention dedup with idempotency keys
- Say "at-least-once with idempotent consumer"

A Staff candidate will additionally:
- Decompose the four sources of duplication and address each independently
- Make watermark and allowed-lateness an empirically-tuned config, not a hardcoded constant
- Explicitly route late events to a side-output, not just "drop them"
- Treat S3 raw archive as source of truth for billing — not Druid, never Druid
- Build reconciliation into the design, not as a post-launch project
- Identify the hot-key skew problem in the aggregator (not the dedup) and propose sub-key sharding
- Volunteer the schema evolution story before being asked
- Distinguish Lambda from Kappa and pick Kappa with reasoning
- Discuss cost with concrete dollar numbers and a lever per cost line

---

## 18. Wrap-up speech (template)

> "To wrap up: the dominant constraints are exactly-once correctness on counted output (because clicks are revenue) and late-arriving events (because mobile is unreliable). The architecture answers both: client-generated `click_id` as idempotency key, dedup via Flink with RocksDB state, Kappa-style streaming with Kafka + Flink + Druid, and a side-output for late events with daily Spark reconciliation. The two pieces I'd worry about in production are (1) dedup state loss during a Flink failure — mitigated by 30-second checkpoints and a reconciliation job that catches the brief duplicate window — and (2) hot-key skew on viral ads — mitigated by sub-key sharding and dynamic detection. To productionize: dual-pipeline against the existing system for 30 days, compare daily aggregates within 0.1% tolerance, gradual rollout per region. SLO-alert on ingest p99, dedup ratio, and late-event rate. The S3 raw archive is the source of truth for billing — Druid is a derived view, never the system of record. If we had another 30 minutes I'd deep-dive (a) the fraud detection ML feature pipeline, or (b) cross-region cellular architecture for global advertisers."

---

## 19. Market-based isolation (cellular architecture) — bonus / "if time permits"

> Bring this up in the last 5 minutes. For an ads pipeline this is unusually high-leverage because of GDPR and data residency.

### What & why

A **cell** is a per-region full-stack replica of the ingest + dedup + aggregation + OLAP serving stack. Cells are independent failure domains.

Drivers (priority order for an ads pipeline):

1. **Compliance / data residency** — GDPR forbids EU click data leaving EU. India's DPDP Act, China's PIPL similar. **Hard requirement, not negotiable.** This alone forces cellular for an ads pipeline operating globally.
2. **Latency** — advertisers in Europe querying their dashboards should hit EU OLAP, not trans-Atlantic.
3. **Blast radius** — a Flink job loop in the US cell shouldn't take down EU billing.
4. **Operational independence** — different regions have different schema versions during rollout, different feature flags, different ML models.
5. **Per-region scaling** — Brazil traffic at 5 PM doesn't fight US East cluster capacity.

### Architecture

```
                        ┌──────────────── Global Edge (CDN + Anycast) ─────────────────┐
                        │   GeoDNS routes click to nearest regional cell                │
                        └───────────┬───────────────┬───────────────┬───────────────────┘
                                    │               │               │
                       ┌────────────▼────┐ ┌────────▼─────┐ ┌──────▼────────┐
                       │   US Cell       │ │  EU Cell     │ │  IN Cell      │
                       │ ┌─────────────┐ │ │ ┌──────────┐ │ │ ┌──────────┐  │
                       │ │ClickIngestor│ │ │ │Ingestor  │ │ │ │Ingestor  │  │
                       │ │Kafka        │ │ │ │Kafka     │ │ │ │Kafka     │  │
                       │ │Flink Dedup  │ │ │ │Dedup     │ │ │ │Dedup     │  │
                       │ │Aggregator   │ │ │ │Agg       │ │ │ │Agg       │  │
                       │ │Druid        │ │ │ │Druid     │ │ │ │Druid     │  │
                       │ │S3 Archive   │ │ │ │S3 (EU)   │ │ │ │S3 (IN)   │  │
                       │ └─────────────┘ │ │ └──────────┘ │ │ └──────────┘  │
                       └────────┬────────┘ └─────┬────────┘ └──────┬────────┘
                                │                │                  │
                                └────────────────┴──────────────────┘
                                                 │
                                  ┌──────────────▼───────────────┐
                                  │   Global Aggregation Layer   │
                                  │   - Per-cell aggregates      │
                                  │     pushed (anonymized)      │
                                  │   - Global advertiser dashb. │
                                  │     merges aggregates        │
                                  │   - NEVER replicates raw     │
                                  │     PII data                 │
                                  └──────────────────────────────┘
```

### Routing: how a click lands in the right cell

- Edge routes click by **user's geographic region** (from IP / device).
- Click stays in that region for the entire pipeline. Raw events never cross borders.
- Advertisers register their *primary* region; cross-region advertisers (most large ones) have separate cell-bound accounts that are merged at the aggregation layer.

### The hard part: global aggregates

A multinational advertiser wants "my CTR across all regions, last 7 days, by country."

Three approaches:

| Approach | How | Pros | Cons |
|---|---|---|---|
| **Replicate raw clicks globally** | Copy all clicks to a global cell | Easiest queries | **Violates GDPR**, massive storage cost, cross-region bandwidth |
| **Replicate aggregates globally (anonymized)** | Each cell pushes pre-aggregated rows (no user_id) to a global Druid cluster | GDPR-safe, low bandwidth | Less flexible — only pre-aggregated dimensions queryable globally |
| **Cross-cell federated query** | Query API fans out to per-cell Druids and merges | No replication, fully fresh | Latency = max of all cells, fragile to cell outage |

**My pick for ad clicks:** **anonymized aggregate replication** — the cell-level aggregates (which are non-PII counts) are pushed to a global aggregate store. Advertisers querying global dashboards hit the global store. Advertisers querying single-region dashboards hit the regional cell directly.

This preserves GDPR (raw data never crosses) while supporting the 80% case of pre-aggregated global queries.

### Cell tradeoffs vs single global system

| Dimension | Single global | Cellular | Net |
|---|---|---|---|
| Total infra cost | Lower | +30% (replicated stacks) | Cellular tax |
| Engineering complexity | Lower | +significant (cell awareness everywhere) | Cellular tax |
| GDPR compliance | Hard / impossible | Easy | **Cellular wins (hard requirement)** |
| Click ingest latency | Higher (cross-region) | Lower (local) | Cellular wins |
| Global advertiser dashboards | Easy | Requires global aggregation layer | Single wins on simplicity |
| Blast radius | 100% | per-cell | Cellular wins |
| Schema deploy velocity | Single rollout | Per-cell, slower | Single wins |

For an ads pipeline at FAANG scale, **cellular is non-optional** because of regulatory pressure. The cost is real; the alternative (global single pipeline) is illegal in EU.

### What changes in the design

- **No global Kafka.** Each cell has its own Kafka cluster.
- **Schema registry is per-cell**, with a global schema steward to keep cells aligned.
- **Click_id collision space is shared** (UUIDs are universally unique), so a user clicking from EU and re-appearing in US still gets dedup'd within their cell — and across cells, if aggregates merge later, dedup is per-region (acceptable: regional totals don't double-count, global is rolled up from regions).
- **Fraud detection is per-cell** with cross-cell ML model sharing (model trained on global anonymized features, deployed per-cell).
- **Reconciliation is per-cell**, with a final cross-cell merge job for global aggregates.
- **Failover policy:** if EU cell is down, do we route EU clicks to US cell? **No** — that violates residency. The cell IS the failure domain; intra-cell HA protects users.

### Operational implications

- **Deploy pipelines:** per cell, gradual cell-N first, observe, then N+1.
- **On-call:** per-cell rotation, plus a "global aggregation" oncall.
- **Capacity:** independent per cell; India growth ≠ US growth.
- **Cost dashboards:** $/click per cell — varies (US is cheap, EU is more expensive due to compliance overhead, IN is cheaper compute but pricier compliance reviews).
- **Schema migrations:** staggered, with cell-N stable while N+1 migrates.

### When NOT to do cellular

- Single-region product (e.g., US-only ad network).
- Pre-scale: under ~10K events/sec, the operational tax exceeds the benefit.
- No regulatory pressure (rare for ads — almost always present).
- Single team operates everything (cellular requires per-cell ops capacity).

### What "Staff signal" sounds like here

> "Cellular isn't an optimization for an ads pipeline at FAANG scale — it's a regulatory requirement. The interesting question isn't whether to do cellular, it's how to handle global aggregates without replicating raw data. I'd push pre-aggregated, anonymized rows to a global aggregate store and forbid raw replication. Advertisers querying a single region hit the regional cell; querying global hits the aggregate store. The 30% infra tax and the engineering complexity surcharge are the price of admission. The alternative is a regulatory finding that costs hundreds of millions, so the math always works out."

---
