# 01 — Design a News Feed (Twitter / Instagram / Facebook)

> Canonical Staff-level HLD problem. The interviewer cares less about the answer and more about whether you correctly identify that **fanout strategy** and the **celebrity problem** are the only things that matter, and you spend your time there.

---

## 0. Problem statement (as the interviewer would drop it)

> **Interviewer:** "Design the news feed for a social network like Twitter or Facebook. Users follow other users, and they see posts from people they follow ranked in some order in their home timeline. Walk me through how you'd build it."

That's all you get. Your job: scope it, not solve it whole.

---

## 1. Clarifying questions

Pick 5–6. Don't fire them as a checklist — interleave with the interviewer's responses.

> **Candidate:** "Before I design, let me scope this. A few questions:"
>
> 1. **"Is this read-heavy or write-heavy?"** — *Expect: read-heavy, ~100:1 read-to-post ratio. Drives fanout-on-write.*
> 2. **"What's the rough scale — DAU and posts per user per day?"** — *Expect: 200M DAU, 2 posts/user/day, 50 feed-loads/user/day.*
> 3. **"Are posts text-only, or do they include media?"** — *Critical for storage and CDN. Assume text + image + short video for realism.*
> 4. **"Is the feed strictly chronological, or ranked?"** — *If ranked, ML ranking becomes a major component. Assume ranked for Staff-level.*
> 5. **"What's the latency target for opening the feed?"** — *Expect: p99 < 200ms for the first paint.*
> 6. **"Are there limits on follow count? Most-followed account size?"** — *Drives the celebrity problem. Expect: top accounts have 100M+ followers.*
>
> **Out of scope (state explicitly):**
> - DM / messaging
> - Notifications (separate problem — see `10-notification-system.md`)
> - Ads insertion (mention as a known integration point, don't design)
> - Authentication
> - Search
> - Trends/explore feed
>
> **Candidate:** "I'm going to design the **home timeline** read path and the **post creation → fanout** write path, with ranking as a black-box service I'll just describe the interface to. Sound right?"

> **Interviewer:** "Yes."

---

## 2. Functional requirements

| Priority | Requirement |
|---|---|
| P0 | User can post a text+image (and optionally short video) post |
| P0 | User can follow / unfollow another user |
| P0 | User opens app and sees a ranked feed of posts from people they follow |
| P0 | Feed pagination (scroll for more) |
| P1 | Like, comment, share — engagement actions |
| P1 | Real-time updates (new post appears at top of feed without refresh) |
| P2 | Hide / mute / report a post |
| Out | DMs, search, ads, trends, notifications, profile pages |

---

## 3. Non-functional requirements

| Dimension | Target | Justification |
|---|---|---|
| Availability | 99.95% (~22 min/month) for read; 99.9% for write | Read path is the front door — must be up |
| Read latency | p50 < 100ms, p99 < 200ms (feed open) | Mobile app teardown happens at ~250ms perceived delay |
| Write latency | p99 < 500ms (post acceptance), fanout async | Users tolerate post-creation taking longer than feed-open |
| Consistency | Eventual for feed; read-your-writes for the poster's own profile | Missing a post for 5 sec is fine; not seeing your own post is not |
| Durability | No post loss after 200 OK | Posts are user-generated content — losing them is unacceptable |
| Scale | 200M DAU, 400M MAU, 1B total users | Industry-standard scale assumption |
| Geography | Global, multi-region active-active | Latency-bound from anywhere |

---

## 4. Capacity estimation

> **Candidate:** "Let me estimate aggressively — round numbers. Stop me if you want me to dig deeper."

### Users
- 200M DAU
- 1B total users
- Avg follow count: 200 (median ~150, but the distribution has a long tail)
- Top 1% of accounts followed by 100M+ users (celebrity problem)

### Writes (posts)
- 200M DAU × 2 posts/user/day = **400M posts/day**
- Per second avg: 400M / 86,400 ≈ **4,600 posts/sec**
- Peak (3×): **~14K posts/sec**

### Reads (feed loads)
- 200M DAU × 50 feed-loads/day = **10B feed-loads/day**
- Per second avg: 10B / 86,400 ≈ **115K feed-loads/sec**
- Peak (3×): **~350K feed-loads/sec**
- **Read:write ratio ≈ 25:1** (closer to 100:1 if you count items rendered per load)

### Storage
- Avg post: 300 bytes text + metadata + 200KB image (90% have images) + occasional video
- Text+metadata: 400M × 1KB = **400 GB/day** for post records
- Media: 400M × 0.9 × 200KB = **72 TB/day** for media (in S3-class storage)
- Per year (text): ~150 TB. Per year (media): ~26 PB.
- Replication 3×: ~80 PB/year for media
- **Decision driver:** media goes to object storage (S3/GCS) with CDN; post records go to a sharded wide-column store

### Fanout work (this is the punchline of the estimation)
- 400M posts/day × 200 followers avg = **80B fanout writes/day**
- Per second: ~1M fanout writes/sec — but **highly skewed**
- One celebrity post (100M followers) = 100M fanout writes from a single post
- **This single number is why fanout-on-write doesn't work for everyone, only for non-celebrities. The hybrid model exists because of this number.**

### Bandwidth
- Feed open: ~50 posts × 1KB metadata = 50KB per feed
- Egress: 350K feed-loads/sec × 50KB = **17.5 GB/s** at peak (just metadata; CDN eats the media)

---

## 5. API design

> **Candidate:** "I'll keep this minimal — public REST surface."

### Read
```
GET /v1/feed?cursor={opaque}&limit=20
→ 200 { items: [{post_id, author_id, text, media_urls[], created_at, ranking_signals{...}}], next_cursor }
```

- Cursor is opaque — encodes the ranking position, not just a timestamp. Lets us swap chronological → ranked without API changes.

### Write
```
POST /v1/posts
  body: { text, media_ids[], visibility }
→ 201 { post_id, created_at }
```

- `media_ids[]` references pre-uploaded media (separate `POST /v1/media` endpoint via signed URL to upload directly to object storage).

### Social graph
```
POST /v1/follow { target_user_id }
DELETE /v1/follow/{target_user_id}
GET /v1/followers/{user_id}?cursor=&limit=
GET /v1/following/{user_id}?cursor=&limit=
```

### Engagement (P1)
```
POST /v1/posts/{post_id}/like
POST /v1/posts/{post_id}/comment { text }
```

### Anti-patterns to avoid
- Don't return raw timestamps — return cursor.
- Don't return author full object — return author_id, let the client batch-resolve via `/v1/users/batch`.
- Don't make the feed endpoint do ranking inline — it should fetch from a precomputed timeline.

---

## 6. Data model

### Tables / collections

#### `users` (sharded by user_id, e.g., DynamoDB / Spanner)
| Column | Type | |
|---|---|---|
| user_id | bigint (Snowflake) | PK |
| username | string | unique |
| profile_meta | json | |
| created_at | timestamp | |

#### `posts` (sharded by post_id; secondary by author_id)
| Column | Type | |
|---|---|---|
| post_id | bigint (Snowflake — sortable by time) | PK |
| author_id | bigint | secondary index (per-author timeline) |
| text | string (≤500 chars) | |
| media_ids | list<string> | references in object storage |
| created_at | timestamp | |
| visibility | enum | |

#### `follows` (sharded by follower_id; **also** stored sharded by followee_id)
- Edge stored twice. Why? Two query patterns:
  - "Who do I follow?" — query by follower_id (used at fanout-on-read time)
  - "Who follows me?" — query by followee_id (used at fanout-on-write time)
- Don't try to make one index serve both — denormalize.

#### `user_timelines` (the precomputed feed) — this is the hot table
- Sharded by `user_id`
- Per user: a sorted list (Redis sorted set, or Cassandra wide-row keyed by user_id with clustering key on score)
- TTL ~1 week of items, cap at ~1500 items
- Score = ranking score (so it's pre-ranked, not just chronological)

#### `home_timeline_cache` (Redis, hot tier)
- Top ~200 items per user, sorted by score
- LRU eviction on inactive users
- Hit rate target: 95%+

### Why these choices

- **Cassandra/ScyllaDB for `posts` and `user_timelines`**: write-heavy, wide rows, eventual consistency tolerable, linear scale. The alternative — DynamoDB — works but locks us in and is expensive at this scale. Spanner is overkill (we don't need multi-row transactions on this hot path).
- **Snowflake IDs for `post_id`**: time-sortable, no central allocator, lets us range-scan per-author timelines by id without a separate timestamp index.
- **Redis for hot timelines**: 95% of feed-opens hit Redis; the warm tier (Cassandra) absorbs misses.

---

## 7. High-level architecture

```
                  ┌────────────────────────────────────────────────────┐
                  │                       CDN                          │
                  │              (Cloudflare / CloudFront)             │
                  └────────────────────────┬───────────────────────────┘
                                           │
                                  ┌────────▼─────────┐
                                  │   API Gateway    │  (auth, rate limit, mTLS)
                                  └────┬─────────┬───┘
                                       │         │
            ┌──────────────────────────┘         └────────────────────────┐
            │ READ                                                  WRITE │
            ▼                                                              ▼
   ┌───────────────────┐                                       ┌─────────────────┐
   │  Feed Service     │                                       │  Post Service   │
   │  (timeline read)  │                                       │  (ingest)       │
   └─────┬─────────┬───┘                                       └────┬────────────┘
         │         │                                                 │
         │         │ miss                                            │ persists post
         │         ▼                                                 ▼
         │  ┌─────────────┐    ┌─────────────────────────────┐  ┌─────────┐
         │  │ Cassandra   │    │  posts table                │  │ Kafka   │
         │  │ user_       │    │  (Cassandra/Scylla)         │  │ topic:  │
         │  │ timelines   │    └─────────────────────────────┘  │ posts.  │
         │  └─────────────┘                                     │ created │
         ▼                                                      └────┬────┘
   ┌─────────────┐                                                   │
   │ Redis       │                                                   ▼
   │ home_       │                              ┌───────────────────────────────┐
   │ timeline_   │                              │      Fanout Service           │
   │ cache       │                              │   (per-post worker pool)      │
   └─────────────┘                              └────┬───────────────┬──────────┘
                                                     │               │
                                              non-celeb        celebrity?
                                                     │               │
                                                     ▼               │
                                            ┌─────────────┐          │ skip fanout;
                                            │  Followers  │          │ pull at read
                                            │  service    │          │
                                            └──────┬──────┘          │
                                                   │ chunked iter    │
                                                   ▼                 ▼
                                       writes user_timelines    (feed merger
                                       (Cassandra + Redis)      pulls celeb
                                                                posts at read
                                                                time)

                            ┌──────────────────────────┐
                            │   Ranking Service        │  (called by Feed Service
                            │  (ML model, feature      │   on cache miss / on
                            │   store, online inf)     │   warming)
                            └──────────────────────────┘
```

### Request walk-through

**Write path (post creation):**
1. Client `POST /v1/posts` → API Gateway → Post Service.
2. Post Service writes to `posts` (Cassandra, RF=3, CL=QUORUM). Returns 201 to client.
3. Post Service publishes `post.created` event to Kafka.
4. Fanout Service consumers pick up event:
   - Look up follower count of author from `users` cache.
   - **If follower count < 100K (non-celebrity):** fan out — for each follower, append `(post_id, score)` to `user_timelines[follower_id]` and push to Redis cache.
   - **If follower count ≥ 100K (celebrity):** skip fanout entirely. Mark the post in a separate `celebrity_posts` table indexed by author + time.
5. Total write latency to user: ~80–150ms. Fanout completes async in seconds for non-celebrities.

**Read path (feed open):**
1. Client `GET /v1/feed` → API Gateway → Feed Service.
2. Feed Service:
   - Reads top N items from Redis `home_timeline_cache[user_id]`.
   - On cache miss: read from Cassandra `user_timelines[user_id]`, populate Redis.
   - **Fetch celebrity posts on the fly:** look up users this person follows whose follower count > 100K, query `celebrity_posts` for last X hours per celebrity, merge into the timeline.
   - Optionally call Ranking Service to re-score (or trust precomputed score).
3. Hydrate `post_id` → full post by batch-reading `posts` (or from a posts cache).
4. Return.

### Why this hybrid

- **Pure fanout-on-write** breaks at celebrities: one Taylor Swift post = 100M fanout writes, which is a 5-minute lag and a thundering herd on `user_timelines`.
- **Pure fanout-on-read** breaks at scale: every feed open does N follow-graph queries + N timeline merges. That's 200 follower lookups × 350K feed-loads/sec = 70M reads/sec on the post store. Doesn't fly.
- **Hybrid (push for non-celebs, pull for celebs)** matches the workload: 99.99% of users have small follower counts (so push works), and the long tail is exactly the celebrities (so pull works for them).

This is the single most important insight in this problem. State it explicitly.

---

## 8. Deep dives

### Deep dive 1: Fanout pipeline scaling

> **Interviewer:** "Walk me through what happens when a user with 5 million followers posts."

**Candidate:**

5M followers, threshold is 100K → not technically a celebrity in the simple model, but borderline. Let's pick a hard threshold: ≥1M followers = celebrity, do pull. So 5M = celebrity.

But suppose the threshold were 10M and this user were 5M (so push). Then:

1. Post Service writes the post, emits to Kafka. Latency to user: ~100ms. ✅
2. Fanout Service must do 5M writes to `user_timelines`. At 50K writes/sec/worker (Cassandra batch writes), that's 100 seconds to drain — across however many parallel workers we throw at it.
3. Concurrency: we shard the Kafka topic by `post_id`, so a single post is processed by one consumer group instance. To parallelize within a post, we **fan out the fanout** — break the follower list into chunks of 5K and emit `fanout_chunk` events to a second Kafka topic, processed by a wider worker pool.
4. So the actual flow:
   ```
   posts.created → FanoutDispatcher (looks up follower count, decides push vs pull)
                                                    │
                                                    │ if push:
                                                    ▼
                                       fanout.chunk (200 chunks of 25K)
                                                    │
                                                    ▼
                                       FanoutWorker pool (1000 workers)
                                                    │
                                                    ▼
                                       writes to user_timelines + Redis
   ```
5. With 1000 workers at 50K writes/sec each = 50M writes/sec capacity. 5M-follower post drains in 0.1 sec. 100M-follower (celebrity) post would take 2 sec — but we don't push for celebs, we pull.

> **Interviewer:** "What if the Fanout Service falls behind?"

**Candidate:**
- Lag is observable as Kafka consumer lag — alert at >30s lag.
- Backpressure mechanism: if lag exceeds 5 min, **temporarily lower the celebrity threshold** dynamically. A user with 500K followers becomes "pull" instead of "push", reducing fanout volume.
- Critical: never lose a fanout — Kafka retains. We catch up.
- User-visible impact: posts may take longer to appear in followers' feeds. This is a known graceful degradation. We expose a SLO: "post visible to follower within 30s, p99 within 2 min".

### Deep dive 2: The celebrity / pull problem

> **Interviewer:** "How does pulling celebrity posts at read time scale?"

**Candidate:**

Two sub-problems:

**(a) The read amplification.** A user follows on average 200 people. Of those 200, suppose 5 are celebrities. On feed open we now do:
- 1 read of `user_timelines[user_id]` (precomputed)
- 5 reads of `celebrity_posts[celeb_id]` (recent posts)
- Merge + rank

The 5 reads are bounded — won't grow with system scale, only with how many celebs a user follows. So it's O(celeb-follow-count), not O(total-follow-count). Acceptable.

**(b) The read hotspot.** 100M users follow Taylor Swift. Each opens their feed → reads `celebrity_posts[taylor_swift]`. That's 100M reads/sec on a single key.

Mitigations:
- **Redis cache for celebrity timelines** — TTL 30s. 100M reads/sec × 30s windows means ~30 reads/sec hit Cassandra, not 100M.
- **Edge caching** — celebrity timelines change predictably (couple posts/day), perfect for CDN-style caching.
- **Per-region replication** — replicate celebrity timeline cache to every region.
- **Stale-while-revalidate** — serve cached version even if it's 30s old; refresh asynchronously.

> **Interviewer:** "What's the consistency story here?"

**Candidate:** A user might see their friend's post 1 second after creation (push, fast) but a celebrity's post might take 30s to appear (pull + cache TTL). That's an asymmetry users don't notice — they expect celebrities to feel "broadcasty" and friends to feel "live".

### Deep dive 3: Ranking

> **Interviewer:** "You mentioned the feed is ranked. Where does ranking happen?"

**Candidate:**

Ranking is a service boundary, not an inline component. The architecture supports two modes:

**Mode A — Precomputed ranking (push-time):**
- At fanout, FanoutWorker calls Ranking Service to score `(viewer_id, post)` and writes the score into `user_timelines` as the sort key.
- Pros: feed-open is just a sorted-set range query. Very fast.
- Cons: ranking happens once at fanout; if ranking signals change (e.g., post engagement spikes), the score is stale.

**Mode B — Refresh on read:**
- Read top 200 from Redis (precomputed scores), call Ranking Service on the candidate set, re-rank, return top 50.
- Pros: scores reflect latest signals.
- Cons: adds 30–50ms to feed open; needs Ranking Service to handle 350K QPS at p99 < 50ms.

**My pick:** Mode B for the top 50 candidates, Mode A for everything below. Hybrid.

Ranking Service internals (black-boxed but mention):
- Online inference of a learned model (gradient boosted trees or a two-tower NN)
- Features pulled from a feature store (Feast / Tecton)
- Feature freshness: user-side ~minutes, post-side ~seconds (engagement signals)
- A/B testing framework hooked in — different model versions to different user buckets

> **Interviewer:** "What if the Ranking Service is down?"

**Candidate:** Fall back to precomputed scores (Mode A only). Feed degrades to "yesterday's ranking" but stays up. SLO breach is logged, on-call paged. **Never** show a blank feed because ranking is down.

### Deep dive 4: Real-time updates ("new posts" pill)

> **Interviewer:** "How does the 'X new posts' indicator at the top of the feed work?"

**Candidate:**

Two approaches:

1. **Long polling**: client `GET /v1/feed/since?cursor=<last_seen>` every 30s. Simple, expensive.
2. **Push via WebSocket / SSE**: client connects to a notification gateway, server pushes "you have N new" when fanout completes.

Pick #2 at scale.

- Connection layer: a WebSocket gateway service (sticky session, sharded by user_id). Each user has 1 persistent connection.
- When fanout writes to `user_timelines[user_id]`, it also publishes a small notification to a per-user channel.
- The connection gateway delivers it.

Bandwidth: 200M concurrent WebSockets is huge. In practice you only need this for users actively in the app — maybe 20M concurrent. Each user holds 1 connection — that's 20M concurrent sockets, sharded across ~2000 gateway hosts (~10K connections/host). Doable.

---

## 9. Tradeoffs & alternatives

### Decision: Hybrid push/pull fanout

| Option | Pros | Cons | When to pick |
|---|---|---|---|
| **Pure push (fanout-on-write)** | Read-time is trivial (single sorted-set read) | Celebrity posts cause write storm | Small social network, no celebrities, write-amenable |
| **Pure pull (fanout-on-read)** | Write-time is trivial (just store the post) | Read-time does N follow-graph fetches × N timeline reads | Tiny system, or systems where users follow very few accounts |
| **✅ Hybrid (push for non-celebs, pull for celebs)** | Optimizes both common cases | Complexity: two code paths, threshold tuning, feed merge at read | **Any social network at scale** |

### Decision: Cassandra for `user_timelines`

| Option | Pros | Cons | When to pick |
|---|---|---|---|
| **PostgreSQL** | Familiar, ACID | Doesn't scale to 80B writes/day | Smaller scale or transactional requirements |
| **MongoDB** | Easy schema | Not optimized for this write volume; rebalancing pain | When schema flexibility matters more than write throughput |
| **DynamoDB** | Managed, scales | Vendor lock-in, costly per-RCU at this scale | AWS-native team without ops capacity |
| **✅ Cassandra / ScyllaDB** | Tunable consistency, wide-row optimal for timeline pattern, linear scale | Operational complexity, secondary indexes weak | Wide-row write-heavy workloads at hyperscale |
| **HBase** | Strong consistency, good for write-heavy | Operationally heavy (HDFS, Zookeeper) | If you're already in a Hadoop shop |

Picking Cassandra: I'd defend by saying "the wide-row + clustering-key pattern maps perfectly to a per-user timeline, and we already have ops experience. The eventual consistency is fine because the user already accepts seconds of lag for fanout."

### Decision: Kafka for fanout pipeline

| Option | Pros | Cons | When to pick |
|---|---|---|---|
| **SQS** | Managed, cheap | No ordering, limited replay, message size limits | Simpler systems, AWS-native |
| **RabbitMQ** | Flexible routing | Doesn't scale to millions/sec | Smaller scale, complex routing |
| **✅ Kafka** | Persistent, partitionable, replayable, ordered per partition | Ops overhead | Stream pipelines at scale |
| **Kinesis** | Managed Kafka-equivalent | AWS lock-in, throttling at high QPS | AWS-native streaming |

### Decision: Redis for hot timeline cache

Alternatives: Memcached (no sorted sets, smaller ecosystem), Couchbase (overkill).
- Sorted sets are the killer feature here — `ZRANGE` + `ZADD` map directly to the feed read/write.

### Decision: Eventual consistency for feed; read-your-writes for own profile

| Scenario | Consistency model | Mechanism |
|---|---|---|
| User sees friend's post in feed | Eventual (≤30s) | Fanout via Kafka, eventually delivered |
| User sees their **own** post on their own profile | Read-your-writes | Sticky session OR: client optimistically prepends post to feed |
| User sees their own like | Read-your-writes | Optimistic UI |
| Like count on a post | Eventual + asymptotic accuracy | Aggregated, may show 1240 vs 1241 briefly |

---

## 10. Defending architectural choices — interviewer attacks

> **Interviewer:** "Why not just store everything in PostgreSQL with read replicas and call it a day?"

**Candidate:** At 80B fanout writes/day, no single PG primary handles the write volume even with sharding. We'd need to shard PG ourselves, replicate per shard, manage failover per shard — at that point we've reinvented Cassandra poorly. PG wins on consistency and joins; we don't need either on this hot path. I would use PG for the social graph ground truth (or a graph store), but not for the timeline.

> **Interviewer:** "Why not just have one fanout service that does both push and pull?"

**Candidate:** They have radically different operational characteristics. Push is bursty (one post → millions of writes); pull is steady-state (every read does a few celeb fetches). Mixing them couples their failure modes — a celebrity-post fanout storm shouldn't degrade celebrity-pull reads. So: separate services, separate Kafka topics, separate scaling.

> **Interviewer:** "Why a celebrity threshold of 100K? Why not 10K or 1M?"

**Candidate:** It's an empirical tuning parameter. Set it based on: at what follower count does push cost (write amplification) exceed pull cost (read amplification × cache miss rate)? Industry rule of thumb is 100K–1M. The exact number is a knob we'd tune in production based on observed Kafka lag, fanout latency p99, and pull-side cache hit rate. **The threshold should be a config value, not a constant** — and ideally dynamic (lower it under load).

> **Interviewer:** "What if a non-celebrity (10K followers) and a celebrity (100M) post at the same time? Same priority?"

**Candidate:** No. The celebrity's post is a single durable write to `celebrity_posts` — instantly available to pull. The 10K-follower post enters the fanout queue. Smaller posts are written **before** they show up in followers' feeds. We could prioritize, but with sufficient parallelism it doesn't matter — both complete in <5s.

> **Interviewer:** "How do you handle a user who unfollows someone?"

**Candidate:** Two options:
- **Lazy:** keep their existing posts in the timeline, just stop adding new ones. On feed open, filter out posts from unfollowed authors. Pro: cheap. Con: timeline includes unwanted posts for a while.
- **Eager:** sweep `user_timelines[user]` and remove posts from `unfollowed_user`. Pro: clean. Con: expensive, especially for users who follow/unfollow a lot.

I'd go lazy. The filter at read time is O(unfollowed-set), which is small per user.

> **Interviewer:** "What if Redis goes down?"

**Candidate:** The cache is in front of Cassandra, not authoritative. Worst case: every read goes to Cassandra. Cassandra alone handles ~10K reads/sec/node; we'd need to be sized for this fallback (we should be running at <30% utilization for headroom). We'd see latency degrade from 50ms p99 to ~200ms p99 during an outage, but the system stays up.

If we lost an entire Redis cluster: graceful degradation. We do not block reads waiting for a missing cache.

---

## 11. Failure modes & mitigations

| Failure | Detection | Blast radius | Mitigation | Recovery |
|---|---|---|---|---|
| Post Service down | 5xx rate alarm | Posting blocked globally | LB removes unhealthy instances; multi-region failover | Auto-rollback if recent deploy; on-call investigates |
| Fanout consumer lag spike | Kafka lag metric | Posts visible to followers delayed | Auto-scale workers; lower celeb threshold dynamically; SLO breach notification | Drain backlog; investigate root cause (slow Cassandra writes? GC?) |
| Cassandra `user_timelines` shard down | Per-shard latency / error rate | 1/N of users have degraded reads (their feed cache miss path) | Read from replica (RF=3, QUORUM allows 1 down); writes route to remaining replicas | Replace failed node, stream-rebuild |
| Redis hot key evicted (celebrity post) | Cache hit rate dip | Brief read storm to celebrity_posts table | Cassandra absorbs; we have provisioned headroom for this case | Re-warm cache via background warmer |
| Ranking Service down | 5xx rate from Ranking | Feed degrades to precomputed scores | Feed Service has explicit fallback: "if ranking call fails, use stored score" | Ranking team pages, restores |
| Kafka cluster lost | Producer error rate / consumer lag | Posts not fanning out; ingest backs up | Posts Service buffers in local WAL; on Kafka recovery, replays | Multi-AZ Kafka prevents this; if it happens, run from standby cluster |
| Multi-region failover | Region health checks | Half of users fail over to other region | Active-active design; DNS / Anycast traffic shift | Automated; full failover < 60s |
| Hot user (celebrity unexpectedly viral) | Per-author write rate metric | One Cassandra partition gets hammered | Per-author rate limit at Post Service; or mirror writes to a "hot author" shard | Auto-detect and route |
| Schema change | Pre-deploy | Service crash on deploy | Forward+backward compatible schemas (additive only); deploy in phases | Roll back if needed |
| DDoS / spam posting | WAF + per-user rate limiter | Could overwhelm fanout | Rate limit at edge (e.g., 100 posts/hour/user); shadow ban suspicious accounts | Rate limit + manual review queue |

---

## 12. Productionization

### Pre-launch
- [ ] Dark launch: shadow real production traffic to new Feed Service for 2 weeks. Compare diff in feed contents (must be identical for chronological mode).
- [ ] Load test to 5× peak (1.75M feed-loads/sec). Verify tail latency holds.
- [ ] Chaos test: kill random Cassandra nodes during load test. Verify feed still serves.
- [ ] Game day: simulate Kafka partition outage, verify fanout backlog recovers without data loss.

### Rollout
- [ ] Feature flag per user-bucket. Start at 1%, wait 24h, check SLOs.
- [ ] 1% → 5% → 25% → 50% → 100%, with 24h bake at each step.
- [ ] Per-region rollout: smallest region first (latency-sensitive market with fewer celebrities).
- [ ] Auto-rollback if: feed-open p99 > 400ms for 5 min, or 5xx rate > 0.5%, or fanout lag > 5 min.

### Capacity
- [ ] Reserve 50% headroom on every component. Black Friday / event spikes need it.
- [ ] Auto-scale Feed Service on QPS, target 60% CPU.
- [ ] Cassandra: provision for 3× current write volume — adding capacity takes hours, not minutes.
- [ ] Kafka: provision partitions for 5× current — repartitioning is expensive.

### Migration (if replacing existing system)
- [ ] Dual-write to old + new for a month.
- [ ] Comparison job: sample feed-loads, diff old vs new responses. Investigate any divergence.
- [ ] Cut over read traffic gradually; old system stays for 2 weeks of rollback insurance.
- [ ] Decommission old.

### Cost
- Estimated $/DAU: ~$0.02/DAU/month for storage+compute (industry public numbers). Egress dominates if media isn't behind a CDN.
- Cost levers: cold-storage tiering for posts older than 90 days; aggressive cache TTLs; sample (not full) ranking on the long tail of inactive users.

### Compliance
- GDPR: user deletion must propagate to all timeline copies. Use a "delete tombstone" flag in `posts`; readers filter on it. Async sweep removes data.
- Data residency: EU users' posts stored in EU region only.
- Audit log of admin actions.

---

## 13. Monitoring & metrics

### Business KPIs
- Feed sessions per DAU
- Time-in-feed
- Post creation rate
- Engagement rate (likes/comments/shares per impression)

### Service-level (SLIs)
- **Feed open** (the front door): p50, p95, p99, p99.9 latency. Error rate. Per-region.
- **Post creation**: p99 latency, error rate.
- **Fanout end-to-end** (post created → visible to follower): p50, p99.
- **SLO**: 99.95% of feed opens < 200ms p99; 99% of fanouts < 30s end-to-end.

### Component metrics

#### Feed Service
- Cache hit rate on `home_timeline_cache` (target ≥95%)
- Calls to Ranking Service: latency, error rate, fallback rate
- Time spent fetching from Cassandra on miss
- Time spent merging celebrity posts

#### Post Service
- Write latency to `posts` table
- Kafka publish latency
- Pre-validation rejection rate (spam, length)

#### Fanout Service
- Kafka consumer lag (alert if > 30s)
- Push throughput (writes/sec)
- Push tail latency (p99 fanout completion)
- Celeb-vs-non-celeb split (track threshold effectiveness)
- Failed fanout writes (retried via DLQ)

#### Cassandra
- Read/write latency per node, per shard
- Replication lag (cross-region)
- Disk utilization
- Compaction queue depth
- Hint queue (during node failures)
- Per-shard write skew (detect hot users)

#### Redis
- Hit rate (per cache: timeline cache, celeb cache)
- Eviction rate
- Memory utilization
- Connection count
- Slow log (latency outliers)

#### Kafka
- Producer publish rate, error rate
- Consumer lag per group
- Broker disk/network utilization
- Partition skew (detect hot partitions = hot users)

#### Ranking Service
- p99 inference latency
- Model version distribution (during A/B)
- Feature staleness
- Fallback-to-precomputed rate

### Alert routing
- **Page** (24/7 wake someone): SLO breach (slow burn or fast burn), regional outage, Kafka lag > 5 min
- **Ticket** (next business day): cache hit rate degraded, single-node Cassandra issue, schema mismatches
- **Dashboard-only**: cost, throughput trends, capacity headroom

### Dashboards
1. **Feed health** — RED metrics, regional, latency percentiles, error budget burn
2. **Write pipeline** — Post → Kafka → Fanout, end-to-end timing, lag
3. **Cassandra** — per-cluster, per-shard
4. **Redis** — hit rates, memory, evictions
5. **Business** — DAU, posts/day, feed sessions/DAU, engagement rate

---

## 14. Security & privacy

- Posts are AuthZ'd by `visibility` (public, friends, private) — checked at feed read time, not assumed at fanout time
- Block lists: a blocked user's posts must not appear in your feed. Filter at read time.
- Spam: rate limit posts per user (100/hour); ML spam classifier; shadowban
- Account takeover: anomaly detection on post velocity, geographic logins
- PII: usernames are public, emails are not. Encrypt at rest. Audit access.
- Compliance: GDPR right-to-erasure, CCPA, regional data residency

---

## 15. Cost analysis

Rough order-of-magnitude:

| Component | Rough cost driver | Optimization lever |
|---|---|---|
| Cassandra (timelines, posts) | Storage + replication × retention | Tiered storage; compress; TTL old data |
| Redis (hot cache) | RAM | Aggressive eviction; shrink hot set |
| Kafka | Disk + retention | Short retention (24h enough for fanout) |
| CDN egress | Bytes served | Image optimization (webp/avif); video adaptive bitrate |
| Compute (Feed/Post/Fanout services) | CPU-hours | Autoscaling; reserved instances for baseline |
| Ranking ML inference | GPU-hours / specialized infra | Model distillation; batch inference; quantization |

Largest cost: usually media + CDN egress. Engineering effort goes furthest there.

---

## 16. Open questions / what I'd validate with PM

These are real Staff-level instincts — surfacing what you don't know.

1. Are there strict per-region data-residency requirements (e.g., EU users' data must never leave EU)? Affects multi-region design substantially.
2. What's the engagement tail? If 80% of feed loads come from 20% of users, we can deprioritize the long tail aggressively.
3. Is "ranked feed" personalization model owned by us, or by a separate ML team? Affects service boundary.
4. What's the spam/fraud baseline? Affects fanout throttling and per-user rate limits.
5. Is "see new posts in real-time" (the "X new posts" pill) a P0 launch feature, or P1?

---

## 17. Staff-level scorecard for this problem

| Signal | Did the candidate... |
|---|---|
| ✅ Identified hybrid fanout as the central insight | Within first 10 min, not as an afterthought |
| ✅ Quantified the celebrity problem | "5M-follower post = 5M writes" calculation made explicit |
| ✅ Picked a database with a defended reason | Cassandra/Scylla justified by write pattern, not "because it scales" |
| ✅ Volunteered failure modes before being asked | Mentioned Kafka lag, Cassandra node failure, Ranking fallback |
| ✅ Treated ranking as a service boundary | Did not get sucked into ML model details |
| ✅ Talked about cost in concrete terms | Mentioned $/DAU, identified egress as dominant |
| ✅ Discussed productionization unprompted | Feature flags, dark launch, gradual rollout |
| ✅ Designed for graceful degradation | "If ranking fails, fall back to precomputed scores" |
| ✅ Asked the interviewer at decision points | "Should I deep-dive ranking or fanout?" |
| ✅ Closed with a wrap-up speech | Restated dominant constraints, choices, what to do next |

### What separates Staff from Senior on *this* problem

A Senior candidate will:
- Correctly draw the hybrid push/pull architecture
- Pick reasonable databases
- Mention sharding

A Staff candidate will additionally:
- Make the celebrity threshold a **dynamic config**, not a constant
- Identify that the fanout pipeline itself needs internal fanout (chunking)
- Build graceful degradation **into the design**, not as an afterthought
- Volunteer the read-amplification math for celebrity pulls
- Identify the cost dominator (CDN egress) and propose a lever
- Discuss multi-region active-active as the default, not as an extra
- Propose an explicit migration plan if replacing an existing system

---

## 18. Wrap-up speech (template)

> "To wrap up: the dominant constraints here are read:write asymmetry (~25:1) and follower-count skew (celebrities have 6–7 orders of magnitude more followers than median). The architecture answers both: hybrid push/pull fanout with a celebrity threshold around 1M followers, Cassandra for the timeline store with Redis as a hot cache, Kafka as the fanout pipeline backbone. The two pieces I'd worry about in production are (1) the fanout consumer falling behind during viral events — mitigated by dynamic threshold lowering and worker auto-scale, and (2) the celebrity-pull read hotspot — mitigated by aggressive caching with stale-while-revalidate and edge replication. To productionize: feature-flagged rollout starting at 1%, dark-launched against existing system for 2 weeks, SLO-alerted on feed-open p99 and fanout end-to-end lag. If we had another 30 minutes, I'd deep-dive (a) the ML ranking pipeline and feature store, or (b) multi-region replication and conflict resolution for cross-region follows."

---

## 19. Market-based isolation (cellular architecture) — bonus / "if time permits"

> Bring this up in the last 5 minutes if there's time. It's a "post-Senior" topic that immediately signals you've operated systems at scale. Frame it as: *"If we had more time, the next architectural layer I'd add is cellular isolation by market."*

### What & why

A **market** is a unit of business+geographic isolation: e.g., US, EU, India, Brazil, Japan, China. A **cell** is a self-contained, independently-deployable, per-market replica of the entire stack: its own Cassandra cluster, its own Redis, its own Fanout workers, its own Kafka, its own Feed Service deployment, its own ranking models even.

Drivers (in priority order at FAANG scale):

1. **Compliance / data residency** — GDPR (EU), India's DPDP Act, China's PIPL, Brazil's LGPD all require user data to stay in-region. Hard requirement, not negotiable.
2. **Blast radius reduction** — a bad config push, a corrupted shard, or a runaway consumer in the EU cell does not take down US users. Cell-level outages are 10–20% of users, not 100%.
3. **Latency** — EU users hit the EU cell, served from EU edge. Saves the trans-Atlantic 80–150ms RTT on cache misses.
4. **Operational independence** — each cell is deployed, monitored, and scaled separately. Markets with different growth curves don't fight for capacity.
5. **Cost attribution** — per-market P&L. PMs want to know what serving Brazil costs vs Japan.
6. **Feature differentiation** — China cell may have different features (mandatory real-name, content filters); India cell may have different ranking models for low-bandwidth.

### Architecture

```
        ┌─────────────────────── Global Edge ───────────────────────┐
        │  CDN + Anycast IP + GeoDNS routes user to nearest cell    │
        └────────┬─────────────────┬──────────────────┬─────────────┘
                 │                 │                  │
       ┌─────────▼────────┐ ┌─────▼────────┐ ┌──────▼────────┐
       │   US Cell        │ │   EU Cell    │ │  IN Cell      │
       │ ┌──────────────┐ │ │ ┌──────────┐ │ │ ┌──────────┐  │
       │ │ Feed Svc     │ │ │ │Feed Svc  │ │ │ │Feed Svc  │  │
       │ │ Post Svc     │ │ │ │Post Svc  │ │ │ │Post Svc  │  │
       │ │ Fanout Svc   │ │ │ │Fanout    │ │ │ │Fanout    │  │
       │ │ Cassandra    │ │ │ │Cassandra │ │ │ │Cassandra │  │
       │ │ Redis        │ │ │ │Redis     │ │ │ │Redis     │  │
       │ │ Kafka        │ │ │ │Kafka     │ │ │ │Kafka     │  │
       │ │ Ranking      │ │ │ │Ranking   │ │ │ │Ranking   │  │
       │ └──────────────┘ │ │ └──────────┘ │ │ └──────────┘  │
       └─────────┬────────┘ └─────┬────────┘ └──────┬────────┘
                 │                │                  │
                 └────────────────┴──────────────────┘
                              │
                  ┌───────────▼───────────┐
                  │  Cross-Cell Bridge    │
                  │  - User → home cell   │
                  │    directory          │
                  │  - Cross-cell follow  │
                  │    & post replication │
                  │  - Read-thru proxy    │
                  └───────────────────────┘
```

### Routing: how a request lands in the right cell

- Each user has a **home cell** assigned at signup (based on country at signup; updated on opt-in migration).
- A small global **User Directory** (replicated KV) maps `user_id → home_cell`. Cached aggressively at the edge.
- API Gateway looks up home cell, routes the request. Mis-routed requests (e.g., user travels) get redirected to home cell, NOT served locally with stale data.
- A **stateless cell** (e.g., a CDN-cached read of public posts) can serve from any cell.

### The hard part: cross-cell interactions

The interesting design question is **what happens when User A (US cell) follows User B (EU cell)?**

Three approaches:

| Approach | How | Pros | Cons |
|---|---|---|---|
| **Full replication** | Every post replicated to every cell; full storage in each | Reads are always local; simplest read-path | Massive storage duplication; violates data-residency for sensitive markets (EU post in CN cell? not allowed) |
| **Selective cross-cell replication** | Only replicate posts that have at least one follower in the other cell | Reduces duplication; respects residency by exclusion lists | Complex follower-set tracking; replication lag affects feed freshness |
| **Cross-cell pull (federation)** | A's feed-read calls EU cell's API for B's recent posts | No replication; clean residency story | Read amplification; cross-region RTT in feed path; failure-coupling between cells |

**My pick for News Feed:** **Selective cross-cell replication** — because:
- Most users follow within their own market (>80% in practice). Cross-cell interactions are the long tail.
- For non-sensitive content, replicate. For residency-restricted markets (CN, EU sensitive), use cross-cell pull and accept the latency.
- This is a **per-market policy**, not a global decision.

### Cell tradeoffs vs single global system

| Dimension | Global | Cellular | Net |
|---|---|---|---|
| Total infra cost | Lower (no duplication) | Higher (replicated stacks) | +20–40% cost for cells |
| Engineering complexity | Lower | Higher (cell awareness everywhere) | Significant tax — pays back at scale |
| Compliance | Hard | Easier | Cellular wins, often non-negotiable |
| Blast radius | 100% on regional outage | 10–20% per cell | Cellular wins |
| Latency for non-cross-cell reads | Higher (wider replication) | Lower (local) | Cellular wins for ≥80% of traffic |
| Cross-cell consistency | N/A | New problem to solve | Cellular pays a tax |

### What changes in the design

- **User Directory** — new global service, replicated KV, sub-millisecond reads at the edge.
- **Cross-Cell Bridge** — service that handles cross-cell follow events and post replication. Owns SLAs for cross-cell propagation.
- **Per-cell Kafka topics** — not one global. Cross-cell events go through Bridge.
- **Per-cell ML models** — each market may have its own ranking model, trained on local data.
- **Per-cell rollouts** — feature flags now have a `cell` dimension. We can roll a feature out to IN first, observe, then EU/US.
- **Failover** — if EU cell goes down: do we route EU users to US cell? Probably not for residency reasons. So the cell IS the failure domain; intra-cell HA is what protects users.

### Operational implications

- **Deploy pipelines** per cell. CI/CD must understand cells.
- **On-call** per cell, or per-team-spanning-cells. Pages route to the cell-aware oncall.
- **Capacity planning** per cell. India growth ≠ US growth.
- **Cost dashboards** per cell — every team has a $/DAU per cell.
- **Schema migrations** per cell — staggered, with cell-N stable while cell-N+1 migrates.

### When NOT to do cellular

- Pre-scale: under ~50M users globally, the operational tax exceeds the benefit.
- No regulatory pressure.
- A single team owns the system and can't operate N cells.
- The product is fundamentally cross-market (e.g., a global marketplace where every user interacts with every other user — replication cost dominates).

### What "Staff signal" sounds like here

> "I'd structure this as a cellular architecture from day one if data residency is a hard requirement, otherwise add it once we cross 50M DAU. The cell IS the unit of blast-radius isolation, deploy, and capacity. The hard part isn't the cells themselves — it's the cross-cell interactions, and I'd default to selective replication with per-market policy. The tax is real — 20–40% more infra cost and a permanent engineering complexity surcharge — but it's the only design that survives an EU regulator audit and a regional AWS outage at the same time."
