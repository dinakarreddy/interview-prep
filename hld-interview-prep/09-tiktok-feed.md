# 09 — Design TikTok's For-You Feed (algorithmic recommendation feed without a follow graph at scale)

> The ML-heavy cousin of the news feed problem. The interviewer cares less about the answer and more about whether you correctly identify that **candidate generation** and **online ranking** are the only things that matter, and you spend your time there. The follow graph is irrelevant; the embedding space is everything.

---

## 0. Problem statement (as the interviewer would drop it)

> **Interviewer:** "Design TikTok's For-You feed. A user opens the app and immediately sees a stream of short-form videos picked algorithmically — not based on who they follow. They swipe to the next, swipe to the next, and the system gets better the more they engage. Walk me through how you'd build it."

That's all you get. Your job: scope it, not solve it whole.

---

## 1. Clarifying questions

Pick 5–6. Don't fire them as a checklist — interleave with the interviewer's responses.

> **Candidate:** "Before I design, let me scope this. A few questions:"
>
> 1. **"Is the feed personalized per-user from session zero, or is there a follow graph anywhere in the picture?"** — *Expect: no follow graph in scope; this is pure algorithmic recommendation. Critical — drives entire architecture.*
> 2. **"What's the rough scale — DAU, sessions, video plays per day?"** — *Expect: 600M DAU, ~30 video plays per session, 18B video plays/day, 50–100M videos in the active corpus.*
> 3. **"Are we recommending from a fixed corpus, or do new uploads need to enter the feed within minutes?"** — *Expect: ~100K new uploads/day, must be discoverable within minutes (cold-start for new content is a P0 problem).*
> 4. **"Should I cover upload + transcoding, or is that a delegated subsystem?"** — *Expect: delegate. Brief mention only. The interesting problem is retrieval + ranking.*
> 5. **"What's the latency target for opening the app and seeing the first video?"** — *Expect: p99 < 300ms for first video metadata; first frame paints from prefetch on client.*
> 6. **"How fresh do engagement signals need to be? Seconds, minutes, hours?"** — *Expect: seconds. A user who skipped a video at second 0:02 should not see similar content next swipe. This drives the entire feature pipeline.*
>
> **Out of scope (state explicitly):**
> - Video upload + transcoding pipeline (delegate to a YouTube-class problem)
> - Live streaming
> - Comments, DMs
> - Ads (mention as a P1 integration point — slot reservation in the ranked feed; don't design the bidding)
> - Search, hashtag pages
> - Following / friends graph (this is the For-You feed, not the Following feed)
>
> **Candidate:** "I'm going to design (a) the feed read path — how we generate candidates and rank them when a user opens the app, (b) the engagement signal pipeline that closes the loop in seconds, and (c) cold-start for both new users and new videos. Sound right?"

> **Interviewer:** "Yes."

---

## 2. Functional requirements

| Priority | Requirement |
|---|---|
| P0 | User opens app, instantly sees a personalized video feed (no follow graph required) |
| P0 | User swipes; next video is ready instantly (client prefetch) |
| P0 | System learns from engagement (play duration, like, share, follow, "not interested") within seconds |
| P0 | New uploads enter the candidate corpus within minutes |
| P0 | Cold-start: new users get a reasonable feed from session 1 |
| P0 | Cold-start: new creators get exposure escalated based on early engagement signal |
| P1 | Diversity: the feed doesn't show 10 videos in a row from the same creator/topic |
| P1 | Geographic / market filtering (per-country content policy) |
| P1 | "Not interested" + creator block must take effect immediately |
| P2 | Ads slot insertion (every Nth video) |
| Out | Upload pipeline, transcoding, live, comments, DMs, search, follow-graph feed |

---

## 3. Non-functional requirements

| Dimension | Target | Justification |
|---|---|---|
| Availability | 99.95% (~22 min/month) for feed read | Core product surface — outage = no app |
| Feed open latency | p50 < 150ms, p99 < 300ms (metadata for first video) | App teardown happens at ~250ms perceived delay; first frame paints from cache |
| Swipe-to-next latency | <50ms (served from prefetch, not network) | Client prefetches next 5 videos in background |
| Engagement signal freshness | <10s from event to feature-store visibility | The "skipped at 2 sec" signal must influence the next swipe |
| Ranking inference latency | p99 < 80ms for top-200 candidates | Online inference budget; dominates feed-open p99 |
| Candidate gen latency | p99 < 30ms for top-1000 from millions | ANN on vector DB |
| Consistency | Eventual everywhere; **monotonic reads** for the user's session (don't show the same video twice in one session) | Feed quality, not correctness |
| Durability | Cannot lose engagement events (drives ranking) | Events are revenue-bearing |
| Scale | 600M DAU, 1B MAU, ~18B video plays/day, ~200K engagement events/sec | TikTok-class |
| Geography | Global, per-country isolation (cellular) | Content policy + ban-state countries (India) + Douyin/TikTok split |

---

## 4. Capacity estimation

> **Candidate:** "Let me estimate aggressively — round numbers. Stop me if you want me to dig deeper. The numbers here are a bit different from the news feed problem because there's no fanout — the cost is in retrieval and ranking, not delivery."

### Users & sessions
- 600M DAU, 1B MAU
- Avg 90 minutes/day in app
- ~30 video plays per session, ~3 sessions/day → ~90 plays/user/day
- 600M × 30 = **18B video plays/day** — this is the system's primary throughput number

### Reads (feed loads)
- A "feed load" here is conceptually different from news feed — the *client* prefetches a batch of 5–10 videos per call, then serves swipes from cache.
- 18B plays / 5 videos per fetch = **3.6B feed-API calls/day**
- Per second avg: 3.6B / 86,400 ≈ **42K feed-API calls/sec**
- Peak (3×): **~125K feed-API calls/sec**

### Engagement events (this is the punchline)
- Every play is an event. Plus likes, shares, follows, "not interested", scrubs, completion-rate signals.
- 18B plays × ~3 events/play (start, progress, end + occasional like) ≈ **50B events/day**
- More conservatively, just the play-event stream: 18B/day → 200K events/sec sustained, 600K events/sec peak
- **This is what Kafka must absorb. This is what Flink must process. This is the dominant throughput number in the system.**

### Corpus
- 50–100M active videos (videos eligible for recommendation in the past N days)
- 100K new uploads/day enter the corpus
- A video typically lives in the active corpus for weeks (decayed by recency); cold storage afterward
- Embeddings: 256-dim float32 = 1KB per video → 100M × 1KB = **100 GB of video embeddings**, fits in RAM on a few hosts (the entire ANN index is RAM-resident)
- User embeddings: 600M users × 1KB = **600 GB**, larger but distributable across a fleet

### Storage
- Video metadata (title, creator, duration, hashtags, audio, thumbnails, encoding URLs): ~5KB per video
- 100M videos × 5KB = 500 GB metadata; plus historical/cold = a few TB
- Watch history (per-user, last 1000 videos watched with engagement): 600M × 1000 × 50 bytes = **30 TB**
- Engagement event log (raw, in Kafka tier, 7-day retention): 50B/day × 200 bytes × 7 = ~70 TB; in Iceberg tier longer
- **Decision driver:** watch history goes to Cassandra (wide-row per user); engagement events to Kafka → Flink → Iceberg + Feature Store; video metadata to Cassandra; embeddings to vector DB (ScaNN/Vespa/in-house FAISS)

### Bandwidth
- Feed-API call returns 5 videos × ~5KB metadata + ~10KB ranking signals + URL set ≈ 50KB
- 125K calls/sec × 50KB = **6.25 GB/s** at peak from API (just metadata; actual video bytes go via CDN)
- CDN egress: 18B plays × ~5MB avg (compressed short video) = **90 PB/day** of video egress — this is the dominant infrastructure cost line, but it's CDN-handled and out of scope for the feed-design conversation. Worth mentioning once and moving on.

### The numbers that drive the design
1. **18B plays/day** → ranking must be cheap per-play; can't run a 100ms heavyweight model on every play, must use two-stage retrieval
2. **200K events/sec sustained** → Kafka + Flink, not REST; can't synchronously write features
3. **100M video corpus, must filter to ~1000 candidates per user in <30ms** → vector DB ANN is the only viable option (can't scan, can't pre-fanout)
4. **Engagement-to-feature-store in <10s** → online feature store with streaming pipeline, not batch

---

## 5. API design

> **Candidate:** "Public surface is small. Internal surfaces are where the interesting work is — but I'll show the public APIs first."

### Read (feed)
```
GET /v1/feed?session_id={id}&context={device,network,location}&n=10
→ 200 {
    items: [{
      video_id, creator_id, video_url, thumbnail_url,
      duration_ms, audio_id, hashtags[],
      ranking_debug: { score, model_version, candidate_source }
    }],
    session_cursor  // opaque; tracks already-served videos in this session
  }
```

- **No pagination cursor** in the traditional sense. The feed is infinite and personalized; we track session state (already-served set) on the server.
- Client calls this when it has fewer than 5 prefetched videos buffered.
- `session_cursor` opaque; encodes already-shown video IDs (or a hash of them) so the server doesn't re-recommend.

### Engagement events (write — this is the high-volume API)
```
POST /v1/events  (batched, 5–20 events per call)
  body: [{
    event_type: "play_start" | "play_progress" | "play_end" | "like" | "share" | "follow" | "not_interested" | "creator_block",
    video_id, user_id, session_id,
    timestamp, watch_duration_ms, watch_completion_pct,
    context: { device, network, location }
  }]
→ 202 Accepted
```

- 202, not 200 — fire-and-forget. Client batches and retries; we cannot block the user on event ingestion.
- Goes directly to Kafka (via a thin event-ingest service that does authn + schema validation).

### Negative signals (must take effect immediately)
```
POST /v1/feedback/not_interested { video_id, reason? }
POST /v1/feedback/block_creator { creator_id }
DELETE /v1/feedback/block_creator/{creator_id}
```
- These need stronger semantics than fire-and-forget — the next feed-API call must respect them. Block list is read at retrieval-time filter.

### Anti-patterns to avoid
- **Don't return the entire ranking signal payload to the client** — it's a leak (and a few KB per video). Return only what the UI needs; ship debug signals only on internal-debug builds.
- **Don't make the feed endpoint do upload or transcoding-related work** — the video URL it returns must already point to CDN-hosted, transcoded content.
- **Don't make the client call multiple endpoints to render a video** — one call, all metadata in the response, even if it duplicates a few fields per video.

---

## 6. Data model

### Tables / collections

#### `videos` (sharded by video_id, Cassandra)
| Column | Type | |
|---|---|---|
| video_id | bigint (Snowflake) | PK |
| creator_id | bigint | secondary index |
| duration_ms | int | |
| audio_id | bigint | secondary; many videos share audio |
| hashtags | list<string> | |
| caption | string | |
| upload_ts | timestamp | |
| visibility | enum (public, private, unlisted) | |
| eligible_markets | set<string> | per-country filter; supports geo-restriction |
| moderation_status | enum (pending, approved, removed) | |
| stats_cache | json | denormalized counts (refreshed periodically) |

#### `creators` (sharded by creator_id, Cassandra)
| Column | Type | |
|---|---|---|
| creator_id | bigint | PK |
| follower_count | bigint | |
| upload_count | int | |
| avg_engagement_rate | float | |
| reputation_score | float | trust signal for ranking |

#### `user_watch_history` (sharded by user_id, Cassandra wide-row)
- Clustering key: timestamp (descending)
- Capped at last ~1000 entries per user (TTL or LRU at compaction)
- Each row: `{video_id, creator_id, watch_duration_ms, completion_pct, liked, shared}`
- This is the source of truth for "have I shown this user this video already?" + behavioral feature aggregation

#### `user_features` (online, served from feature store; Redis-backed)
- Real-time slice: last-N video embeddings averaged (recency-weighted), recent dwell times, recent likes
- Daily slice: demographics, long-term interest vectors, lifetime engagement stats
- Computed by Flink (real-time slice) and Spark/Iceberg (daily slice), merged at read

#### `video_features` (online, served from feature store; Redis-backed)
- Static: creator embedding, audio embedding, hashtag embeddings, duration bucket
- Dynamic: last-1h CTR, last-1h completion rate, last-24h follow-on-from-video rate, current "trending" score
- Updated by Flink every few seconds for dynamic features

#### `embeddings` (vector DB — Vespa, ScaNN, FAISS, or proprietary)
- Two indices: `user_embeddings` (600M × 256-dim) and `video_embeddings` (100M × 256-dim)
- ANN with HNSW or IVF-PQ; p99 query latency < 10ms for top-1000
- Sharded by ID range; each shard fully RAM-resident

#### `engagement_events` (Kafka topic, then Iceberg sink)
- Topic: `events.video.engagement`
- Partitioned by user_id (so a user's events are ordered)
- Retention 7 days in Kafka; sunk to Iceberg for offline training

#### `session_state` (Redis, ephemeral)
- Per-session: bloom filter of already-served video IDs
- TTL ~3 hours; rebuilt at session start

### Why these choices

- **Cassandra for `videos`, `user_watch_history`, `creators`**: write-heavy ingestion (uploads, watch events), wide-row friendly, eventual consistency tolerable. Same logic as news feed.
- **Vector DB (Vespa/ScaNN) for embeddings**: ANN search at <10ms over millions of items is impossible with a row store. This is a category that didn't exist 10 years ago and now is non-negotiable for recommendation systems. We pick Vespa for production-readiness + multi-tenant; in-house FAISS-based service is the alternative and what TikTok actually runs.
- **Redis for online feature store**: <1ms p99 reads on tens of features per request. Tens of thousands of QPS. The feature store layer (Tecton/Feast) sits in front of Redis and handles the offline+online merge.
- **Kafka + Flink for engagement pipeline**: 200K events/sec sustained, ordered per user, replayable for backfills. SQS or Kinesis don't have the per-key ordering and are too costly at this volume.

---

## 7. High-level architecture

```
                           ┌──────────────────────────────────────────┐
                           │                  CDN                     │
                           │   (video bytes; out of scope here)       │
                           └──────────────────────────────────────────┘

                           ┌──────────────────────────────────────────┐
                           │               API Gateway                │
                           │   (auth, rate limit, market routing)     │
                           └────────┬───────────────────────┬─────────┘
                                    │ READ                  │ WRITE (events)
                                    ▼                       ▼
                       ┌─────────────────────────┐    ┌─────────────────┐
                       │   Feed Orchestrator     │    │  Event Ingest   │
                       │   (per-request flow)    │    │  (thin proxy)   │
                       └─┬─────┬──────┬─────┬────┘    └────────┬────────┘
                         │     │      │     │                  │
                         │     │      │     │                  ▼
                         │     │      │     │         ┌──────────────────┐
                         │     │      │     │         │  Kafka topic:    │
                         │     │      │     │         │  events.video.   │
                         │     │      │     │         │  engagement      │
                         │     │      │     │         └────────┬─────────┘
                         │     │      │     │                  │
                         │     │      │     │                  ▼
                         │     │      │     │         ┌──────────────────┐
                         │     │      │     │         │  Flink jobs      │
                         │     │      │     │         │  (real-time      │
                         │     │      │     │         │  feature compute)│
                         │     │      │     │         └────┬───────┬─────┘
                         │     │      │     │              │       │
                         │     │      │     │              │       └────► Iceberg
                         │     │      │     │              │              (offline)
                         │     │      │     │              ▼
                         │     │      │     │      ┌──────────────────┐
                         │     │      │     │      │  Feature Store   │
                         │     │      │     │      │  (Tecton/Feast)  │◄─── reads from
                         │     │      │     │      │  Redis online    │     ranking
                         │     │      │     │      │  Iceberg offline │
                         │     │      │     │      └──────────────────┘
                         │     │      │     │
                         │     │      │     │
                         ▼     ▼      ▼     ▼
              ┌─────────────────────────────────────────┐
              │         Candidate Generation            │
              │  ┌──────────┐ ┌──────────┐ ┌─────────┐  │
              │  │ Embed-   │ │ Trending │ │ Cold-   │  │
              │  │ based    │ │ /        │ │ start / │  │
              │  │ (ANN     │ │ Recency  │ │ Pop.    │  │
              │  │  on      │ │          │ │         │  │
              │  │  vector  │ │          │ │         │  │
              │  │   DB)    │ │          │ │         │  │
              │  └──────────┘ └──────────┘ └─────────┘  │
              │                                          │
              │       merge + dedupe + filter            │
              │       (eligibility, blocklist,           │
              │        already-served, market)           │
              │                                          │
              │       → top 1000 candidates              │
              └────────────────┬─────────────────────────┘
                               │
                               ▼
              ┌─────────────────────────────────────────┐
              │           Ranking Service               │
              │  (online inference, GPU-backed)         │
              │   - reads features from Feature Store   │
              │   - multi-objective scoring             │
              │   - returns top 200 ranked              │
              └────────────────┬─────────────────────────┘
                               │
                               ▼
              ┌─────────────────────────────────────────┐
              │         Diversification + Mix           │
              │  - creator-fairness rebalance           │
              │  - topic diversity                      │
              │  - ads slot insertion (P1)              │
              │  - return top 10 to client              │
              └─────────────────────────────────────────┘
```

### Request walk-through

**Read path (feed open / next batch):**
1. Client `GET /v1/feed` → API Gateway (authn, rate-limit, market routing) → Feed Orchestrator.
2. Feed Orchestrator pulls user features from the Feature Store (one batch read, ~30 features, 5ms).
3. **Candidate Generation (parallel, fan-out):**
   - **Embedding-based retrieval** — query vector DB with user embedding; get top-800 video IDs by cosine similarity. ~10ms.
   - **Trending/recency retrieval** — pull top-100 from a global "hot videos" Redis list, filtered by market.
   - **Cold-start/popularity retrieval** — pull top-100 demographic-bucketed popular videos.
   - Merge, dedupe by video_id, filter against (a) user's already-served bloom filter, (b) blocked creators, (c) per-market eligibility, (d) moderation status. Result: ~1000 candidates.
4. **Ranking** — Feed Orchestrator calls Ranking Service with `{user_id, candidate_video_ids[]}`. Ranking Service:
   - Batch-fetches video features for all 1000 candidates from Feature Store (1 round-trip, ~10ms).
   - Runs the model on GPU (batched, ~40ms for 1000 candidates).
   - Returns top-200 with per-objective scores.
5. **Diversification** — soft constraints applied: no more than 2 videos from same creator in top 10, topic diversity, ads slot insertion. Final top-10 returned.
6. Client renders first video (from prefetch); when buffer drops below 5, calls `/v1/feed` again.
7. **Latency budget:** 5ms (gateway) + 5ms (user features) + 30ms (candidate gen, parallel) + 50ms (ranking incl. feature fetch) + 5ms (diversify) = ~100ms p50. Tail dominated by ranking GPU + slowest candidate-gen path.

**Write path (engagement event):**
1. User watches video for 2 seconds, swipes away.
2. Client batches `{play_start, play_end with 2000ms duration, completion_pct=0.05}` and POSTs to `/v1/events`.
3. Event Ingest validates schema, stamps server-side timestamp, publishes to Kafka topic `events.video.engagement`, partition key = user_id.
4. **Flink job 1 (real-time user features):** consumes the partition, updates user's recent-engagement aggregates: rolling 1-min dwell time avg, last-N video embeddings, "skipped fast" counter. Writes to Redis (online slice of Feature Store). End-to-end latency: <5s.
5. **Flink job 2 (real-time video features):** keyed by video_id, updates per-video recent-CTR, completion-rate, follow-on rate. Writes to Redis.
6. **Flink job 3 (already-served bloom filter):** updates session bloom filter so this video isn't re-recommended.
7. **Iceberg sink:** raw events also batched to S3/Iceberg every minute for offline training and analytics.
8. Next time user calls `/v1/feed`, ranking sees the updated user features and the recently-skipped video influences re-ranking.

### Why this two-stage architecture

- **You cannot run a heavyweight model over 100M videos for every feed call.** That's 100M × 600M DAU × 30 calls = absurd. You need a fast retrieval that prunes 100M → 1000 cheaply, then careful ranking on the 1000.
- **You cannot run trivial ranking on 100M.** Embedding-similarity gives you "what's similar," but the ranker is what picks "what will this *specific* user engage with right now given their context." Different mental models.
- **The candidate-gen / ranking split mirrors how every modern recommender works** — Netflix, YouTube, Pinterest all use this. State it explicitly to the interviewer; this is the central design insight.

---

## 8. Deep dives

### Deep dive 1: Candidate generation

> **Interviewer:** "Walk me through what happens at retrieval time when a user opens the app."

**Candidate:**

The retrieval problem: from 100M videos, return the top ~1000 most-likely-to-engage candidates for this user, in <30ms p99. We can't scan; we can't pre-fanout (no follow graph). The only viable path is approximate nearest-neighbor (ANN) lookup in an embedding space, plus a few non-embedding fallbacks.

**Architecture:**

We run **multiple retrieval strategies in parallel** and merge:

1. **Embedding-based (the primary, ~70% of candidates):**
   - Each video has a 256-dim embedding learned from a two-tower model (user-tower + video-tower trained jointly on engagement data).
   - Each user has a 256-dim embedding, refreshed real-time (the running average of their recent engaged videos, plus a daily-batched long-term embedding).
   - Vector DB (Vespa/ScaNN/in-house FAISS) holds the video index. ANN query: "give me top-800 videos by cosine similarity to this user's embedding."
   - HNSW index — log-time lookup, <10ms p99 over 100M items.
   - Index sharded by video_id range; each shard ~10M items, ~10GB RAM.

2. **Trending / recency (~10% of candidates):**
   - Global "trending" list per market (top-1000 videos by recent engagement velocity), refreshed every minute by a Flink job.
   - Stored in Redis sorted set, keyed by `trending:{market}`.
   - Pull top-100 from this list.
   - **Why include this?** Embedding-based recall is biased toward what the user already engages with; we need *fresh* content to break filter bubbles and surface truly viral content the user hasn't seen anything similar to yet.

3. **Cold-start / popularity (~10% of candidates):**
   - Per-demographic-bucket popular videos (e.g., "popular among 18–24 in US right now").
   - For new users with no embedding, this is the *only* signal we have; for existing users, it's a small fraction.

4. **Creator-affinity (~10%, optional):**
   - "This user has liked 3+ videos from creator X in the past week" → pull recent uploads from creator X.
   - This is the only place the system uses anything like a "follow graph" — implicit creator-affinity, not explicit follow.

**Merge & filter:**

After parallel retrieval, we get ~1000 unique candidates. We then apply hard filters:
- Already-served bloom filter (don't repeat in same session).
- Blocked-creator list.
- Per-market content policy.
- Moderation status (only `approved`).
- Visibility (public only).

What's left: ~800–1000 candidates. Pass to ranking.

> **Interviewer:** "How do you keep the user embedding fresh?"

**Candidate:**

User embedding has two components:
- **Long-term (daily batch):** trained nightly on the user's last 90 days of engagement. Stable, captures durable interest. Stored as a 256-dim vector, refreshed once a day.
- **Short-term (real-time):** running average of the embeddings of the last N (say, 50) videos the user engaged with positively, with recency weighting. Updated by Flink every few seconds.
- **Final user embedding** = α × long_term + (1-α) × short_term, with α tuned (typically 0.3–0.5).

The short-term component is what lets a user's feed shift in real-time as they engage. If you suddenly start liking cooking videos in this session, your short-term embedding shifts toward "cooking" and the next ANN lookup pulls cooking-similar candidates.

> **Interviewer:** "What happens when a new video is uploaded? How does it enter the candidate pool?"

**Candidate:**

This is the **new-creator cold-start** problem.

1. New video enters the system (post-transcoding, post-moderation).
2. Embedding is computed from content features (audio embedding, visual embedding from a ResNet/ViT pretrained model, text embedding from caption + hashtags). This gives an *initial* embedding without any engagement data — a "content-based" embedding.
3. Video is inserted into the vector DB index immediately (or within minutes — see below on index updates).
4. The video is now eligible for retrieval, but its *engagement-based* features (CTR, completion rate) are unknown.
5. We **boost exploration**: for the first ~1000 impressions of a new video, we deliberately over-rank it slightly so we collect engagement signal quickly. This is an explicit exploration term in the ranking model.
6. After ~1000 impressions, we have meaningful CTR/completion data; the embedding gets refined (collaborative-filtering-style update); ranking now uses real engagement features. The video either takes off (positive signal → escalate exposure) or fades.
7. This is how TikTok turns a 0-view upload into a 10M-view viral hit in 24h: the system *deliberately* funnels traffic to new content to discover its quality fast.

> **Interviewer:** "How is the vector DB updated with new videos? Can you do it online?"

**Candidate:**

Two-tier strategy:
- **Hot tier (live updates):** small in-memory index (~1M items, last 24h of uploads), updated within seconds of a new upload. Queried in parallel with the main index, results merged.
- **Main tier (batch rebuild):** full 100M index rebuilt nightly. Streaming inserts into HNSW are theoretically possible but operationally fragile — index drift, recall degradation. Cleaner to rebuild and hot-swap.
- **Index hot-swap:** new index built on a separate fleet, validated against production queries (recall@1000 must match within 1%), then DNS/router swap. Old index drained, then released.
- This means a new video might be in the hot tier for up to 24 hours before joining the main index. Acceptable.

### Deep dive 2: Online ranking + feature store

> **Interviewer:** "OK, candidate gen returns 1000 videos. Now what? How does ranking work?"

**Candidate:**

Ranking is the most expensive single component in the system, and it's where ML quality lives or dies.

**Model:**

We're predicting multiple objectives, not one. Each candidate gets multiple scores:
- P(like)
- P(complete watch)
- P(share)
- P(follow creator from this video)
- P(comment)
- P(skip in <2 sec) — *negative* signal; we want this low
- E[watch duration]

These are combined into a single **utility score**:
```
score = w_like × P(like)
      + w_share × P(share)
      + w_complete × P(complete)
      + w_follow × P(follow)
      - w_skip × P(skip)
      + w_dwell × E[watch_duration]
```
The weights are learned from a higher-level objective (long-term retention / time-spent), tuned via offline replay + online A/B.

**Model architecture:** historically gradient-boosted trees (XGBoost) on hand-engineered features; modern systems use a two-tower deep neural network (the same architecture as candidate gen, but a heavier ranking-specific head). Production-grade is typically a hybrid: deep model with cross-features.

**Inference:** TensorFlow Serving or Triton, GPU-backed. Batch inference over all 1000 candidates in a single forward pass — vectorized.

**Latency budget for ranking:**
- Feature fetch: 10ms (one batch read to Feature Store for 1000 video features + 1 user features object)
- Forward pass: 30–40ms on a single GPU for batch-of-1000
- Post-processing + return: 5ms
- Total: ~50ms p99

**Feature store (the unsung hero):**

The Feature Store is what makes online ranking possible. Without it, every ranking call would do 1000 separate Cassandra reads to fetch video features. Instead:

- **Online slice (Redis-backed):** all features needed at inference time, keyed by entity ID. Tens of features per entity, sub-ms reads.
- **Offline slice (Iceberg-backed):** historical features for training. Same logical schema as online — *consistency between training and serving* is critical (the "training-serving skew" problem).
- **Feature freshness varies by feature:**
  - `video.last_1h_ctr` — Flink-updated, ~5s lag
  - `video.last_24h_completion_rate` — Flink-updated, ~30s lag (longer aggregation window)
  - `user.recent_engaged_videos` — Flink-updated, <5s lag
  - `user.demographic_bucket` — daily batch (Spark), 24h lag
  - `video.audio_embedding` — computed once at upload, immutable
- **The Feature Store layer (Tecton or Feast)** provides a unified API: "give me these features for this entity," and it transparently merges online + offline.

> **Interviewer:** "What if the model is bad? Like, you push a new version and it tanks engagement?"

**Candidate:**

This is the most common operational failure in a recommendation system, and the answer is not "test better." It's:

1. **A/B testing is the law.** Every model change is rolled out to 1% of users for 1–2 weeks before promotion. Quality metric is **retention** (D1, D7, D30), not just engagement — engagement-optimization can hurt retention (clickbait spirals).
2. **Canary detection on aggregate metrics:** for every model serving variant, watch p(skip), avg watch duration, and CTR every 5 minutes. If any drops > 5% relative to baseline, **automatic rollback** within 15 minutes.
3. **Shadow serving:** new model version runs in parallel with old, scoring the same candidates; we log both predictions and the actual user action. Lets us measure offline whether new model would have made better picks, before any user sees it.
4. **Holdout cells:** ~1% of users always get "no personalization beyond demographic" — gives us a clean control to measure long-term lift of the personalization stack.
5. **Reverse experiments:** before retiring an old model, run a *reverse* A/B for 4 weeks — give some users the old model, see if their long-term retention goes down. Catches metric-gaming.

### Deep dive 3: The engagement signal pipeline

> **Interviewer:** "You mentioned engagement events flow back into features in seconds. Walk me through that pipeline."

**Candidate:**

The signal pipeline is what makes the For-You feed *adaptive* — without it, you have a static recommender that doesn't react to user behavior. With it, the user can see their feed shift in 1–2 swipes.

**Pipeline:**
```
client batches events ─► Event Ingest service ─► Kafka topic
                              │                      │
                              │ (validation,         │ (partitioned by user_id,
                              │  schema check,       │  ordered per user)
                              │  authn)              │
                              │                      │
                              ▼                      ▼
                            DLQ                   Flink jobs (3 parallel)
                                                      │
                            ┌───────────┬─────────────┼─────────────┐
                            ▼           ▼             ▼             ▼
                       UserFeatures VideoFeatures BloomFilter   IcebergSink
                       (Flink)     (Flink)        (Flink)       (Flink)
                            │           │             │             │
                            ▼           ▼             ▼             ▼
                          Redis       Redis         Redis         Iceberg
                          (online)   (online)      (session)      (offline)
```

**Per-component:**

- **Event Ingest:** thin proxy (Go service, ~20K QPS per host). Validates schema, applies per-user rate limits (a single user shouldn't generate >1000 events/sec — sign of bot). Writes to Kafka. Returns 202 in <10ms.
- **Kafka:** topic partitioned by user_id, RF=3, retention 7 days. Sustains 200K events/sec sustained, 600K peak. ~50 brokers.
- **Flink job: UserFeatures**
  - Keyed by user_id.
  - Maintains rolling state per user: last 50 engaged videos (with embeddings), last 1-min dwell-time avg, recent-skip counter.
  - On every event, recomputes the running aggregates and writes to Redis.
  - End-to-end latency from event publish to Redis update: 2–5s p99.
- **Flink job: VideoFeatures**
  - Keyed by video_id.
  - Maintains: 1-hour CTR, 1-hour completion rate, 1-hour like rate, 1-hour follow-from-this-video rate, total view count.
  - Updates Redis every ~30s for hot videos (more frequent for trending).
- **Flink job: SessionBloomFilter**
  - Keyed by session_id.
  - Maintains a bloom filter of video IDs already served to this session. Used at retrieval-time to filter candidates.
  - Lazy-expiring; sessions older than 3 hours are dropped.
- **Iceberg sink:** every event also batch-written to Iceberg every minute for offline training. This is the source of truth for model retraining.

> **Interviewer:** "What if Flink falls behind?"

**Candidate:**

Lag is observable as Kafka consumer lag. We alert at >30s lag.
- **Backpressure within Flink:** if a downstream Redis write is slow, Flink's source operator slows; Kafka backs up.
- **At >5 min lag:** real-time features become stale. Ranking degrades to using last-known good values. We surface this as a `feature_staleness_warning` flag in the ranking response and slightly down-weight stale features in the model (the model has explicit features for "freshness of feature X" — meta-features).
- **At >30 min lag:** auto-scale Flink (more task slots), and consider lowering the ingest rate by sampling at the Event Ingest layer (drop low-value events like progress-pings, keep play_end and likes).
- **Critical:** never lose events. Kafka retains 7 days. We catch up.
- **User-visible impact:** ranking is slightly behind; user might see a video they "should have" already trained the system to skip. Acceptable degradation.

> **Interviewer:** "Can you guarantee ordering of events for a single user?"

**Candidate:**

Yes — Kafka partition by user_id gives us per-user ordering. The user's events are processed in publish order by a single Flink subtask. This matters for state correctness: a `like` event must be applied after the corresponding `play_end`. Without ordering, you'd see weird state transitions.

Cross-user ordering doesn't matter (and would cost us partition parallelism).

### Deep dive 4: Cold-start (new users + new videos)

> **Interviewer:** "A user just installed the app. They have no history. What do they see?"

**Candidate:**

Two cold-start problems, distinct:

**(a) New user cold-start.**

A new user has no embedding, no watch history, no engagement signal. Their first feed call must still feel personalized within 5–10 swipes.

Strategy:
1. **Demographic priors:** at signup, we collect age, country (from IP), device type, possibly OS-language. We have precomputed "popular videos per demographic bucket" updated daily.
2. **Geographic trends:** show what's trending in their country/region right now (filtered by their language preference).
3. **Diversity over personalization:** the first 10 videos should span topics (comedy, music, dance, food, sports) so we sample the user's reaction.
4. **Aggressive learning rate on the first session:** the user-embedding update from the first 5–10 engagements is weighted heavily — we want to converge to their interest space fast. Within ~20 swipes, the personalization signal dominates.
5. **Explicit onboarding (optional):** "pick 5 topics you're interested in" at install. TikTok historically didn't do this; some competitors do. The system works without it.

After 50 engagements, the user has a usable embedding; after 500, they're indistinguishable from any other user in terms of recommendation quality.

**(b) New video / new creator cold-start.**

Already covered partly in Deep Dive 1. Reiterating:
1. **Content embedding** computed at upload time from audio + visual + text. Gives the video an *initial* position in embedding space without needing engagement data.
2. **Exploration boost:** new videos get an artificial uplift in ranking score for their first ~1000 impressions. We deliberately allocate eyeballs.
3. **Fast feedback loop:** within 1000 impressions (which can happen in minutes for the right embedding placement), we have real CTR and completion-rate data.
4. **Escalation:** if early signal is positive, expand the audience. If negative, fade. This is the "TikTok algorithm" mythology in concrete form — it's not magic; it's exploration + exploitation with a tight feedback loop.
5. **Creator reputation:** new creators with a track record (e.g., a creator with 5 prior videos averaging 50% completion) get a head start; first-time creators get base exploration.

> **Interviewer:** "How do you balance exploration and exploitation? You said you boost new videos — at what cost?"

**Candidate:**

There's a budget. Roughly: 5–10% of every user's feed is "exploration content" — content we're less confident the user will like, but we want to learn from. The rest is exploitation.

This is implemented as a **stochastic ranking step**: with probability ε, we override the top candidate with a random pick from a lower-ranked-but-needs-exploration pool. ε might be 0.05 globally.

The cost: a small short-term engagement hit (these videos perform worse on average). The benefit: a far better recommender in the long run, because we never get stuck in a filter bubble. This is why TikTok's algorithm "feels alive" — it's deliberately injecting noise.

This is also why **A/B tests must run for weeks**, not days: short-term engagement metrics can be misleading; the long-term effect of better exploration shows up in retention.

---

## 9. Tradeoffs & alternatives

### Decision: Two-stage retrieval (candidate gen → ranking)

| Option | Pros | Cons | When to pick |
|---|---|---|---|
| **Single-stage (one heavy model over all videos)** | Simpler, one model to operate | Impossible at 100M corpus + 18B plays/day; would need exaflops of compute | Tiny corpus (<100K items) |
| **✅ Two-stage (cheap retrieval, careful ranking)** | Bounded compute per request, scales to billion-item corpora | More moving parts; training-serving skew between stages | **Any modern recommender at scale** |
| **Three-stage (retrieval → coarse ranking → fine ranking)** | Even better quality at scale | More complexity, harder to tune | Megascale (YouTube, Meta — they actually do this) |

We pick two-stage. Three-stage is the natural evolution at the next order of magnitude; for now, the second stage handles 1000 candidates and can afford a careful model.

### Decision: Vector DB for candidate generation

| Option | Pros | Cons | When to pick |
|---|---|---|---|
| **Elasticsearch with vector field** | Familiar, hosted, integrated with text search | Slow at >10M items, recall drops at high recall@k | Smaller corpora, hybrid text+vector |
| **Pinecone / Weaviate (managed)** | Managed, fast onboarding, good APIs | Vendor lock-in; cost at hyperscale; opacity on internals | Up to ~10M items, prefer-managed |
| **Vespa** | Multi-tenant, mature, good operators, hybrid retrieval (vector + keyword + structured) | Java; ops complexity | **Production-grade in-house** |
| **ScaNN (Google)** | State-of-art ANN quality | Needs to be wrapped in a service yourself | Building from scratch, max recall |
| **✅ In-house FAISS-based service** (with HNSW or IVF-PQ) | Full control, optimal cost at hyperscale, can co-locate with ranking | Build cost (months of eng), ops responsibility | TikTok-class scale; 100M+ items |

I'd default to Vespa for a "this team has 10 engineers and needs to ship in 6 months" answer. For TikTok-actual scale, it's an in-house service.

Defending: at 100M items × 600M users × 30 retrievals/day, the ANN query is on the critical path of every feed open. A 10ms vs 30ms tail latency directly translates to user-visible responsiveness. Owning the index gives us flexibility to do things like co-located retrieval+ranking (skip a network hop), per-shard fanout strategies, and custom recall/latency tradeoffs.

### Decision: Online feature store backed by Redis

| Option | Pros | Cons | When to pick |
|---|---|---|---|
| **No feature store (read directly from primary stores)** | Simplest, no extra infra | 1000 reads per ranking call kills your DB; training-serving skew rampant | Prototype only |
| **Just Redis (no Tecton/Feast layer)** | Cheap, fast | No standard schema, no offline parity, every team reinvents | Small ML org |
| **Feast (open source)** | Open standard, multi-store, strong community | Requires you to operate; less feature-rich | Mid-scale ML orgs |
| **Tecton (managed)** | Production-grade feature engineering, strong offline-online consistency, built-in monitoring | Vendor lock-in; cost | Larger ML orgs without infra capacity |
| **✅ In-house feature store on top of Redis + Iceberg** | Full control, integrated with internal tooling | Build cost | TikTok-class scale |

The point is that *something* implementing the feature-store contract (offline-online parity, point-in-time correctness, batched reads at inference time) is non-negotiable. Whether it's Tecton, Feast, or in-house is an org-size decision.

### Decision: Kafka + Flink for engagement pipeline

| Option | Pros | Cons | When to pick |
|---|---|---|---|
| **Kinesis + Lambda** | Managed, low ops | Kinesis throttling at this volume; Lambda cold starts | AWS-native, lower volume |
| **Pulsar + Pulsar Functions** | Multi-tenant, geo-replication built-in | Less mature ecosystem than Kafka | Multi-region first |
| **✅ Kafka + Flink** | Battle-tested at this volume; rich windowing/state in Flink; per-key ordering | Ops overhead; tuning is real work | Stream-processing at scale |
| **Spark Streaming** | Familiar to Hadoop shops | Microbatch (latency floor ~1s); state mgmt weaker | Already-Spark teams |

Defending: 200K sustained, 600K peak events/sec; per-user ordering is required for state correctness; Kafka's partition-key ordering + Flink's exactly-once-state-with-checkpointing is the canonical pattern. We've operated this at scale; the ops burden is known.

### Decision: Eventual consistency for features; monotonic reads per session

| Scenario | Consistency model | Mechanism |
|---|---|---|
| User sees their just-liked video reflected in ranking | Eventual (≤5s) | Flink writes to Redis online slice |
| User doesn't see the same video twice in a session | Monotonic / read-once | Per-session bloom filter, populated synchronously in Feed Orchestrator |
| Engagement counts on a video (likes, plays) | Eventual + asymptotic | Aggregated, displayed value can be 100s off from truth briefly |
| New upload appears in feed | Eventual (≤5 min) | Hot-tier vector index updated within seconds; full index rebuilt nightly |

---

## 10. Defending architectural choices — interviewer attacks

> **Interviewer:** "Why not just precompute every user's feed nightly and serve it from a cache? Like the news feed problem."

**Candidate:** Three reasons:
1. **Engagement-driven adaptation is the product.** If I precompute the feed, I can't react to in-session signal. A user starts watching cooking videos *this session*; the fed should pivot. Precomputed feeds can't do this.
2. **The "session bloom filter" alone makes precompute impractical.** The next video to show depends on what we've already shown — that's session state, not user state.
3. **Storage doesn't shake out.** 600M users × 200 precomputed videos × 100 bytes = 12 TB just for the cache, and it's stale within seconds of the user's first swipe.

The news feed problem can precompute because the input set (your follow graph's posts) is bounded and doesn't shift in real-time. Here, the input set is 100M and the user's interests are shifting per swipe.

> **Interviewer:** "Why not just use chronological order from a follow graph? Why this whole ML thing?"

**Candidate:** Because the product *is* the algorithm. The hypothesis underlying TikTok is that "social graph" is a worse signal than "implicit interest" for short-form video. Empirically validated — TikTok's growth came from this insight. Building this product means building this algorithm.

Architecturally: even if we wanted "follow graph + ranked," we'd still need ranking, which means we'd still need the feature store, the engagement pipeline, and all of section 8. The candidate generation strategy would change (replace embedding-ANN with follow-graph-fanout), but the rest stands.

> **Interviewer:** "Why a single global ranking model? Wouldn't different markets need different models?"

**Candidate:** They do, and we run different models per market. The architecture supports this — Ranking Service is per-cell (per-market), with cell-specific models. The candidate-gen vector DB is also per-cell. The only globally shared layer is the embedding *training* pipeline, which can incorporate cross-market data when it doesn't violate residency rules.

> **Interviewer:** "What if the vector DB is down or returns nothing?"

**Candidate:** Graceful degradation. Candidate gen has multiple parallel retrieval strategies; if embedding-based returns empty, the trending + popularity-based strategies still return ~200 candidates. Quality drops (less personalized), but the feed stays up.

If *all* retrieval strategies fail (which would mean Redis + vector DB + Cassandra all down) — we serve a hardcoded "popular videos" emergency feed from a static fallback file in S3. Never show a blank app.

> **Interviewer:** "Why do you trust a 256-dim embedding to capture user interest?"

**Candidate:** I don't trust it as the *sole* signal — that's why ranking exists, and ranking uses dozens of features beyond the embedding (recency, demographic, engagement-velocity, content-type). The embedding gives us a *retrieval* signal: "the candidate set is plausibly relevant." Ranking does the actual sorting.

Empirically, two-tower embeddings with sufficient training data give recall@1000 of 0.95+ against a held-out set of "videos this user later engaged with positively." That's high enough that the candidate set rarely misses the right answer.

> **Interviewer:** "If a creator wants to game the system — buy fake views, generate engagement — how do you defend?"

**Candidate:** This is a real adversarial problem. Defenses:
1. **Bot detection at Event Ingest:** rate limits, device fingerprinting, behavioral anomaly detection (humans don't watch 10K videos/hour). Suspicious accounts shadow-banned (their engagement events are ingested but down-weighted to 0 in feature pipelines).
2. **View-farm detection:** clusters of accounts that engage only with one creator, with similar device fingerprints. Identified by an offline graph-clustering job.
3. **Engagement-quality features:** the ranking model has features like "fraction of engaging users with high reputation" — if a video's engagement is dominated by low-rep accounts, it gets less of a boost.
4. **Reputation system:** creators and viewers both have reputation scores that decay with bad behavior (creators) or grow with engagement that correlates with broader engagement (viewers).
5. **This is a never-ending arms race.** Mention an offline trust-and-safety pipeline that retrains adversarial classifiers weekly.

---

## 11. Failure modes & mitigations

| Failure | Detection | Blast radius | Mitigation | Recovery |
|---|---|---|---|---|
| Ranking model serves bad version (drop in engagement) | Auto-canary on aggregate metrics every 5 min | Whatever % of traffic the bad model is serving | Auto-rollback within 15 min on metric drop | Investigate offline; re-train if needed |
| Feature store stale (Flink lag) | Kafka consumer lag metric | Ranking quality degrades, not user-blocking | Use cached features with staleness flag; ranker has explicit staleness features | Auto-scale Flink; if persistent, sample lower-value events at ingest |
| Vector DB index drift (recall degraded) | Daily recall@k eval against held-out set | Candidate gen quality drops | Rebuild index nightly; canary new index for 6h before swap | Roll back to previous index version |
| Candidate gen returns empty / errors | Per-strategy success rate metric | Degraded feed quality for affected users | Multiple parallel retrieval strategies; fall back to popularity-based feed | Investigate per-strategy failure |
| Vector DB shard down | Per-shard latency / error rate | Affected ID range candidates missing | Replicas (RF=3); other shards still serve | Restore failed shard from snapshot |
| Kafka cluster lost | Producer error rate | Engagement events not flowing → features go stale | Multi-AZ Kafka; client retries; events buffer in client up to 5 min | Multi-AZ failover; investigate |
| Redis (online feature store) cluster lost | Cache miss rate spike | Ranking can't fetch features → falls back to default features | Redis replicas; partial degradation rather than total | Restore from snapshot |
| Hot video (viral spike) hammers feature store | Per-key Redis QPS metric | Read storm on a single key | Local cache in Ranking Service for top-1000 hot videos (TTL 30s) | Pre-emptive caching of viral content |
| Cold-start fallback returns same generic feed for everyone | Diversity metric across new-user cohort | New users see homogeneous content | Demographic-bucketed cold-start (not single global list); diversity term in cold-start ranker | Tune buckets; A/B |
| Adversarial signals (view farms) inflate a video's engagement | Anomaly detection on engagement velocity vs viewer-reputation | Low-quality content surfaces in ranking | View-quality features; reputation-weighted engagement | Trust-and-safety review; demote |
| Multi-region failover (one cell down) | Cell health check | Users in that market routed to fallback or degraded | Per-cell HA (in-cell replicas); for hard-cell-down, route to nearest healthy cell with warning | Restore cell |
| Schema change to ranking features breaks training-serving consistency | Offline eval shows lift drop, online eval flat | Silent quality degradation | Feature schema versioning; training pipeline + serving must use same version | Schema versioning enforced at Feature Store layer |
| Client sends bad events (bug in app) flooding Kafka | Per-client error rate | Kafka backs up; features get noisy data | Per-client rate limiting at Event Ingest; schema validation rejects malformed | Push fixed app version |
| GPU inference cluster scaling lag during traffic spike | Inference queue depth | Ranking p99 spikes; user-visible latency | Pre-provisioned headroom; HPA on queue depth (not just CPU); distilled "lightweight" model as a fallback | Auto-scale; emergency capacity |

---

## 12. Productionization

### Pre-launch
- [ ] Dark launch: shadow real production traffic to new Feed Orchestrator for 2 weeks. Compare candidate sets and ranking outputs against existing system.
- [ ] Offline eval: replay last 30 days of engagement events through new model, compute counterfactual metrics (if we'd shown video X instead of Y, would the user have engaged?).
- [ ] Load test to 5× peak (~625K feed-API calls/sec). Verify ranking GPU pool scales.
- [ ] Chaos test: kill random Flink task managers during load; verify event pipeline catches up.
- [ ] Game day: simulate vector DB shard failure; verify candidate gen degrades gracefully.

### Rollout
- [ ] Per-user-bucket feature flag. Start at 1%, wait 1 week (retention takes time to surface), check D1/D7 metrics.
- [ ] Per-region rollout: smallest market first, then rest.
- [ ] 1% → 5% → 25% → 50% → 100%, with **at least 1 week bake at each step** — this differs from the news feed rollout cadence, because the metric of record (retention) takes longer to move than engagement.
- [ ] Auto-rollback if: D1 retention drops > 1% relative, ranking p99 > 200ms for 10 min, or 5xx rate > 0.5%.

### Capacity
- [ ] Reserve 50% headroom on Ranking GPU pool — viral events can 2x traffic in hours.
- [ ] Auto-scale Feed Orchestrator on QPS, target 60% CPU.
- [ ] Vector DB: provisioned for 3× current corpus (we add 100K videos/day).
- [ ] Kafka: partitions provisioned for 5× current event volume; repartitioning costs us a day.
- [ ] Flink: parallelism scaled by Kafka lag; alert at lag > 30s.

### Migration (if replacing existing system)
- [ ] Dual-serve: both old and new ranking systems run; old serves users, new shadows.
- [ ] Comparison job: sample feeds, evaluate predicted vs actual engagement.
- [ ] Gradual shift of read traffic to new system.
- [ ] Old system kept warm for 4 weeks of rollback insurance — longer than news feed because regressions in recommendation quality compound (filter-bubble effects accumulate).

### Cost
- Estimated $/DAU: ~$0.05–0.10/DAU/month for storage + compute + ML inference. CDN egress (out of scope here) is much larger; total stack is ~$0.50/DAU/month.
- **GPU inference for ranking is the dominant compute cost.** Levers:
  - Model distillation (train a smaller student model on the big teacher, similar quality at 1/10 the FLOPs).
  - Batch inference for non-real-time features (e.g., batch-update user embeddings nightly instead of inference-time).
  - Cache feed for inactive users (tail of users opens the app once a week; pre-compute their next session).
  - Quantization (INT8 inference on GPU; 2–4x throughput at small accuracy cost).

### Compliance
- Per-market data residency: EU users' data stored in EU cell only.
- GDPR right-to-erasure: user deletion propagates to watch history, embeddings, all derived features. Audit trail of derived data lineage.
- Per-country content policy: video `eligible_markets` filter at retrieval; some content available only in some markets.
- Children's privacy (COPPA): under-13 accounts get a separate, more restrictive ranking model with no behavioral tracking; manually curated content pool.

---

## 13. Monitoring & metrics

### Business KPIs (the only metrics that ultimately matter)
- **Time-spent per session** (the primary engagement metric)
- **D1 / D7 / D30 retention** (the metric that matters more than engagement; engagement-optimization can hurt this)
- **Sessions per DAU**
- **Videos watched per session**
- **Creator retention** (do creators keep uploading? they're the supply side)

### Service-level (SLIs)

- **Feed open**: p50, p95, p99, p99.9 latency. Per-region. SLO: 99.95% of feed opens < 300ms.
- **Swipe-to-next** (client-measured): p99 < 50ms (served from prefetch). Reported via client telemetry.
- **Engagement event ingest**: p99 < 50ms acceptance, error rate.
- **Engagement event-to-feature-store latency**: p99 < 10s.
- **Ranking inference**: p99 < 80ms.
- **Candidate gen recall**: recall@1000 against held-out positives, daily eval.

### Component metrics

#### Feed Orchestrator
- Per-stage latency: candidate gen, ranking, diversification
- Cache hit rates on user features, video features, session bloom
- Fallback rates (which retrieval strategies returned 0)

#### Candidate Generation
- Per-strategy: latency, hit count, downstream-engagement rate
- Vector DB query latency (per shard)
- Hot-tier vs main-tier hit rate
- Empty-candidate-set rate (alarm: >0.1%)

#### Ranking Service
- p99 inference latency
- Model version distribution (during A/B)
- Feature staleness per feature (max lag observed)
- GPU utilization, queue depth
- Fallback-to-default-features rate
- Per-objective score distribution (catch model drift)

#### Feature Store (online slice)
- Read latency p99 (target <2ms)
- Hit rate
- Staleness per feature (lag from event to availability)
- Memory utilization

#### Vector DB
- Query latency p99
- Recall@k (against eval set, daily)
- Index size, RAM utilization
- Index build duration (nightly job)

#### Kafka (engagement events)
- Producer rate, error rate
- Consumer lag per Flink job (alert >30s)
- Broker disk/network utilization
- Partition skew (detect bot users hammering one partition)

#### Flink jobs
- Per-job lag (alert >30s)
- State size, checkpoint duration
- Task failure rate, restart count
- Watermark progression

#### Cassandra (videos, watch_history)
- Read/write latency per node
- Replication lag
- Disk utilization
- Compaction queue depth

#### ML model health
- **Online metrics:** daily aggregate engagement/retention vs control
- **Offline metrics:** per-objective AUC, log-loss, NDCG
- **Calibration:** predicted P(like) vs actual P(like) — are predictions well-calibrated?
- **Drift detection:** feature distribution shift week-over-week

### Alert routing
- **Page** (24/7): SLO breach (slow burn or fast burn), Kafka lag >5min, ranking GPU pool saturated, regional outage
- **Ticket** (next business day): cache hit rate degraded, feature staleness warning, single-shard issues
- **Dashboard-only**: cost, throughput trends, model AUC drift, exploration ratio

### Dashboards
1. **Feed health** — RED metrics, regional, latency percentiles, error budget burn
2. **ML pipeline** — model versions in flight, A/B status, AUC trends, calibration
3. **Engagement pipeline** — Kafka → Flink → Feature Store, end-to-end timing, lag
4. **Vector DB** — per-shard, query latency, recall
5. **Business** — DAU, time-spent, retention curves, content-creation rate

---

## 14. Security & privacy

- **Per-video visibility** enforced at retrieval (private/unlisted videos filtered out)
- **Block lists**: blocked-creator filter at retrieval; blocking a creator is read-after-write consistent
- **PII**: usernames public, emails private; engagement data is sensitive PII (tied to user_id)
- **Embedding privacy**: user embeddings encode behavioral data; treat as PII. Don't expose via APIs.
- **Adversarial defense**:
  - Bot detection at Event Ingest (device fingerprint, behavioral anomaly)
  - View-farm detection via offline graph clustering
  - Reputation systems for creators and viewers
- **Compliance**:
  - GDPR right-to-erasure: user data + derived features (embeddings, history) purged on request, with audit
  - COPPA: under-13 mode has restricted recommendations, no behavioral tracking
  - Per-country content policies enforced at retrieval (e.g., political content filters in some markets)
- **Account takeover**: anomaly detection on session geography, device changes, sudden engagement-pattern shifts

---

## 15. Cost analysis

Rough order-of-magnitude:

| Component | Rough cost driver | Optimization lever |
|---|---|---|
| GPU inference (ranking) | GPU-hours | Model distillation, quantization, batched inference |
| Vector DB (candidate gen) | RAM (whole index in memory) | IVF-PQ compression for less-hot videos |
| Cassandra (videos, history) | Storage + replication × retention | Tiered storage; old watch history → cold storage |
| Redis (feature store online slice, session state) | RAM | TTL aggressive eviction; only top features online |
| Kafka (engagement events) | Disk + retention | 7-day retention enough; older → Iceberg cheap |
| Flink (real-time pipeline) | CPU + state RAM | Tune state TTL; sample low-value events |
| CDN egress (video bytes — out of scope) | Bytes served | Adaptive bitrate, edge caching, regional CDN |

Largest cost in *this* design (excluding video CDN): **GPU inference for ranking.** A single GPU does ~10K candidates/sec of inference; at 125K feed-API calls/sec × 1000 candidates = 125M candidates/sec to score = ~12,500 GPUs. Real number is a bit less due to batching efficiency, but still thousands.

This is why distillation matters. A 10× smaller model with 1% quality loss is a 90% cost reduction on the dominant line item. State-of-art recommender ML orgs put serious effort here.

---

## 16. Open questions / what I'd validate with PM

1. **What's the retention vs. engagement tradeoff appetite?** If PM says "max engagement at all costs," we tune the ranker one way; if "balance long-term retention," we tune differently. Both must be measured for weeks before committing.
2. **How much exploration budget can we spend?** 5% reduces short-term engagement by ~2% but is critical for long-term recommendation quality. PM owns this tradeoff.
3. **Per-market model variants — same architecture, different tuning, or different architectures?** Operationally, same architecture is way easier; quality-wise, different architectures might win in some markets (e.g., Japan video preferences differ).
4. **Cold-start onboarding flow — interest survey or no?** Affects new-user metrics significantly; worth A/B'ing.
5. **Ads integration** — when, where, what fraction of slots? Affects ranking objective formulation.
6. **Creator-side controls** — should creators be able to see their content's recommendation eligibility? Their performance per-bucket? This is product/policy, not engineering.

---

## 17. Staff-level scorecard for this problem

| Signal | Did the candidate... |
|---|---|
| ✅ Identified two-stage retrieval as the central architectural insight | Within first 10 min, framed it explicitly |
| ✅ Quantified the corpus + retrieval problem | "100M videos × 600M users × 30 retrievals = need ANN" calculation made explicit |
| ✅ Treated the feature store as a first-class architectural concern | Not just "we have features"; named freshness, online-offline parity, staleness |
| ✅ Designed for the engagement signal loop in seconds | Kafka → Flink → Redis pipeline with lag SLO; not "we'll batch overnight" |
| ✅ Volunteered failure modes before being asked | Bad model rollout, vector DB drift, Flink lag, adversarial signals |
| ✅ Treated cold-start as a separate sub-problem with two halves | Not lumped in with ranking; demographic priors + content embeddings + exploration boost |
| ✅ Picked databases with defended reasons | Vector DB, Cassandra, Redis, Kafka — each justified by workload, not "because it scales" |
| ✅ Identified A/B testing as the **operational** core | Not just "we'd test it"; named retention not engagement, weeks not days, holdouts, reverse experiments |
| ✅ Discussed cost in concrete terms | Identified GPU inference as dominant; named distillation as lever |
| ✅ Discussed multi-market / cellular | Per-cell models, per-market data residency |
| ✅ Closed with a wrap-up speech | Restated dominant constraints, choices, what to do next |

### What separates Staff from Senior on *this* problem

A Senior candidate will:
- Correctly draw two-stage retrieval (candidate gen → ranking)
- Mention vector DBs and embeddings
- Pick reasonable databases
- Mention A/B testing

A Staff candidate will additionally:
- Explicitly name **training-serving skew** as a feature-store invariant and design for it
- Identify that **retention, not engagement, is the metric of record** and that A/B tests need weeks because of this
- Volunteer the **exploration vs exploitation budget** and put a number on it (5–10%)
- Design for **adversarial signals** (view farms, bot engagement) as a first-class concern
- Name **cold-start as two distinct problems** (new user, new creator/video) with different solutions
- Identify **per-market model variants** as the natural next step
- Discuss **canary auto-rollback on aggregate engagement metrics** as a fundamental ops pattern for ML
- Quantify the **GPU inference cost** and name distillation as the lever
- Make the **ε-exploration ratio a config**, not a constant — and mention dynamic adjustment under load

---

## 18. Wrap-up speech (template)

> "To wrap up: the dominant constraints here are (1) retrieving from a 100M-video corpus per user with no follow graph to cheat with, and (2) closing the engagement-feedback loop in seconds so the feed adapts in-session. The architecture answers both: two-stage retrieval (vector-DB ANN for candidate gen, GPU-served ranking on the top 1000), backed by a feature store with offline-online parity, and a Kafka+Flink engagement pipeline that updates user features within 5 seconds of an event. The two pieces I'd worry about in production are (1) training-serving skew silently degrading model quality — mitigated by feature schema versioning, point-in-time-correct training, and continuous calibration monitoring, and (2) GPU inference cost dominating the bill — mitigated by model distillation and quantization. To productionize: per-user-bucket flag rollout starting at 1%, with at least a week bake at each step because retention takes time to surface, auto-rollback on aggregate engagement drop, holdout cells for long-term measurement. If we had another 30 minutes, I'd deep-dive (a) the per-market cellular architecture and cross-market content rules, or (b) the ML training pipeline — how often we retrain, point-in-time correctness, and the offline replay framework for evaluating new models before they go live."

---

## 19. Market-based isolation (cellular architecture)

> Bring this up in the last 5 minutes. For TikTok specifically, this is **not optional** — the company famously runs entirely separate stacks in China (Douyin) and globally (TikTok), driven by regulation and ban-state exposure (India). Frame it as *"cellular isolation is the design TikTok actually runs; here's why."*

### What & why

A **market** is a unit of regulatory + business + geographic isolation: US, EU, India (banned), Brazil, Japan, China (Douyin). A **cell** is a self-contained, independently-deployable, per-market replica of the entire stack — its own Cassandra, its own Redis, its own vector DB, its own Flink jobs, its own ranking models, its own creator pool.

Drivers (this problem amplifies the news-feed drivers):

1. **Regulatory / data residency** — GDPR (EU), India's DPDP, China's PIPL, Brazil's LGPD. Hard requirements. India outright banned TikTok in 2020 — the cell architecture made it possible to wall off India and continue serving rest-of-world without disruption.
2. **Content policy variance** — what's allowed in Brazil isn't allowed in Saudi Arabia. Per-market moderation rules, per-market eligible-content lists. Implementing this globally is intractable; per-cell is natural.
3. **National-security pressure** — the US has explicit pressure on TikTok to silo US user data. The cell architecture is the technical answer.
4. **Per-market ML models** — Japan's content preferences are dramatically different from US's. A US-trained model performs worse in Japan than a Japan-trained model, even with the same architecture.
5. **Blast radius** — a bad ranking model push in EU doesn't affect US.
6. **Latency** — US users hit US cell, served from US edge. Saves trans-Pacific RTT.
7. **Operational independence** — each cell rolls out, scales, monitors independently. Major Chinese New Year traffic spike in CN cell doesn't fight US cell for capacity.

### Architecture

```
        ┌───────────────────────── Global Edge ─────────────────────────┐
        │  CDN + Anycast + GeoDNS routes user to home cell              │
        └────────┬─────────────────┬──────────────────┬─────────────────┘
                 │                 │                  │
       ┌─────────▼────────┐ ┌──────▼────────┐  ┌──────▼────────┐
       │   US Cell        │ │   EU Cell     │  │  CN Cell      │
       │ ┌──────────────┐ │ │ ┌───────────┐ │  │ ┌───────────┐ │
       │ │ Feed Orch.   │ │ │ │Feed Orch. │ │  │ │  Douyin   │ │
       │ │ Cand. Gen    │ │ │ │Cand. Gen  │ │  │ │  (entirely│ │
       │ │ Ranking (US- │ │ │ │Ranking    │ │  │ │  separate │ │
       │ │  trained)    │ │ │ │ (EU-      │ │  │ │  stack +  │ │
       │ │ Cassandra    │ │ │ │  trained) │ │  │ │  features)│ │
       │ │ Vector DB    │ │ │ │ Cassandra │ │  │ │           │ │
       │ │ Redis        │ │ │ │ Vector DB │ │  │ │           │ │
       │ │ Kafka/Flink  │ │ │ │ Redis     │ │  │ │           │ │
       │ └──────────────┘ │ │ │ Kafka/Fl. │ │  │ └───────────┘ │
       │                  │ │ └───────────┘ │  │               │
       └─────────┬────────┘ └──────┬────────┘  └───────────────┘
                 │                 │
                 └─────────────────┘
                         │
              ┌──────────▼──────────┐
              │  Global Coordination │
              │  - User Directory    │
              │  - Content Sync      │
              │    (filtered by      │
              │     market policy)   │
              │  - Aggregate metrics │
              └──────────────────────┘

    (CN cell does NOT participate in global coordination — fully separate.)
```

### Routing

- Each user has a **home cell** assigned at signup (country at signup).
- Global User Directory (replicated KV) maps `user_id → home_cell`. Cached at the edge.
- API Gateway looks up home cell, routes the request.
- A user traveling abroad is still served by their home cell (not the cell of their current location) — this preserves data residency.

### The hard part: cross-cell content sharing

When User A (US cell) uploads a video, can User B (EU cell) see it? Three policies:

| Approach | How | Pros | Cons |
|---|---|---|---|
| **Full content replication** | Every video metadata + embedding replicated to every cell | Cross-market viral content possible | Storage duplication, residency violations possible |
| **Selective replication by policy** | Replicate only "globally eligible" content; mark per-market eligibility at upload | Respects residency by exclusion; reasonable storage | Complex policy engine; replication lag |
| **Cross-cell pull** | EU cell's Feed Orchestrator queries US cell for additional candidates if needed | No replication; clean residency | Read amplification across regions; failure coupling |

**My pick for For-You feed:** **Selective replication.** Most engagement is intra-market (>85%). Globally viral content (the "world video" — a dance trend, a meme) gets replicated to all cells where it's policy-eligible. Cross-cell pull is reserved for explicit user-requested cross-market browsing, not the For-You feed.

### What changes in the design

- **Per-cell vector DB** — each cell has its own video index, populated from its own corpus + replicated globally-eligible content.
- **Per-cell ranking models** — trained on cell-local engagement data. Architecture is shared; weights are not.
- **Per-cell Kafka** — engagement events stay in-cell; cross-cell aggregation happens via batch ETL on stripped-of-PII data.
- **Per-cell ML training pipelines** — the offline training cluster for each cell is separate; no cross-cell PII leakage.
- **Per-cell rollouts** — feature flags now have a `cell` dimension. We can roll a feature to a single small market first.
- **Per-cell on-call** — at TikTok scale, each cell has its own on-call rotation.
- **Aggregate metrics** — global business dashboard pulls anonymized aggregates from each cell. No raw cross-cell data.

### Cell tradeoffs vs single global system

| Dimension | Global | Cellular | Net |
|---|---|---|---|
| Total infra cost | Lower (no duplication) | Higher (per-cell stacks) | +30–50% cost |
| ML model quality per market | Worse (one-size-fits-all) | Better (per-market tuning) | Cellular wins |
| Compliance | Hard | Easier | Cellular wins, often non-negotiable |
| Operational complexity | Lower | Significantly higher | Cellular tax |
| Blast radius | 100% on regional outage | 10–20% per cell | Cellular wins |
| Time-to-deploy a feature globally | Fast | Slow (per-cell sequencing) | Global wins |

### When NOT to do cellular

- Pre-scale: under 50M global users, the operational tax exceeds the benefit.
- Single regulatory regime, no national-security exposure.
- A single ML team can't operate N parallel training pipelines.
- The product requires global engagement (not relevant here — TikTok is intrinsically per-market in interest patterns).

### What "Staff signal" sounds like here

> "I'd structure this as cellular from day one. TikTok's history makes this non-negotiable: when India banned the app in 2020, the cellular structure made it possible to disable the IN cell while keeping global service running. The hard part isn't the cells — it's the cross-cell content policy: which videos replicate globally, which stay local. I'd default to selective replication based on per-market eligibility, with the policy engine as a separate service. The cost is real — 30–50% more infra, separate ML training pipelines per market, cell-aware deploys — but it's the only architecture that survives a regulator audit and a regional outage at the same time. Per-market model variants are also where the *quality* upside lives — Japan and Brazil don't want the same recommendations, and a per-cell model trained on local engagement beats a global model by measurable retention margins."
