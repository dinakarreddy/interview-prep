# 12 — Design a Live Comments / Live Streaming Chat (Twitch / YouTube Live / Facebook Live)

> Canonical Staff-level "fanout-extreme" problem. The interviewer cares less about the answer and more about whether you correctly identify that **fanout amplification at concurrent-viewer scale** and **server-side sampling** are the only things that matter, and you spend your time there. If you spend 20 minutes designing the comment ingest path and 5 on fanout, you scored Senior.

---

## 0. Problem statement (as the interviewer would drop it)

> **Interviewer:** "Design the live chat / live comments system for a streaming platform like Twitch or YouTube Live. Viewers watching a stream can post comments in real-time, and every other viewer of that stream sees those comments appear. The video itself is not in scope — assume that's solved. Walk me through how you'd build it."

That's it. Your job: scope it, not solve it whole. Recognize within 60 seconds that this is **not a "messaging" problem**. It is a **fanout amplification problem** — the worst-case fanout factor in this design dwarfs the news feed problem by 3–4 orders of magnitude.

---

## 1. Clarifying questions

Pick 5–6. Don't fire them as a checklist — interleave with the interviewer's responses.

> **Candidate:** "Before I design, let me scope this. The headline question for me is what the largest-stream fanout factor is, because that drives almost every choice."
>
> 1. **"What's the typical and peak concurrent viewer count for a single stream?"** — *Expect: typical streamer 10K, mid-tier 100K, top 1M+, and event-class (Super Bowl, World Cup) 10M+ on a single stream.*
> 2. **"What's the comment posting rate per stream?"** — *Expect: scales sub-linearly with viewers — small stream 10/sec, big stream 1K–10K/sec. Engagement caps at the rate humans can read.*
> 3. **"How many concurrent streams platform-wide, and total concurrent viewers?"** — *Expect: 10M concurrent viewers across all streams, 1M comments/sec aggregate write rate.*
> 4. **"Is strict ordering required, or is best-effort acceptable?"** — *Critical. At 10M concurrent viewers per stream, individual ordering becomes meaningless. Push back: best-effort is the only option.*
> 5. **"What moderation features are in scope — banned words, mod delete, slow mode, sub-only chat?"** — *All of these are P0 in real products. Scope them in.*
> 6. **"Are pinned messages and reactions in scope?"** — *Yes — they have a different reliability tier than regular comments and need their own delivery channel.*
> 7. **"What about VOD / replay — when someone watches the stream later, do they see the live comments replayed at the right timestamps?"** — *Important secondary requirement. Drives the storage decision.*
>
> **Out of scope (state explicitly):**
> - The video stream itself (delegate to the YouTube/Twitch streaming HLD)
> - Monetization: subscriptions, bits, donations (mention as integration points)
> - Direct messages between viewers
> - Search across past chats
> - Authentication / login
> - Channel page / profile / VOD storage of the video itself
>
> **Candidate:** "I'm going to design the **comment post → real-time fanout** path for streams up to 10M concurrent viewers, plus moderation, pinned messages, and slow mode. Replay-on-VOD I'll cover at the end as a storage variant. Sound right?"

> **Interviewer:** "Yes."

---

## 2. Functional requirements

| Priority | Requirement |
|---|---|
| P0 | Viewer can post a text comment to the live stream they're watching |
| P0 | Viewer sees comments from other viewers in near-real-time (sub-second perceived) |
| P0 | Comments fan out to all viewers of the stream (with sampling at extreme scale) |
| P0 | Synchronous moderation: banned-word filter, length limit, profanity filter |
| P0 | Async moderation: ML toxicity classifier; mods can delete comments retroactively (delete must propagate to all viewers within seconds) |
| P0 | Slow mode: per-stream rate limit on viewer comment cadence (e.g., 1 comment per 5s per viewer) |
| P0 | Follower-only / sub-only chat mode: permission check at submit |
| P0 | Pinned messages: streamer/mod pins a comment, all viewers see it (reliable, not sampled) |
| P1 | Reactions / emotes (lightweight engagement signal — different fanout tier) |
| P1 | Per-stream kill switch (lock chat instantly) |
| P1 | Mod delete propagates retraction to all viewers |
| P2 | VOD replay: comments stored with stream timestamp, re-fanned at correct offset on VOD playback |
| Out | DMs, search, video itself, monetization, authentication |

---

## 3. Non-functional requirements

| Dimension | Target | Justification |
|---|---|---|
| Availability | 99.95% for chat read; 99.9% for chat post | Chat is core to live experience; brief outage for posters tolerable, brief outage for readers ruins the broadcast |
| End-to-end latency | p50 < 500ms, p99 < 2s (post → visible to other viewers) | Live chat has to feel "live"; 2s lag is the upper bound before users notice |
| Post acceptance latency | p99 < 200ms | Submitter needs immediate confirmation |
| Consistency | Best-effort per-stream ordering; no global ordering; sampling at extreme scale | Strict ordering is meaningless when viewers see different sampled subsets |
| Durability | Best-effort for normal comments; **strict** for pinned messages and mod actions | Sampling already implies regular comments are not durable to all viewers |
| Scale | 10M concurrent viewers platform-wide; 1M comments/sec aggregate; single stream peaks at 10M concurrent viewers, 10K comments/sec | Industry-standard event scale (Super Bowl streaming) |
| Geography | Global; per-region cells; data residency for EU |
| Moderation latency | Sync filter: <50ms inline; async ML: <30s to retraction |

The non-negotiable is **end-to-end latency** — chat that's 5 seconds behind the video feels broken. The non-negotiable is **not** strict ordering or full delivery, and you should call this out explicitly: "I am going to relax delivery semantics to make latency hold."

---

## 4. Capacity estimation

> **Candidate:** "Let me estimate. The interesting numbers here are the fanout amplification factors — that's where the design pivots."

### Viewers and streams
- 10M concurrent viewers across all streams platform-wide.
- ~100K concurrent streams, but heavily skewed: 99% of streams have <1K viewers, 1% have ≥10K.
- Top 100 streams account for ~50% of all viewers.
- Event-class stream peak: 10M concurrent on a single stream (Super Bowl, World Cup final).

### Comment write rate
- Aggregate: ~1M comments/sec across all streams at peak.
- Per-stream comment rate scales sub-linearly with viewers — there's a human ceiling on what's readable. Empirically:
  - 10K-viewer stream: 10–100 comments/sec.
  - 1M-viewer stream: 1K–5K comments/sec.
  - 10M-viewer stream: 5K–10K comments/sec (chat becomes a "wall of text" beyond this).

### Fanout amplification (this is the punchline)
- One comment in a 1K-viewer stream = 1K deliveries.
- One comment in a 1M-viewer stream = 1M deliveries.
- 10K comments/sec × 1M viewers = **10B deliveries/sec on a single stream**.
- 10K comments/sec × 10M viewers = **100B deliveries/sec on a single stream**.
- **This number cannot be served. We have to sample.**

This single calculation drives every architectural choice. State it explicitly within the first 10 minutes.

### Storage
- Comment record: ~200 bytes (id, stream_id, user_id, text ≤500 chars compressed, timestamp, flags).
- 1M comments/sec × 86,400 = **86.4B comments/day**.
- 86.4B × 200 B = **~17 TB/day** of raw comments.
- Replication 3×: **~50 TB/day**.
- VOD retention (90 days): ~4.5 PB.
- Hot storage (Redis pub/sub + recent buffer): ~few GB per active stream.

### Connection count
- 10M concurrent WebSocket connections.
- Per-connection memory: 10–50 KB (WS frame buffers, auth state, sub state) → ~500 GB aggregate.
- At 10K connections/host (typical for a tuned WS gateway), **need ~1000 gateway hosts** just for connections. Multi-region multiplies this.

### Bandwidth
- Outbound to viewers: 200 B per comment delivered.
- 10M concurrent viewers receiving sampled stream of ~50 comments/sec = 500M deliveries/sec × 200 B = **100 GB/s aggregate egress** even with sampling.
- Without sampling on the largest streams, this is 10× higher.

### The math collapses to four design forcing functions:
1. **We cannot deliver every comment to every viewer at extreme scale → server-side sampling is mandatory.**
2. **Connection memory dominates infra cost → must be horizontally scaled and tightly packed.**
3. **Single-stream pub/sub will saturate any single broker → hierarchical (tree) fanout is required for top streams.**
4. **Strict ordering is meaningless under sampling → relaxed delivery semantics from day one.**

---

## 5. API design

> **Candidate:** "Three surfaces: client-facing WebSocket protocol for real-time, REST for control plane, and an internal API for moderators."

### Client real-time (WebSocket)

```
WS  /v1/chat/connect?stream_id={id}&token={jwt}
```

**Server → client frames:**
```
{ type: "comment", id, user_id, display_name, text, ts, flags{sub,mod,donation} }
{ type: "delete", id }                 // mod retracted a comment
{ type: "pin", id, text, expires_at }  // pinned message
{ type: "unpin", id }
{ type: "system", code, message }      // slow mode on, sub-only on, chat locked, etc.
{ type: "reaction_burst", emote_id, count, window_ms }  // aggregated reactions
{ type: "ping" }
```

**Client → server frames:**
```
{ type: "post", local_id, text }       // local_id is client idempotency token
{ type: "react", emote_id }
{ type: "pong" }
```

Server replies to a `post` with:
```
{ type: "post_ack", local_id, server_id, ts }       // accepted
{ type: "post_reject", local_id, reason }           // slow mode, banned, sub-only, etc.
```

### REST control plane

```
POST /v1/streams/{id}/chat/mode   { slow_mode_seconds, follower_only, sub_only }
POST /v1/streams/{id}/chat/lock   { locked: true|false }
POST /v1/streams/{id}/chat/pin    { comment_id }
DELETE /v1/streams/{id}/chat/pin
POST /v1/streams/{id}/chat/banned-words   { words: [...] }
DELETE /v1/streams/{id}/chat/comments/{comment_id}     // mod delete
POST /v1/streams/{id}/chat/timeout         { user_id, duration_seconds }
POST /v1/streams/{id}/chat/ban             { user_id }
```

### VOD replay
```
GET /v1/streams/{id}/vod/comments?from_offset_ms={t}&to_offset_ms={t}&limit=500
→ 200 { comments: [{ id, user_id, text, offset_ms, flags }], next_offset_ms }
```
Used by VOD player to render comments at the right point in playback.

### Anti-patterns to avoid
- Don't make `POST /comment` a separate REST call — round-trip kills the live feel; piggyback on the existing WS.
- Don't use long-polling for delivery; WS is non-negotiable at this scale.
- Don't return all comments in the WS connect response; return last N (e.g., 50) for context, then stream forward.

---

## 6. Data model

### `comments` (Cassandra; partitioned by `stream_id`, clustered by `comment_id` (Snowflake-time-sortable))

| Column | Type | |
|---|---|---|
| stream_id | bigint | partition key |
| comment_id | bigint (Snowflake) | clustering key (time-sortable) |
| user_id | bigint | |
| display_name | text | denormalized for read speed |
| text | text (≤500) | |
| flags | int (bitmap) | sub, mod, donation, sampled-out, deleted |
| stream_offset_ms | bigint | for VOD replay alignment |
| created_at | timestamp | |
| ttl | int | 90 days for VOD retention |

Why Cassandra: write-heavy (1M+ writes/sec aggregate), partitioning by `stream_id` localizes writes and reads per stream, time-clustered reads support VOD replay range scans, eventual consistency is fine.

### `pinned_messages` (Redis + Cassandra backup)
| Column | Type | |
|---|---|---|
| stream_id | bigint | PK |
| comment_id | bigint | |
| text | text | |
| pinned_by | bigint | |
| pinned_at | timestamp | |
| expires_at | timestamp | |

Pinned messages get their own table because they're delivered reliably (not sampled) and need to be available on connect (every new viewer should see the current pin).

### `chat_settings` (Redis hot, Cassandra cold)
| Column | Type | |
|---|---|---|
| stream_id | bigint | PK |
| slow_mode_seconds | int | |
| follower_only | bool | |
| sub_only | bool | |
| locked | bool | |
| banned_words | set<text> | |
| updated_at | timestamp | |

Cached at every gateway with 5-second TTL or change-notification invalidation.

### `slow_mode_state` (Redis only — ephemeral)
- Key: `slowmode:{stream_id}:{user_id}` → last_post_timestamp.
- TTL: max(slow_mode_seconds × 2, 60s).
- Checked at gateway on every post; rejected if violation.

### `user_bans` (Redis hot, Cassandra cold)
- Key: `ban:{stream_id}:{user_id}` → ban_metadata.
- Checked at gateway on connect AND on every post.

### Why these choices
- **Cassandra** for `comments`: write throughput, time-series partition pattern is a textbook fit. Alternative (DynamoDB) costs too much at 1M writes/sec; ScyllaDB would also work and is what Discord uses for messages.
- **Redis** for hot state (slow-mode counters, settings, pin, bans, pub/sub): O(1) lookups, in-memory speed, expiration semantics built in.
- **Snowflake IDs** for `comment_id`: time-sortable, lets us range-scan VOD comments by id without a separate timestamp index.

---

## 7. High-level architecture

```
                  ┌─────────────────────────────────────────────────────────┐
                  │                  Edge / WS Load Balancer                │
                  │     (Anycast, sticky by stream_id hash, mTLS)           │
                  └────────────────────────┬────────────────────────────────┘
                                           │
                                           │ WSS
                                           ▼
                  ┌──────────────────────────────────────────────────────────┐
                  │       Connection Gateway (sticky by stream_id)           │
                  │   - holds 10K WS conns/host                              │
                  │   - subscribes to stream's pub/sub topic                 │
                  │   - applies per-viewer backpressure (drop on slow conn)  │
                  │   - applies sampling (drop fraction for big streams)     │
                  │   - caches stream chat-settings (5s TTL)                 │
                  └──────┬─────────────────────────┬─────────────────────────┘
                         │ post                    │ subscribe
                         ▼                         ▼
              ┌────────────────────┐     ┌─────────────────────────────────┐
              │  Comment Ingest    │     │   Pub/Sub Fanout Mesh           │
              │  Service           │     │  (Redis Streams / Kafka /       │
              │ - validate text    │     │   proprietary mesh)             │
              │ - moderation sync  │     │                                 │
              │ - slow-mode check  │     │   For top streams: hierarchical │
              │ - persist to       │     │   tree:                         │
              │   Cassandra        │     │      root publisher             │
              │ - publish to       │     │           │                     │
              │   stream's topic   │     │      tier-1 (10–100 nodes)      │
              └────┬─────────────┬─┘     │           │                     │
                   │             │       │      tier-2                     │
                   │             │       │           │                     │
                   │             │       │      leaves (gateways)          │
                   │             │       └──────┬──────────────────────────┘
                   ▼             │              │
              ┌──────────┐       │              │
              │Cassandra │       │              │
              │comments  │       │              │
              │table     │       │              │
              └──────────┘       │              │
                                 ▼              │
                       ┌──────────────────┐     │
                       │ Async Moderation │     │
                       │ (ML toxicity,    │     │
                       │  Kafka consumer) │     │
                       │ - issues delete  │─────┤  publish "delete" event
                       │   events on bad  │     │
                       │   comments       │     │
                       └──────────────────┘     │
                                                │
                       ┌──────────────────┐     │
                       │ Pin Service      │─────┤  publish "pin" event
                       │ (reliable        │     │   (separate reliable
                       │  channel)        │     │    delivery, not sampled)
                       └──────────────────┘     │
                                                │
                       ┌──────────────────┐     │
                       │ Sampling Service │─────┘  (only for ≥1M-viewer
                       │ - decides which  │       streams; samples comments
                       │   comments to    │       to ~500/sec ceiling
                       │   forward)       │       per gateway)
                       └──────────────────┘

                  ┌──────────────────────────────────────────────────────────┐
                  │               Stream Registry (KV)                       │
                  │   stream_id → {topic_id, viewer_count, tier_assignment,  │
                  │                 chat_settings_version}                   │
                  └──────────────────────────────────────────────────────────┘
```

### Request walk-through

**Post path (viewer types a comment):**
1. Client emits `{type: "post", local_id, text}` over its open WS to its Connection Gateway.
2. Gateway runs cheap local checks: `chat-locked?`, `sub-only?`, `slow-mode for this user?`, `length`. Most rejections happen here at <1ms.
3. Gateway forwards to Comment Ingest Service over gRPC.
4. Ingest does:
   - Banned-word filter (bloom filter, in-memory).
   - Sync profanity/spam filter (~10ms).
   - Mints `comment_id` (Snowflake).
   - Writes to Cassandra (`comments` partition keyed by `stream_id`, async — don't block on durability for normal comments).
   - Publishes onto the stream's pub/sub topic.
5. Ingest replies `post_ack` to gateway → forwarded to client.
6. Async path: comment is also published onto a Kafka topic for the ML toxicity classifier; if classified bad, classifier emits a `delete` event back into the pub/sub, which fans out to gateways and retracts the comment from all viewers' UIs.

**Subscribe / receive path (viewer reads chat):**
1. Client opens WS, authenticates, sends `{stream_id}`.
2. Gateway: lookups stream tier in registry. If small/mid stream, gateway directly subscribes to the stream's Redis Streams / Kafka topic. If extreme-scale stream, gateway connects to its assigned tier-1 or tier-2 fanout node in the tree.
3. Gateway sends client the recent N comments (last 50, hot Redis cache) for context.
4. Gateway streams new comments as they arrive on the pub/sub.
5. Per-viewer per-connection bounded buffer: if buffer fills (slow client), drop oldest comments — never block the gateway's outbound loop.
6. For ≥1M-viewer streams, gateway applies sampling: only forwards a fraction of comments (e.g., 500 of 10K/sec) to its connected viewers. Each viewer sees a representative subset; sampling is randomized per-viewer.

### The hierarchical (tree) fanout — why and when

For a 10M-viewer stream, even Redis Streams or a single Kafka topic can't fan out 10K msgs/sec to 1000+ subscriber gateways efficiently — the publisher saturates on outbound network. Solution:

```
        Comment Ingest
              │
              ▼
        ┌─────────┐
        │  Root   │  (1 publisher per stream)
        └────┬────┘
             │
   ┌─────────┼─────────┐
   ▼         ▼         ▼
┌──────┐ ┌──────┐ ┌──────┐
│Tier-1│ │Tier-1│ │Tier-1│   (10–100 nodes; each subscribes to root)
└──┬───┘ └──┬───┘ └──┬───┘
   │        │        │
 ┌─┴──┐   ┌─┴──┐   ┌─┴──┐
 ▼    ▼   ▼    ▼   ▼    ▼
┌──┐┌──┐ ┌──┐┌──┐ ┌──┐┌──┐
│G││ G│ │G ││ G│ │G ││ G│   (tier-2 gateways; each holds 10K WS clients)
└──┘└──┘ └──┘└──┘ └──┘└──┘
```

Each level fans out to ~100 nodes, total amplification 10K. For 10M viewers: root → 100 tier-1 → 10K gateways → 10M viewers. The root only sees 100 outbound connections, not 10K. The pub/sub mesh is a tree rather than a star.

### Why this hybrid (per-stream tier) architecture

- **Pure star fanout (one Redis topic, all gateways subscribe)** breaks at top streams: 10K gateway subscribers on one Redis instance overwhelms the publisher.
- **Pure tree fanout for everything** is operational overhead for the 99% of streams with <1K viewers — overkill.
- **Per-stream tier assignment** (Stream Registry says "this stream is tier-3, use tree; that stream is tier-1, use direct Redis Streams") matches workload to mechanism.

State this hybrid explicitly as the central insight, just like push/pull was the insight in the news feed problem.

---

## 8. Deep dives

### Deep dive 1: Sampling at extreme scale

> **Interviewer:** "Walk me through what happens for a 10M-viewer stream with 10K comments/sec."

**Candidate:**

10K comments/sec × 10M viewers = 100B deliveries/sec. We cannot do that. So we **sample server-side** at the gateway tier.

**Algorithm:**
1. Gateway receives all comments from its tier-2 subscription (full firehose, 10K/sec).
2. For each viewer connected to the gateway, the gateway applies an independent random sample at a target rate (say 500 comments/sec/viewer is the "feels alive" threshold).
3. Sample selection is **per-viewer randomized** so different viewers see different subsets — chat appears different to everyone, which actually feels right (you can't read 10K/sec anyway).
4. Special messages (mod, sub donating, pinned, mentions of @viewer) bypass sampling and are always delivered.

**Sampling rate calculation:**
- Target per-viewer cadence: 100–500 comments/sec (max human readable).
- Stream rate: 10K comments/sec.
- Sampling fraction: 5%.
- This is configurable per-stream and dynamic — when comment rate drops, sampling fraction goes up automatically; users see most comments at low rates.

**The "engagement" trick** (this is a Twitch/YouTube real production behavior):
- When a viewer hovers/pauses on the chat, the gateway momentarily increases delivery rate to that viewer (or replays unsampled buffer).
- This makes individual messages "feel" delivered even if they were initially dropped by sampling, because the user can see them when they engage.

**Why this works psychologically:**
- Chat that scrolls 10K msg/sec is unreadable — sampling improves UX, not just infra.
- Each viewer seeing a different sampled subset means everyone has an "I saw it first" experience.
- Mods/streamer see the full firehose (separate, unsampled gateway connection — they need to see everything to moderate).

**Cost benefit:**
- Without sampling: 100B deliveries/sec → ~10 TB/s gateway egress. Infeasible.
- With 5% sampling: 5B deliveries/sec → ~500 GB/s gateway egress. Hard but feasible across 1000s of gateways.

> **Interviewer:** "Doesn't this mean some comments are 'lost'?"

**Candidate:** Yes — and that's accepted. Comments are stored durably in Cassandra (every comment is persisted regardless of sampling). Sampling only affects real-time fanout. VOD replay can show all comments at the right offsets. Mods see all comments. So nothing is lost; it's only sampled at the live-delivery layer.

### Deep dive 2: Hierarchical (tree) fanout pipeline for 10M-viewer streams

> **Interviewer:** "Walk me through what changes when one stream has 10M concurrent viewers."

**Candidate:**

Several things break vs. the small-stream case:

1. **Single Redis topic saturates.** Redis Streams can do ~100K msg/s in, but its outbound to N subscribers is N × 10K msg/s. 10K gateway subscribers × 10K comments/sec = 100M msgs/sec out. One Redis instance can't.

2. **Single Kafka partition** has the same problem — partition leader is on one broker.

3. **Solution: tree fanout** with intermediate fanout nodes.

```
Root publisher (1 instance, owns the stream's authoritative topic)
       │  ~10K msg/sec out, to 100 tier-1 nodes
       ▼
Tier-1 fanout nodes (100 instances; each subscribes to root)
       │  each sees 10K msg/sec in, fans to ~100 tier-2 nodes
       ▼
Tier-2 fanout nodes (10K instances; one per gateway, or co-located on the gateway)
       │  each sees 10K msg/sec in, fans to ~1000 viewer WS connections
       ▼
Viewer connections (10M total, with sampling at the tier-2 gateway)
```

Each node only sees 10K msg/sec in and ~10K msg/sec out (uniform load). Total amplification of 10M is achieved without any one node carrying all of it.

4. **Tier assignment** is dynamic, in the Stream Registry:
   - tier 0 = small (<10K viewers): one Redis Stream, all gateways subscribe directly.
   - tier 1 = mid (10K–100K viewers): single Kafka topic with many partitions.
   - tier 2 = large (100K–1M): two-level tree (root + tier-1 fanout nodes).
   - tier 3 = mega (1M–10M+): three-level tree with explicit fanout mesh.

5. **Promotion/demotion:** A viral stream might cross thresholds during a single broadcast. Promotion logic:
   - Stream Registry watches `viewer_count`.
   - On crossing threshold, registers the new tier; provisions tier-1 nodes from a warm pool.
   - Gateway re-subscribes from the new tier-1 instead of root.
   - Brief (1–5s) chat-lag during promotion; we accept this as graceful degradation.

> **Interviewer:** "What if a tier-1 fanout node goes down?"

**Candidate:**
- Detection: heartbeat lost (5s).
- Blast radius: 1/100 of tier-2 nodes lose their publisher. The viewers connected to those gateways get no chat for ~5–15s.
- Mitigation: each tier-2 has two upstream tier-1 publishers (active/standby). On detection, switch to standby. Comments may briefly duplicate; gateways dedupe by `comment_id`.
- Recovery: Kubernetes/scheduler replaces failed node from warm pool, re-attaches to root.
- We never serve from the wrong tier. Better to lag for 5s than to deliver inconsistent state.

### Deep dive 3: Connection Gateway sharding and lifecycle

> **Interviewer:** "How are viewers distributed across gateways? What happens when a gateway crashes?"

**Candidate:**

**Sharding by `stream_id`, not `user_id`:**
- All viewers of stream X land on a small number of gateway hosts that are subscribed to X's topic.
- Why: pub/sub coherence. If we sharded by `user_id`, we'd need every gateway subscribed to every popular stream — fanout amplification at the pub/sub layer.
- WS LB does consistent-hash on `stream_id` from the connect URL → routes to gateway pool for that stream.

**Capacity:**
- 10K WS connections per gateway host (well-tuned Linux + epoll/io_uring).
- A 1M-viewer stream needs ~100 gateways subscribed to its topic.
- A 10M-viewer stream needs ~1000 gateways → thus the tier-2 layer with hierarchical fanout.

**Crash scenario (a gateway dies with 10K viewers connected):**
1. Detection: WS LB notices via TCP failure / health check (~5s).
2. Blast radius: 10K viewers disconnect simultaneously.
3. **Reconnect storm.** All 10K clients immediately try to reconnect — naive design would cause 10K simultaneous TLS handshakes hitting the LB at the same instant, plus the `chat connect` flow which loads recent-comments cache, etc.
4. **Mandatory mitigation: client backoff with jitter.** Clients use exponential backoff with full jitter (e.g., wait 0–5s randomly before reconnect, then 0–10s, etc.).
5. **Server-side: per-LB rate limit on new WS handshakes per second.** Excess clients get 503 with `retry_after`.
6. Pre-warmed gateway capacity in the pool absorbs the displaced viewers (capacity headroom is 30–50%, mandatory).

**Even worse case: a whole AZ goes down:**
- 100 gateways, 1M viewers displaced.
- Multi-AZ design: WS LB has global state, routes new connections to remaining AZs.
- Reconnect with jitter is the only thing standing between "graceful degradation" and "thundering herd self-DoS."

> **Interviewer:** "What about per-viewer slow connections — what if a mobile viewer has a bad network?"

**Candidate:** Per-WS bounded send buffer, e.g., 1MB. If buffer fills (mobile is slow), gateway drops oldest queued comments rather than blocking. The viewer experiences a brief gap (chat looks "thinned out") but never a stalled connection. Critically, **slow viewers must not slow down delivery to other viewers on the same gateway** — bulkhead per-connection. Each WS's send is non-blocking; backpressure is on the per-connection buffer, not the gateway-wide select loop.

### Deep dive 4: Moderation pipeline (sync + async)

> **Interviewer:** "How does moderation work end-to-end?"

**Candidate:**

Two layers, both required:

**Sync (inline at ingest, before fanout):**
- Banned-word list (per-stream, edited by mods, cached at ingest as a bloom filter).
- Length / rate / character validation.
- Slow-mode check (Redis read).
- Sub-only / follower-only check (cached at gateway; verified at ingest).
- Per-user ban check.
- Aho-Corasick pattern matching for compound banned phrases.
- All of this in <50ms inline. Comment is rejected with reason if any fails — never reaches fanout.

**Async (post-fanout, with retraction):**
- Every accepted comment is also pushed to a `chat.toxicity` Kafka topic.
- ML toxicity classifier service consumes, scores comments. Latency: 1–5s.
- If score > threshold → emit `delete` event back to the stream's pub/sub.
- `delete` event fans out to all gateways like a normal message; clients receive `{type: "delete", id}` and remove from UI.

**Why this split:**
- Sync filtering catches the obvious (banned words). Cheap, fast, in-line.
- ML classifier is too slow for inline (would add 1–5s to post latency, breaks "live" feel).
- We accept that toxic comments are visible for 1–5s before retraction. Industry practice. (This is exactly how Twitch/YT Live operate.)

**Mod-issued deletes:**
- A mod clicks "delete" on a comment in their UI.
- REST call: `DELETE /v1/streams/{id}/chat/comments/{comment_id}`.
- Service marks comment as `deleted` in Cassandra, emits `delete` event onto pub/sub.
- Fans out to all viewers within seconds — same channel as ML-issued deletes.

**Mod actions are reliable, not sampled:**
- The delete event bypasses sampling logic at the gateway. Every viewer who saw the original (or who has it in their UI) must get the retraction.
- Implementation: `delete` events are flagged "must-deliver" and sent ahead of sampled comments in the gateway's outbound queue.

> **Interviewer:** "How fast does delete propagate?"

**Candidate:** Goal is <2s p99 from mod-click to all viewers' UIs updated. Component breakdown:
- REST → service: ~50ms.
- Cassandra write: ~20ms.
- Publish to pub/sub: ~10ms.
- Tree fanout: ~100–500ms for 10M-viewer stream (1–3 hops).
- Gateway → client WS: ~100ms.
- Client UI update: ~50ms.
- Total: ~300–800ms p50, ~2s p99.

### Deep dive 5: Slow mode, follower-only, sub-only enforcement

> **Interviewer:** "Walk me through slow mode."

**Candidate:**

Slow mode is a per-stream rate limit on per-viewer comment cadence. E.g., "1 comment per 5 seconds per viewer."

**Path:**
1. Mod sets slow mode via `POST /v1/streams/{id}/chat/mode {slow_mode_seconds: 5}`.
2. Setting persists in Cassandra `chat_settings`, also written to Redis with key `settings:{stream_id}`.
3. Stream Registry emits a `chat_settings_version` bump on the stream's control-channel topic.
4. Gateways receive the bump, refresh their local 5s-TTL cache for that stream's settings.
5. On viewer post:
   - Gateway reads `slow_mode_seconds` from local cache.
   - Reads `slowmode:{stream_id}:{user_id}` from Redis (last post time).
   - If now − last_post < slow_mode_seconds → reject locally with `post_reject {reason: "slow_mode"}`.
   - Otherwise: increment / set the Redis key with TTL = slow_mode_seconds × 2.

**Why Redis:**
- Per-(stream,user) counter. Cardinality is high (10M viewers × N streams) but TTL bounds it.
- Atomic SET-with-NX or Lua script avoids race.
- Alternative: in-memory per-gateway counter — won't work because a viewer might post via different gateways across reconnects.

**Follower-only / sub-only:**
- Gateway has a cached per-viewer flag: `is_follower(stream)`, `is_subscriber(stream)` — fetched on connect from Identity Service, cached for the WS session.
- On post: check cache; reject locally if violation.
- Subscription expiration during stream: separate event from billing service updates the cache.

**Mod and streamer are exempt** from all of these — mods can post fast in slow mode to handle moderation responses.

### Deep dive 6: Pinned messages — the reliable channel

> **Interviewer:** "Pins behave differently — explain."

**Candidate:**

Pins are guaranteed to all viewers, including new viewers who join mid-stream. Three properties differ from regular comments:

1. **Always delivered** — bypass sampling. Special channel.
2. **Available on connect** — when a new viewer connects, they receive the current pin (if any) immediately along with recent comments.
3. **Durable** — written to a separate `pinned_messages` table keyed by `stream_id`, plus Redis hot copy.

**Pin path:**
1. `POST /v1/streams/{id}/chat/pin {comment_id}` (mod call).
2. Pin Service writes to `pinned_messages` (Cassandra, QUORUM — durable) AND Redis (`pin:{stream_id}` → comment).
3. Pin Service publishes `{type: "pin", id, text, expires_at}` on the stream's pub/sub control channel.
4. Gateways receive, store locally per-stream (so future connecters get it immediately), and forward to all currently connected viewers (no sampling).

**Edge-cache for pins on extreme streams:**
- For a 10M-viewer stream, even reliably delivering a pin to all viewers is 10M deliveries.
- But it's ONE delivery per viewer (no per-second rate). Manageable.
- Plus we cache the pin at edge (CDN), so the WS connect handshake serves the pin from the edge — saves a Redis round-trip per connect.

**Unpin:** Same path with `unpin` event, removes from cache, removes from edge.

### Deep dive 7: VOD replay (out of scope but mention)

> **Interviewer:** "What about replay — does the chat replay when someone watches the stream later?"

**Candidate:**

Yes. Comments are stored in Cassandra with `stream_offset_ms` (millisecond offset from stream start). On VOD playback:

1. VOD player calls `GET /v1/streams/{id}/vod/comments?from_offset_ms=X&to_offset_ms=Y&limit=500`.
2. Service issues a Cassandra range query: `SELECT * FROM comments WHERE stream_id = X AND comment_id BETWEEN snowflake(X) AND snowflake(Y)` (since `comment_id` is Snowflake, it's roughly time-sorted, but `stream_offset_ms` is the truth).
3. VOD player buffers chunks of comments and renders at the right playback timestamp.

**Drift consideration:** Real-time stream had a 1–5s broadcast delay; we record `stream_offset_ms` based on when the comment was posted relative to broadcast wall-clock time, not relative to its receipt at gateway. Otherwise replay would be desynced.

**Storage cost:** ~17 TB/day raw, 90-day retention = ~1.5 PB. Tiered: first 7 days hot Cassandra, then cold object storage (Parquet on S3) with a metadata index. Read path is rare enough that cold is fine.

---

## 9. Tradeoffs & alternatives

### Decision: Hybrid pub/sub tier per stream

| Option | Pros | Cons | When to pick |
|---|---|---|---|
| **Single Redis Streams topic per stream, star fanout** | Simple, low latency | Breaks at >1M viewers (publisher saturates) | Up to ~1M viewers |
| **Kafka topic per stream** | Durable, replayable, partitionable | Higher latency, harder to operate at this granularity | Mid-tier streams |
| **✅ Hybrid tier-based (small=Redis, mid=Kafka, big=tree)** | Right tool for each scale | Operational complexity, runtime tier promotion | Anytime there's >3 orders of magnitude scale spread between streams |
| **Proprietary mesh (custom)** | Tunable for exact workload | Build cost, ops cost — only Twitch/YT scale justifies | Hyperscale only |

### Decision: Server-side sampling for extreme streams

| Option | Pros | Cons | When to pick |
|---|---|---|---|
| **Deliver every comment** | Strict semantics | Infeasible at 10M viewers × 10K comments/sec | Small streams only |
| **Client-side sampling** | Server simpler | Wastes bandwidth, every viewer pays for unread comments | Doesn't actually solve the problem |
| **✅ Server-side sampling with special-message bypass** | Bandwidth scales with stream size, not (size × rate) | Some comments not delivered live; users may not see specific friend's comment | Required at extreme scale |
| **Per-viewer queues** | Clean isolation | Memory cost: 10M viewers × buffer = TBs | Not viable at this scale |

### Decision: Cassandra for `comments`, Redis for hot state

| Option | Pros | Cons | When to pick |
|---|---|---|---|
| **PostgreSQL** | Familiar, ACID | Won't handle 1M writes/sec | Small platforms only |
| **MongoDB** | Flexible schema | Scale ceiling lower than Cassandra | Smaller scale |
| **DynamoDB** | Managed | Cost prohibitive at this write volume | AWS-native + small |
| **✅ Cassandra/Scylla** | Linear scale, time-clustered partitions ideal for stream timelines | Eventual consistency, ops overhead | Hyperscale write workloads |
| **Kafka as primary store** | Durable log | Not query-able for VOD replay arbitrary ranges | Backbone, not store |

### Decision: Async ML moderation with retraction

| Option | Pros | Cons | When to pick |
|---|---|---|---|
| **Sync ML moderation (block before fanout)** | Toxic comments never visible | Adds 1–5s to post latency, kills "live" feel | Don't pick |
| **✅ Async with retraction within seconds** | Live latency preserved | Toxic comment briefly visible | Industry standard |
| **Client-side moderation** | No server cost | Bypassed easily, doesn't propagate | Augment-only |

### Decision: Sticky session sharding by `stream_id` (not `user_id`)

The non-obvious choice. By sharding by stream_id:
- Pub/sub coherence: gateways subscribe once per stream, not once per viewer.
- All viewers of the same stream are co-located on a small set of gateways.
- Cost: a viewer reconnecting to a different stream lands on different gateways — fine, the WS reconnect is already a fresh setup.
- If we sharded by user_id, every gateway would need every popular stream subscription → fanout explosion at the pub/sub layer.

### Decision: Best-effort ordering, not strict

Strict per-comment ordering is impossible to guarantee under sampling and tree fanout. We commit to:
- Per-stream best-effort ordering: comments roughly in posted order (within a tree-fanout drift window of <1s).
- Comments have monotonic Snowflake IDs; clients can sort within a small reorder window if they care.
- Mods see strict ordering on their unsampled feed.
- Average viewer doesn't notice or care about strict ordering at 1000+ msg/sec scrolling chat.

---

## 10. Defending architectural choices — interviewer attacks

> **Interviewer:** "Why not just use a single global Kafka cluster with one topic per stream? Kafka is durable and scales."

**Candidate:** Three problems.
1. **Topic count.** 100K active streams = 100K topics. Kafka's metadata layer (ZK historically, KRaft now) doesn't love this; rebalancing and broker startup degrade.
2. **Latency.** Kafka producer-to-consumer p99 is 100–500ms even at low load. Our budget is 2s end-to-end with several other hops eating into it. Redis pub/sub is sub-10ms.
3. **Fanout amplification.** Kafka's consumer model is partitions × consumer groups. To fan out to 10K gateway subscribers, we'd need 10K consumer groups on one topic — that's not the model. We'd end up rebuilding tree fanout on top of Kafka anyway.

I'd use Kafka for the durable backbone (every comment goes through it for the moderation classifier and async processing), but not for real-time fanout to gateways. Redis Streams + tree mesh handles real-time better.

> **Interviewer:** "Why allow toxic comments to be visible at all? Just block before fanout."

**Candidate:** Latency budget. Sync ML inference is 1–5s minimum (the model is non-trivial; we can't shrink it without losing accuracy). Adding that to post latency makes posters feel laggy — they post, see no echo for 5s, post again, system gets duplicate-posts, UX breaks. Async-with-retract is the industry pattern (Twitch, YouTube, Discord all do it). The trade is: 1–5s of toxic comment visibility for some viewers vs. 5s lag for every poster. We pick the former.

If a particular stream is very high-stakes (kids' content, news), we can lower the threshold of synchronous filters or apply chat-locked-by-default and require mod approval — that's a per-stream policy.

> **Interviewer:** "Why server-side sampling? The client could just throttle rendering."

**Candidate:** Bandwidth and connection memory. Even if the client throttles rendering, we still send every comment over the WS — that's egress cost. For 10M viewers × 10K comments/sec × 200B = 20 TB/sec egress. We can't pay for that. Sampling at the gateway means we send only ~500 msgs/sec/viewer × 10M = 1 TB/sec — still huge but in the realm of feasible. Plus per-WS send-buffer memory at the gateway is bounded.

The client throttling is additionally useful (browser can't render 10K msg/sec anyway), but it's not sufficient.

> **Interviewer:** "Why not use a single global pub/sub mesh for all streams, instead of per-stream tier?"

**Candidate:** Two reasons.
1. **Isolation.** A bad fanout for one stream (say a viral 10M-viewer stream's tree gets congested) shouldn't degrade chat for an unrelated 1K-viewer stream.
2. **Cost.** Most streams are small and don't need the tree-mesh apparatus. Provisioning one of those per stream wastes resources. Per-stream tier keeps the tooling lightweight for small streams and reserves heavy infra for the few big ones.

> **Interviewer:** "What if Cassandra goes down?"

**Candidate:** The real-time fanout path uses Redis pub/sub, not Cassandra. Cassandra is only on the durable-write path. Worst case during Cassandra outage:
- Real-time chat continues working (viewers see comments, fanout via Redis works).
- Comments not durable to Cassandra → possible loss if Redis pub/sub drops them.
- Mitigation: Comment Ingest writes to a local WAL on the host first, queues for Cassandra retry. On Cassandra recovery, drains WAL.
- Pinned messages and mod actions are higher-stakes — for those, we wait for Cassandra ack before publishing the pub/sub event. They're rare enough that the latency cost is fine.
- VOD replay degraded for the outage window (no comments to replay).

> **Interviewer:** "Why not just deliver comments in batches every 200ms instead of per-comment?"

**Candidate:** Good optimization, and yes, we batch at the gateway → client level — every 100–200ms the gateway flushes accumulated comments to each WS. This:
- Reduces WS frame overhead (~30B/frame).
- Lets us drop or sample efficiently across the batch window.
- Improves render efficiency on the client.

The post → ingest → pub/sub path is per-comment (we don't want to add 200ms latency on the publish side). The gateway → client side is batched. That's a good answer to give before being asked — staff signal.

> **Interviewer:** "How do you handle a viewer who's a bot creating fake comments?"

**Candidate:** Several layers:
1. Per-user post rate limit at gateway (e.g., 5/sec hard limit regardless of slow mode).
2. Per-IP rate limit at edge LB.
3. CAPTCHA challenge on chat connect for unauthenticated/suspicious accounts.
4. ML-based bot detection (typing cadence, content similarity to other accounts) — runs async, can ban accounts retroactively.
5. Shadow ban mode: bot's comments accepted, written to Cassandra, but not published to fanout. Bot doesn't realize.

---

## 11. Failure modes & mitigations

| Failure | Detection | Blast radius | Mitigation | Recovery |
|---|---|---|---|---|
| Connection Gateway crash | Health check / TCP failure | 10K viewers disconnect; reconnect storm | Mandatory client-side jittered backoff (0–5s, exp); per-LB rate limit on new WS handshakes; warm pool absorbs | Auto-replace pod; viewers reconnect over 30s |
| Whole AZ outage | Multi-region health probes | 100K viewers, 100 gateways disconnect | Multi-AZ active-active gateways; LB shifts traffic to remaining AZs; clients reconnect with jitter | Restore AZ; rebalance load |
| Pub/sub partition leader loss (Redis primary fails) | Redis sentinel / health metric | Stream delivery paused for failover window (5–10s) | Redis Sentinel auto-failover; gateways reconnect to new primary with retry; comments queued in publisher buffer | Failover completes; lag drains within 30s |
| Tree fanout tier-1 node down | Heartbeat lost | 1/N of tier-2 nodes lose feed; viewers see lag for 5–15s | Tier-2 has hot-standby tier-1; auto-failover; dedup at tier-2 by `comment_id` | Replace failed node from warm pool |
| Comment Ingest overload (viral spike) | Latency / queue depth alarm | Posts rejected with 503 | Auto-scale; per-viewer rate limit; drop posts at edge with `slow_down` reason; mods/streamer prioritized | Scale completes in 1–2 min |
| Cassandra `comments` partition hot | Per-partition write latency | One stream's writes lag → fanout stalls if we wait | Don't wait — fanout publishes in parallel with persist; if persist fails, retry from WAL | Add capacity; if a single stream is too hot, shard by `(stream_id, time_bucket)` |
| Moderation classifier down | 5xx from classifier service | Toxic comments not retracted; visible until classifier recovers | Sync filters still active; queue comments for backfill on recovery; alert mods to manual-mod | Bring service back; replay queued comments through |
| Pin Service down | 5xx from pin endpoint | Mods can't pin/unpin | Existing pins continue to be served (cached); new pins fail with explicit error | Restore service |
| Redis hot-pin cache evicted | Cache hit miss spike | Pins not returned on connect (until DB read fills) | Cassandra fallback; re-warm | Cache repopulates within seconds |
| ML classifier produces false positives | Mod feedback / quality alarm | Real comments deleted | Mod can un-delete (admin action that publishes "undelete" event); tune threshold | Rebuild model; manual review queue |
| Gateway → client WS slow (mobile network) | Per-WS send-buffer high water | One viewer lags; should not slow others | Bounded per-WS buffer; drop oldest comments on overflow; bulkhead per-connection | Viewer sees gap, reconnects |
| Stream tier promotion under load | Watch viewer_count crossing threshold | Brief 1–5s chat-lag during topology change | Pre-warm tier-1 pool; promotion is a config flip, not a deploy | Promotion completes; lag drains |
| Delete event doesn't reach all viewers | Delete-success metric per stream | Toxic comment lingers on some viewers' UIs | Delete events have higher priority and bypass sampling; gateway resends on suspected drop; Cassandra `deleted` flag means VOD replay never shows it | Manual mod cleanup if needed |
| VOD replay timestamp drift | Diff between live and replay | Comments appear at wrong moment in VOD playback | Use `stream_offset_ms` based on broadcast wall-clock, not gateway-receipt time; reconcile against video PTS in player | Tune offset calculation |
| DDoS on chat connect endpoint | WS handshake rate spike | Legitimate viewers can't connect | Edge LB rate limit per IP; CAPTCHA for new connections; shed at LB before reaching gateway | DDoS mitigation kicks in; capacity headroom absorbs |

---

## 12. Productionization

### Pre-launch / known-event prep
- [ ] **Pre-warm cells for scheduled major events.** Super Bowl streams: 24h ahead, provision 10× normal gateway capacity in the relevant region; pre-allocate tier-1 fanout nodes; warm Redis clusters.
- [ ] Load test at 10× expected stream peak. For a known 10M-viewer event, simulate 100M concurrent in pre-prod.
- [ ] Game day: simulate gateway-AZ outage during mock event; verify reconnect jitter prevents thundering herd.
- [ ] Chaos test: kill 10% of tier-1 fanout nodes during load; verify tier-2 failover works.

### Rollout
- [ ] Feature flag per stream-cohort. Start with non-event small streams.
- [ ] Per-region rollout: smallest region first.
- [ ] Auto-rollback if: end-to-end latency p99 > 5s for 5 min, drop rate (sampling exceedance) > 90%, post error rate > 1%.

### Capacity
- [ ] 50% headroom on every gateway pool.
- [ ] Auto-scale gateway pool on connection count, target 70% capacity per host.
- [ ] Auto-scale Comment Ingest on QPS, target 60% CPU.
- [ ] Cassandra: 3× current write capacity for spikes.
- [ ] Redis: 2× current memory footprint.
- [ ] Tier-1 / tier-2 mesh nodes: warm pool of 50% capacity, instantly promotable.

### Per-stream kill switch
- [ ] **Lock chat per-stream:** mod or admin can flip `locked: true` on `chat_settings`. All gateways refresh; Comment Ingest rejects posts. Reads continue. This is the nuclear option for spam waves or moderation incidents.
- [ ] **Lock chat platform-wide:** for catastrophic incidents (bot army, exploitation). Different config; requires admin.

### Migration (if replacing existing system)
- [ ] Dual-publish to old + new fanout for a month.
- [ ] Per-stream cutover (start with smallest, work up to event streams).
- [ ] Old system stays on standby for instant rollback.

### Cost analysis
- Largest cost driver: **Connection Gateway memory** (RAM for 10M concurrent WS connections). At 20 KB/conn × 10M = ~200 GB platform-wide, but with overhead, per-conn buffers, OS state — closer to 1–2 TB. ~1000 hosts.
- Second largest: **Pub/sub bandwidth.** Tree fanout amplifies, but each link is bounded. ~100 GB/s aggregate at peak.
- Third: **Cassandra storage.** ~17 TB/day. Tier hot/cold.
- Levers:
  - Sampling rate (lower → less egress, less CPU).
  - Connection multiplexing (HTTP/3 multiple streams per conn? marginal).
  - Move VOD comments to cold tier after 7 days.

### Compliance
- **GDPR right-to-erasure:** user delete must remove all their comments from `comments` (+ tombstones in pub/sub history). Async sweep, batched.
- **Data residency:** EU comments stored in EU region; cross-region replication only if user opts in or for global broadcasts (see cellular section).
- **Audit:** mod actions logged immutably; CSAM detection mandatory.

---

## 13. Monitoring & metrics

### Business KPIs
- Comments per stream-minute (engagement)
- Active chat-participating viewers / total viewers (engagement rate)
- Stream-watch duration with chat enabled vs disabled
- Mod action rate (toxicity proxy)

### Service-level (SLIs)
- **End-to-end comment latency** (post → visible to other viewers): p50, p95, p99 — the hero metric. SLO: 99% < 2s end-to-end.
- **Post acceptance latency** (post → ack to poster): p99 < 200ms.
- **Connection establishment latency** (WS connect → ready to receive): p99 < 1s.
- **Drop rate** (sampling rate per stream): expose, alert if it spikes unexpectedly (which means we're failing harder than designed).
- **Moderation latency** (toxic comment detected → all viewers see retraction): p99 < 5s.
- **Pin propagation** (pin set → all viewers see): p99 < 2s.

### Component metrics

#### Connection Gateway
- WS connection count per host (target ≤10K)
- New connection rate per second (alert on spikes — reconnect storm signal)
- Per-WS send-buffer high-water-mark distribution
- Disconnects per second (with reason: client, network, gateway-evicted, OOM)
- Outbound batch flush latency
- Memory utilization per host

#### Comment Ingest
- Post QPS per stream
- Sync moderation rejection rate (and breakdown: banned-word, length, slow-mode, sub-only)
- Cassandra write latency
- Pub/sub publish latency
- End-to-end ingest latency

#### Pub/sub fanout (Redis Streams / tree mesh)
- Per-stream publish rate
- Per-stream subscriber count (= gateway count for that stream)
- Tier-1 → tier-2 propagation latency
- Tree node CPU / network saturation
- Per-stream tier (track promotions/demotions over time)

#### Async moderation
- Toxicity classifier QPS
- Classifier inference latency p99
- False-positive rate (mod undelete / 1000 deletes — quality signal)
- Backlog depth on classifier Kafka topic

#### Slow mode / settings
- Slow-mode rejection rate per stream
- Settings cache hit rate at gateway
- Settings refresh latency on bump

#### Pin Service
- Pin/unpin rate
- Pin propagation latency p99

#### Cassandra
- Per-partition write latency (per-stream)
- Hot-stream detection (write-skew alert)
- Compaction queue depth

### Alert routing
- **Page** (24/7): e2e latency p99 > 5s for 5 min; connection gateway pool saturation; ingest 5xx rate > 1%; mod action propagation lag > 30s
- **Ticket** (next business day): drop-rate trending up; cassandra hot-partition; classifier false-positive rate up
- **Dashboard-only**: cost, throughput trends, tier promotion frequency

### Dashboards
1. **Live chat health** — RED metrics, e2e latency, drop rate, connection count, by region, by stream tier
2. **Per-stream deep-dive** — for a specific big stream: post rate, fanout amplification, sampling rate, mod activity
3. **Moderation** — sync vs async rejection rates, classifier latency, false-positive rate
4. **Infrastructure** — gateway pool utilization, pub/sub mesh health, Cassandra
5. **Business** — engagement, comments/min during big events

---

## 14. Security & privacy

- **AuthN at WS connect** via JWT bearing user_id + session; rotated; revocable.
- **AuthZ for posting** — checked at gateway from cached subscription/follower state; revalidated periodically (subscription expiration).
- **Posts validated server-side** for length, character set, embedded URLs (anti-phishing).
- **Per-IP and per-user rate limiting** at the edge LB.
- **CAPTCHA** for new accounts attempting to post.
- **Shadow ban** for detected bots — comments accepted but never published to fanout.
- **PII**: user display name is public; email/IP is not — encrypt at rest, audit access.
- **CSAM detection** mandatory: every comment + media reference scanned by content-classifier in async path.
- **Mod action audit log** (immutable, 7-year retention for legal).
- **DDoS protection** at edge: WAF, anycast, per-IP rate limit.

---

## 15. Cost analysis

Rough order-of-magnitude:

| Component | Rough cost driver | Optimization lever |
|---|---|---|
| **Connection Gateway pool** | RAM × concurrent connections | Connection multiplexing; tighter buffer tuning; offload TLS to LB |
| **Pub/sub mesh / Redis** | RAM + bandwidth | Per-stream tier (don't run mesh for small streams); short retention |
| **Cassandra (comments)** | Storage + writes | TTL old comments; tiered storage to S3 Parquet |
| **Egress bandwidth** | Bytes to viewers | Sampling; batch frame compression |
| **Compute (Comment Ingest, classifier)** | CPU/GPU | Auto-scale; classifier model distillation; batch inference |
| **Moderation ML inference** | GPU | Quantization; smaller models on hot path, big model only when smaller is uncertain |

Largest cost: **gateway memory + egress bandwidth** in tandem. Engineering effort goes furthest on connection density and sampling tuning.

Rough $/concurrent-viewer at peak: ~$0.10–0.20/concurrent-viewer-month depending on stream profile.

---

## 16. Open questions / what I'd validate with PM

These are real Staff-level instincts — surfacing what you don't know.

1. Per-event SLOs: is a Super Bowl stream allowed to degrade differently than a normal stream? (Different sampling rates? Different latency targets?)
2. How important is comment delivery to moderators specifically? (Probably P0 — they need full firehose for moderation; design assumes a separate "mod" connection that bypasses sampling.)
3. Does the platform support cross-stream chat (a global lobby channel, e.g., "all viewers of company X's events")? Affects sharding strategy.
4. Are reactions (heart, fire) treated as comments or as separate aggregate counts? (Big difference: an aggregate "1000 hearts in last 5s" is much cheaper than 1000 individual comments.)
5. Is there a tipped/bits/donation tier where the message MUST be delivered to everyone? (Yes in production — that's a separate "guaranteed" channel similar to pins.)
6. VOD replay: do viewers expect chat at full rate, or sampled like live? (Probably sampled like live for consistency.)

---

## 17. Staff-level scorecard for this problem

| Signal | Did the candidate... |
|---|---|
| ✅ Identified server-side sampling as the central insight | Within first 10 min, framed by the 10B-deliveries/sec calculation |
| ✅ Quantified the fanout amplification | "10K comments/sec × 1M viewers = 10B deliveries/sec" made explicit |
| ✅ Identified hierarchical tree fanout for top streams | Drew the tree, named the depth and amplification |
| ✅ Sharded gateways by stream_id, not user_id, with reasoning | Justified by pub/sub coherence |
| ✅ Designed sync + async moderation split | Both sync filters AND async ML classifier with retraction |
| ✅ Volunteered reconnect-storm mitigations | Jittered backoff is mandatory, not optional |
| ✅ Distinguished pinned messages as a reliable channel | Bypass sampling; durable; available on connect |
| ✅ Made tier assignment dynamic (small/mid/large/mega) | Promotion logic for viral streams |
| ✅ Talked about cost in concrete terms | Gateway memory + egress identified as dominant |
| ✅ Discussed productionization unprompted | Pre-warming for known events, kill switch, jittered reconnect |
| ✅ Designed for graceful degradation | Cassandra outage → fanout still works; classifier outage → sync filters still active |
| ✅ Asked the interviewer at decision points | "Should I deep-dive sampling or moderation?" |
| ✅ Closed with a wrap-up speech | Restated dominant constraints, choices, what to do next |

### What separates Staff from Senior on *this* problem

A Senior candidate will:
- Correctly draw a WS gateway + pub/sub + comment store
- Pick reasonable databases
- Mention sharding and moderation

A Staff candidate will additionally:
- Make the **fanout amplification calculation** load-bearing in scoping ("we cannot deliver everything to everyone — sampling is the design")
- Identify **tree fanout** for extreme streams and explain the amplification factors per tier
- Explain **dynamic stream-tier promotion** as a runtime concern, not a static config
- Volunteer the **reconnect storm** mitigation (jittered backoff with rate limit) before it's asked
- Distinguish **pinned messages and mod actions as a separate reliability tier** that bypasses sampling
- Explain **bulkhead per-WS-connection backpressure** (slow viewers don't slow other viewers)
- Identify the **cost dominator** (gateway memory + egress) and propose levers (sampling rate, batch flushes, connection density)
- Discuss **per-stream cells for major events** as a productionization step (Super Bowl gets dedicated infra)
- Propose an explicit **kill switch** at per-stream and platform-wide levels

---

## 18. Wrap-up speech (template)

> "To wrap up: the dominant constraint here is fanout amplification — at 10K comments/sec × 10M viewers per stream, that's 100B deliveries/sec on a single stream, which is physically infeasible. The architecture answers that with three layers: (1) server-side sampling at the gateway tier so each viewer sees a representative ~500 msgs/sec subset rather than the full firehose, (2) hierarchical tree fanout for top streams so no single pub/sub broker becomes a bottleneck, and (3) per-stream tier promotion so we apply heavy infra only where needed. The data plane is Cassandra (partitioned by stream_id) for durable comments, Redis pub/sub for real-time delivery on small/mid streams, and a tree mesh for the top 1%. Sticky-by-stream-id connection sharding gives us pub/sub coherence. The two pieces I'd worry about in production are (1) reconnect storms when a gateway or AZ fails — mitigated by mandatory client-side jittered backoff plus per-LB handshake rate limit and warm pool capacity, and (2) toxic comment visibility window — mitigated by aggressive sync banned-word filters plus async ML classifier with retraction within seconds, accepting that <5s of brief visibility is the industry norm. To productionize: per-stream kill switch for nuclear stops, pre-warming dedicated capacity for scheduled major events like the Super Bowl, SLO-alerted on end-to-end latency p99 and drop rate. If we had another 30 minutes, I'd deep-dive (a) the cellular / cross-region story for global simulcasted events like Apple keynotes, or (b) the VOD replay storage tiering, or (c) the mod-tooling firehose — mods need a special unsampled connection."

---

## 19. Market-based isolation (cellular architecture) — bonus / "if time permits"

> Bring this up in the last 5 minutes if there's time. For live comments specifically, cellular has TWO flavors that differ from a generic news feed: **per-region cells** (data residency, blast radius) AND **per-event/per-stream cells** (Super Bowl gets its own dedicated stack).

### What & why

Two complementary cell taxonomies:

**Geographic cells (per-region):**
- US, EU, IN, BR, JP, CN — own Cassandra cluster, own Redis, own gateway pool, own Comment Ingest, own moderation classifier (locally trained on local language and norms).
- Drivers: data residency (GDPR, India DPDP, China PIPL), blast radius isolation, latency.

**Per-event / per-stream cells:**
- A Super Bowl stream gets its own dedicated cell — its own gateway pool, its own pub/sub mesh, its own Comment Ingest fleet.
- Why: a 10M-viewer stream's failure modes (reconnect storms, mesh saturation) shouldn't affect the rest of the platform. Bulkhead.
- This is unique to live-streaming chat — generic news feed doesn't need this.

### Architecture

```
        ┌────────────────────────── Global Edge ─────────────────────────────┐
        │  Anycast + GeoDNS routes by user country + stream-cell affinity    │
        └────────┬─────────────────┬──────────────────┬──────────────────────┘
                 │                 │                  │
       ┌─────────▼────────┐ ┌─────▼────────┐ ┌──────▼─────────────────────┐
       │   US Cell        │ │   EU Cell    │ │  Event Cell                │
       │ ┌──────────────┐ │ │ ┌──────────┐ │ │  ┌───────────────────────┐ │
       │ │WS Gateway    │ │ │ │WS Gateway│ │ │  │ WS Gateway (xxxL)     │ │
       │ │Ingest        │ │ │ │Ingest    │ │ │  │ Ingest                │ │
       │ │Cassandra     │ │ │ │Cassandra │ │ │  │ Cassandra (event-     │ │
       │ │Redis         │ │ │ │Redis     │ │ │  │  partition)           │ │
       │ │Mod-Classifier│ │ │ │Classifier│ │ │  │ Tree Mesh (3-tier)    │ │
       │ └──────────────┘ │ │ └──────────┘ │ │  │ Pin Service           │ │
       └─────────┬────────┘ └─────┬────────┘ │  └───────────────────────┘ │
                 │                │          └─────────┬──────────────────┘
                 │                │                    │
                 └────────────────┴────────────────────┘
                                  │
                  ┌───────────────▼────────────────────┐
                  │  Cross-Cell Bridge                 │
                  │  - Stream Registry (global view)   │
                  │  - Cross-cell viewer routing       │
                  │  - Federated pub/sub for global    │
                  │    simulcast streams (e.g., Apple  │
                  │    keynote, World Cup)             │
                  └────────────────────────────────────┘
```

### Routing: how a viewer lands in the right cell

- Each user has a **home region** (signup country).
- Each stream has a **home cell** assigned at stream-create:
  - For a regional stream (e.g., "US-only Twitch streamer"), the stream cell == streamer's region.
  - For a global event stream, the stream cell is either a dedicated event cell OR a primary region with cross-cell federation.
- On chat connect, edge LB routes the user to the cell hosting that stream — viewers cross cells if the stream isn't in their home region.

### The hard part: globally simulcast streams (Apple keynote, World Cup)

This is where live chat differs sharply from news feed. A globally simulcasted event has 100M concurrent viewers spread across regions. Two options:

| Approach | How | Pros | Cons |
|---|---|---|---|
| **Single global event cell** | One mega-cell handles all viewers globally; viewers cross regions on connect | Single source of truth for chat ordering; simpler topology | Cross-Atlantic RTT for EU viewers (150ms each way); single failure domain; data residency violations possible |
| **Per-region event cells with cross-cell federation** | Each region has its own event-cell; comments published locally + replicated to other regions via Bridge | Local latency, blast radius bounded per region, residency-friendly | Cross-cell sync introduces ordering / consistency challenges; comments from EU may appear ~500ms later in US |
| **Per-region with NO cross-cell sync** | Each region's chat is independent; viewer in EU only sees EU comments | Simplest; lowest latency | Breaks "global event" feel — EU viewer doesn't see US viewer's comment, kills shared experience |

**My pick for globally simulcasted live chat:** **per-region cells with cross-cell federation** for the comment stream, **but** with a sampling-aware sync — we don't replicate every comment cross-cell, only a sampled subset. This actually composes naturally with our existing sampling: each region is already sampling for its viewers; the cross-cell sync sends the per-region sampled feed to other regions, where it's merged with their own local sample.

So a US viewer sees: US comments at 80% sample weight + EU/IN/BR comments at 20% combined sample weight. This gives the "shared global event" feel while bounding cross-cell bandwidth.

### Tradeoffs of cellular for live chat specifically

| Dimension | Single global | Cellular | Net |
|---|---|---|---|
| Total infra cost | Lower (no duplication) | Higher (replicated stacks) | +30–50% cost |
| Engineering complexity | Lower | Higher | Significant tax |
| Compliance (residency) | Hard | Easier | Cellular wins, often non-negotiable |
| Blast radius | Whole platform | One region or one event | Cellular wins by 10× |
| Latency for in-region viewers | Higher | Lower | Cellular wins for 80% of traffic |
| Big-event scale | Concentrated risk | Dedicated event cell isolates | Cellular wins for events |
| Cross-cell consistency | N/A | New problem | Cellular pays a tax |

### What changes in the design

- **Stream Registry becomes global.** Replicated KV with sub-millisecond reads at the edge — same model as user directory in news-feed.
- **Cross-Cell Bridge** carries pub/sub events for federated streams.
- **Per-cell moderation classifiers** — trained on local language; banned-word lists per locale (slurs differ by language and culture).
- **Per-cell pin service** — pins still local but federated for global events.
- **Per-cell rollouts** — feature flags have a `cell` dimension.
- **Pre-warming for known events** — Super Bowl: warm the dedicated event cell 24h ahead, pre-allocate gateway pool, pre-create pub/sub mesh, run synthetic load test.

### When NOT to go cellular for live chat

- Pre-scale: under ~10M viewers globally, the operational tax exceeds the benefit.
- Single-region product (only US streamers, only US viewers).
- No regulatory pressure.

### What "Staff signal" sounds like here

> "I'd default to per-region cells from day one if data residency is a hard requirement, otherwise add it once we cross 10M concurrent viewers. The cell is the unit of blast radius and deploy. The unique twist for live chat vs other systems is the **per-event cell** — known mega-events like the Super Bowl get dedicated infra, pre-warmed 24h ahead, with a 3-level tree fanout mesh. Cross-cell global simulcast streams use sampled federation rather than full replication, which composes with our existing sampling layer rather than fighting it. The tax is real — 30–50% more infra and a permanent engineering surcharge — but for a live-streaming platform, the alternative is one bad event taking down chat for everyone, which is never acceptable when the broadcast is happening live and viewers can't recover the moment."
