# 04 — Design a Video Streaming Platform (YouTube / Netflix)

> The canonical "huge bytes, huge fanout, ML-flavored" HLD problem. Interviewer cares about whether you (a) recognize that **CDN egress is the dominant cost and the dominant operational concern**, (b) can decompose the upload → transcode → deliver pipeline into independent failure domains, and (c) treat ABR (adaptive bitrate) and watch-progress as first-class subsystems instead of footnotes. The candidate who walks in trying to "design YouTube end-to-end" loses. The candidate who picks UGC-scale, scopes ruthlessly, and spends 20 minutes on the transcoding farm + CDN tier is the one who passes Staff.

---

## 0. Problem statement (as the interviewer would drop it)

> **Interviewer:** "Design a video streaming platform — think YouTube or Netflix. Users can upload videos (or for the Netflix variant, the catalog is curated), the system transcodes them, stores them, and serves them to viewers worldwide with adaptive bitrate playback. Walk me through how you'd build it."

That's it. Your first job is to pick **which** product — UGC at YouTube scale or curated at Netflix scale — because the design surface differs.

---

## 1. Clarifying questions

> **Candidate:** "Before I design, two big forks. First — is this YouTube-style UGC, where uploads are continuous and the long tail of content dwarfs the head? Or Netflix-style, where the catalog is curated, smaller, and the design is dominated by delivery economics? They're different problems."
>
> **Interviewer:** "Let's go with YouTube — UGC."
>
> **Candidate:** "Good — that's the harder problem because of the upload+transcode pipeline. A few more questions:"
>
> 1. **"What's the rough scale — DAU, hours uploaded per minute, hours watched per day?"** — *Expect: 2B MAU, 1B DAU, 500 hours uploaded/min, 1B hours watched/day.*
> 2. **"Are we doing live streaming or strictly VOD (video on demand)?"** — *Live is a separate problem; scope out. Assume VOD.*
> 3. **"What's the upload size envelope? Mobile clips up to 4K hour-long?"** — *Drives chunked/resumable upload + transcode SLA. Assume up to ~50 GB per upload, multi-hour 4K possible.*
> 4. **"What playback formats and devices? Web, iOS, Android, smart TVs, Chromecast?"** — *Drives codec mix (H.264 universal, H.265 newer, AV1 cutting-edge, VP9 web).*
> 5. **"What's the latency target for time-to-first-frame (startup)?"** — *Industry: < 2 sec startup, < 1% rebuffer ratio. The "gold metric."*
> 6. **"Geographic scope and content licensing? Geo-fencing required?"** — *Less critical for UGC, but YouTube has region-blocks. Assume yes, per-region rights enforcement is a hard requirement.*
>
> **Out of scope (state explicitly):**
> - Live streaming (WebRTC / HLS-LL is its own problem)
> - Comments, likes, subscriptions UI (mention as integrations, don't design)
> - Monetization / ads insertion (note SSAI/CSAI as a hook, don't design)
> - Recommendations / ranking model internals (treat as black-box service)
> - Authentication, account management
> - Search ranking quality (Elasticsearch is a black box; we surface the API)
> - Copyright fingerprinting (Content ID — separate problem, mention only)
>
> **Candidate:** "I'll design the **upload → transcode → store → deliver → playback → resume** end-to-end pipeline, with ranking and search as service boundaries. The two deep dives I want to land on are (a) the transcoding farm and (b) the CDN tiering with ABR. Sound right?"
>
> **Interviewer:** "Yes."

---

## 2. Functional requirements

| Priority | Requirement |
|---|---|
| P0 | User uploads a video file (chunked, resumable, multi-GB) |
| P0 | System transcodes uploaded video into multiple bitrates + codecs + segments |
| P0 | User can play any video by ID; player adaptively selects bitrate based on bandwidth |
| P0 | Watch progress is durably stored and synced across devices (resume where you left off) |
| P0 | Geo-fencing: per-region content availability enforced |
| P1 | Search by title/description/tags (basic Elasticsearch surface) |
| P1 | Recommendations homepage / "up next" (black-box model returns candidate IDs) |
| P1 | Subtitles / captions (multi-language) |
| P2 | Thumbnails generated automatically (multiple per video) |
| P2 | Download for offline (mobile) |
| Out | Live streaming, ads, comments, monetization, content fingerprinting, DRM key infra design (mention only) |

---

## 3. Non-functional requirements

| Dimension | Target | Justification |
|---|---|---|
| Availability (playback) | 99.99% (~52 min/year) | Front door. Watch path failure is the most user-visible event possible |
| Availability (upload) | 99.9% | Tolerable to retry; users will re-attempt |
| Availability (transcode) | 99.5% (eventual) | Job can be retried; latency from upload to "watchable" is the SLA |
| Startup latency (TTFF) | p50 < 1 sec, p99 < 2 sec | Below 2 sec or users abandon (industry research: every 1s = ~6% drop) |
| Rebuffer ratio | < 0.5% of playback time | The single most-watched quality metric in this domain |
| Upload acceptance latency | p99 < 1 min for the **acknowledgement**; transcode SLA separate | "Your video is uploaded; processing starts now" |
| Transcode SLA | 90% of videos available in 720p within 10 min, 100% in 1 hour | Lazy higher-res tiers acceptable |
| Watch progress write SLA | p99 < 200ms; durability ~5-min RPO acceptable | If we lose 5 min of resume-position on a crash, users tolerate it |
| Durability (raw uploads) | 11 nines (S3-class) | Losing a user's upload is unacceptable; the original is irreplaceable |
| Consistency | Eventual everywhere except billing (N/A here) and watch-progress-self-read | A user must see their own resume position; one user not seeing another's view count immediately is fine |
| Geography | Global; regional CDN cache fill; metadata multi-region active-active | Latency-bound for playback worldwide |

---

## 4. Capacity estimation

> **Candidate:** "Let me run the numbers — they drive every architectural decision in this problem."

### Users
- 2B MAU
- 1B DAU
- Avg watch time per DAU: ~60 min/day → **1B hours watched/day**
- Avg upload rate: **500 hours uploaded per minute** (YouTube public number)

### Uploads
- 500 hr/min × 60 = **30,000 hours of video per hour** uploaded
- Per day: **720,000 hours uploaded**
- Per second: 500 hr/min ÷ 60 = **~8.3 hours/sec** of new content
- Avg file size at upload: assume 1 hr = ~2 GB (mobile-recorded mix of 480p–1080p), so ~17 GB/sec or **~1.5 PB/day** of raw uploads
- After transcode fanout (multiple renditions of each), output is ~3–5× the input size: **~5–7 PB/day of transcoded segments**
- **Per year storage growth (raw + transcoded): ~3 EB/year**. (Yes, exabytes. This is why YouTube is a hyperscale storage problem.)

### Watches
- 1B hours/day watched. Avg bitrate served: assume **3 Mbps** (mix of 480p and 720p, weighted by mobile/desktop split)
- 1B hr × 3,600 sec/hr × 3 Mbps = **10.8 × 10¹⁵ bits/day = ~1.35 PB/day egress**
  - Wait — that's only 1.35 PB. Real YouTube does ~100 PB+/day of CDN egress. Let me redo with a better avg bitrate.
  - Use 5 Mbps average (HD-weighted, since modern viewing skews higher): 1B × 3,600 × 5 Mbps × 1/8 bytes/bit ≈ **2.25 PB/day**.
  - Still low — published estimates land around **100 PB/day** for YouTube. The discrepancy is concurrency assumptions; let me use the published number directly. **~100 PB/day egress** is the working number.
- **Per second egress (peak): ~10–20 Tbps globally**. This is delivered by CDN, not origin.
- **This is the punchline:** CDN egress is the dominant cost line item, full stop. Everything else is rounding error.

### Reads (metadata + manifest)
- A "playback start" hits manifest service once per video. Subsequent segment requests go to CDN.
- Manifest QPS: 1B hours/day ÷ 30 min avg session = ~50M sessions/day = **~600 manifest QPS avg, ~3K peak**.
- That's tiny compared to feed-style systems. **Manifest service is not the bottleneck.**

### Watch-progress writes
- Active concurrent viewers: assume 100M peak concurrent.
- Each player checkpoints every 30s.
- Writes/sec = 100M / 30 = **~3.3M writes/sec at peak**.
- Per write: 100 bytes (user_id, video_id, position_ms, last_updated). 330 MB/sec write volume.
- **This is a major write-heavy table.** Drives Cassandra/Dynamo choice.

### Transcoding compute
- 8.3 hours/sec of new video × ~5 renditions × ~2× realtime CPU per rendition = **~80 CPU-hours/sec of encode load**, i.e., **~80 × 3600 = 288K CPU-hours/hour**, i.e., **~7M CPU-hours/day**.
- At 32 vCPUs/host, that's ~10K transcode hosts steady-state. **Real number is much higher with GPU mix and codec spread.**
- **Per-asset transcoding cost: ~$1–5** (industry public number).
- 720K hours/day × $3 = **~$2M/day transcoding compute**. Significant but a fraction of egress.

### The rough cost stack
| Cost line | Order of magnitude | Lever |
|---|---|---|
| **CDN egress** | $billions/year (dominant) | Multi-CDN mix, owned PoPs (YouTube GGC / Netflix Open Connect), better codecs (AV1 saves ~30%) |
| Storage (raw + transcoded) | $hundreds of millions | Tiered storage: hot SSD → warm HDD → cold archive, popularity-driven |
| Transcoding compute | $hundreds of millions | GPU encoders for popular codecs, lazy higher-res, per-popularity transcode tier |
| Metadata/DB | $tens of millions | Cassandra/DynamoDB, regional |
| Compute (services) | $tens of millions | Trivial |

> **State this loudly:** "CDN egress is the dominant cost. Everything I design will be oriented around minimizing it or extracting maximum efficiency per byte served."

---

## 5. API design

> **Candidate:** "Three surfaces — upload, playback, watch-progress. I'll keep them minimal."

### Upload (chunked, resumable — TUS protocol or equivalent)

```
POST /v1/uploads
  body: { filename, total_size, mime_type, metadata{title, description, tags[], visibility} }
  → 201 { upload_id, upload_url, chunk_size_bytes }

PATCH /v1/uploads/{upload_id}
  headers: Upload-Offset, Content-Length, Tus-Resumable
  body: <binary chunk>
  → 204 (Upload-Offset: <new offset>)

HEAD /v1/uploads/{upload_id}     # client checks where to resume from
  → 200 (Upload-Offset, Upload-Length)

POST /v1/uploads/{upload_id}/complete
  → 202 { video_id, status: "queued_for_transcode" }
```

- **Why TUS-style (not single multipart POST)**: multi-GB uploads on flaky mobile networks. Resume on packet loss is required, not optional.
- The actual bytes go directly to **S3 multipart upload** via a signed URL — the upload service brokers metadata only. We do **not** stream multi-GB through the API service; that would be insane.

### Playback (manifest + segment fetch)

```
GET /v1/videos/{video_id}/manifest?device={web|ios|android|tv}
  → 200 application/dash+xml  (DASH manifest)
   or application/vnd.apple.mpegurl  (HLS m3u8)

# Manifest contains URLs of segments, served by CDN, not origin.
# Each segment is 2–6 sec of video.

GET /v1/videos/{video_id}/license  (DRM, optional)
  → 200 { license_token, drm_key_id }

# Segments themselves served via CDN-direct URLs in the manifest:
GET https://cdn.example.com/v/{video_id}/{rendition}/{segment_n}.ts
```

- Manifest is the **only** call that hits our origin per playback session. After that, it's CDN-only segment fetches.
- Manifest is tiny (~2–10 KB) and cacheable for ~30s on the CDN edge for popular videos.

### Watch progress

```
PUT /v1/watch_progress
  body: { video_id, position_ms, duration_ms, device_id }
  → 204

GET /v1/watch_progress?video_ids=[...]
  → 200 { progress: [{video_id, position_ms, last_updated}] }
```

- PUT is fire-and-forget from the player perspective (best-effort, async). A failed PUT is logged but doesn't block playback.
- GET batches up to ~50 video IDs (used by "continue watching" rail).

### Search & metadata

```
GET /v1/search?q=&filters={...}&page=
  → 200 { results: [{video_id, title, thumbnail_url, channel_id, duration, view_count}], next_cursor }

GET /v1/videos/{video_id}
  → 200 { video_id, title, description, channel_id, duration, view_count, ... }

GET /v1/recommendations?context={home|next|sidebar}
  → 200 { items: [{video_id, reason}] }
```

### Anti-patterns to avoid

- **Don't stream upload bytes through your API tier.** Sign a URL and hand off to S3. Your API tier should never see the bytes.
- **Don't return absolute segment URLs that point at origin.** They must point at CDN edges.
- **Don't make manifest a database call.** Manifest is a static blob per (video, rendition_set) — generate at transcode-completion time, store as an object.
- **Don't bake bitrate selection logic into the server.** ABR is a *client* decision based on local bandwidth + buffer state. Server provides options.

---

## 6. Data model

### Tables / collections

#### `videos` (metadata) — DynamoDB or Cassandra, sharded by `video_id`
| Column | Type | Notes |
|---|---|---|
| video_id | string (Snowflake or UUIDv7) | PK |
| owner_id | string | secondary index |
| title | string | |
| description | string | |
| duration_ms | int | |
| upload_status | enum {uploading, uploaded, transcoding, ready, failed} | |
| created_at | timestamp | |
| visibility | enum {public, unlisted, private} | |
| geo_block | list<region_code> | regions where this video is unavailable |
| view_count | int (denormalized, eventually consistent) | |
| thumbnail_urls | list<string> | |
| renditions | list<{codec, bitrate, resolution, manifest_url}> | populated post-transcode |

#### `upload_sessions` (transient) — Redis or DynamoDB, TTL 7 days
| Column | Type | |
|---|---|---|
| upload_id | string | PK |
| user_id | string | |
| video_id | string | the future video_id, allocated at upload start |
| total_size | bigint | |
| received_bytes | bigint | |
| s3_upload_id | string | S3 multipart upload handle |
| s3_parts | list<{part_number, etag}> | |
| status | enum | |

#### `transcode_jobs` (Cassandra / DynamoDB, sharded by `video_id`)
| Column | Type | |
|---|---|---|
| video_id | string | PK |
| job_id | string | |
| input_s3_path | string | |
| profile | enum {fast_pass, hd_pass, premium_pass} | which set of renditions |
| status | enum {queued, in_progress, complete, failed} | |
| renditions_done | list<{codec, bitrate, resolution, segment_count, manifest_path}> | |
| attempts | int | for idempotent retry |
| started_at | timestamp | |
| completed_at | timestamp | |

#### `watch_progress` — Cassandra (keyed by user_id, clustered by video_id)
- Partition key: `user_id`
- Clustering key: `video_id`
- Columns: `position_ms`, `duration_ms`, `last_updated`, `device_id`
- Why Cassandra: 3M writes/sec, simple key access, multi-region replication required, eventual consistency tolerable.
- TTL: ~90 days (resume position auto-expires).
- Hot read pattern: user opens "Continue Watching" rail → `SELECT * FROM watch_progress WHERE user_id = ? LIMIT 50` — perfectly served by partition.

#### `view_count_buffer` (Redis) → `video_stats` (Cassandra, batched)
- Increments first hit Redis (HINCRBY), flushed every 10s to Cassandra.
- Avoids hammering the metadata DB on every play.

#### `search_index` (Elasticsearch)
- Document per video: `{video_id, title, description, tags[], channel, popularity_score, geo_blocks}`.
- Updated async from `videos` table via CDC.

#### `region_catalog` (per-market, derived) — Redis cache backed by `videos` filtered by `geo_block`
- "Show me videos available in IN" — pre-filtered, cached.

### Why these choices

- **DynamoDB or Cassandra for `videos`**: high read volume on hot videos (view counts, metadata), partition-friendly access by `video_id`. Spanner is overkill (no transactional cross-row needs); Postgres doesn't scale to billions of rows × QPS at p99 < 50ms.
- **S3 (or GCS) for raw uploads + transcoded segments**: 11 nines durability, multipart upload, lifecycle policies (move to Glacier when cold). Object storage is the only sane choice for petabytes of media.
- **Cassandra for `watch_progress`**: write-heavy (3M w/s), partition-key access, multi-region active-active.
- **Kafka for transcode pipeline events**: durable, replayable, partition-able by video_id (so retries land on same worker).
- **Redis for hot manifest cache + view-count buffer + upload sessions**: low latency, ephemeral data.
- **Elasticsearch for search**: standard choice, indexes from CDC.

---

## 7. High-level architecture

```
                ┌───────────────────────────────────────────────────────┐
                │   GLOBAL EDGE: GeoDNS + Anycast → nearest PoP         │
                └───────────────────────────────────────────────────────┘
                              │                              │
                       PLAYBACK PATH                    UPLOAD PATH
                              │                              │
        ┌─────────────────────▼───────────┐         ┌────────▼──────────┐
        │       CDN tier 1 (edge POP)     │         │   API Gateway     │
        │   (Akamai / CloudFront / GGC)   │         │ (auth, rate limit)│
        │   - Serves segments hot-cached  │         └────────┬──────────┘
        │   - 90%+ hit rate target        │                  │
        └────────┬────────────────────────┘                  │
                 │ miss                                       ▼
        ┌────────▼─────────────────────────┐         ┌───────────────────┐
        │   CDN tier 2 (regional shield)   │         │   Upload Service  │
        │   - Origin shield, dedup misses  │         │ (TUS broker only) │
        └────────┬─────────────────────────┘         └────┬──────────────┘
                 │ miss                                    │ signs S3 URL,
                 ▼                                         │ tracks session
        ┌──────────────────────────────────┐              ▼
        │       Origin Storage (S3)        │       ┌──────────────┐
        │   (transcoded segments, multi-   │       │   S3 (raw    │
        │    region replicated)            │       │  uploads)    │
        └────────────────────────────────┬─┘       └──────┬───────┘
                                          │               │
                                          │  on transcode │ on complete:
                                          │  output       │ emit event
                                          │               ▼
                                          │   ┌─────────────────────────┐
                                          │   │  Kafka: transcode.queue │
                                          │   └────────┬────────────────┘
                                          │            │
                                          │            ▼
                                          │   ┌─────────────────────────────┐
                                          │   │   Transcoding Dispatcher    │
                                          │   │   (assigns jobs, profiles)  │
                                          │   └────────┬────────────────────┘
                                          │            │
                                          │       fanout per rendition
                                          │            │
                                          │   ┌────────▼────────────────────┐
                                          │   │  Transcoding Workers        │
                                          │   │  (CPU + GPU pools)          │
                                          │   │  - H.264, H.265, AV1, VP9   │
                                          │   │  - HLS + DASH segmenter     │
                                          │   └────────┬────────────────────┘
                                          │            │ writes segments
                                          └────────────┘
                                                       │
                                                       ▼
                              ┌────────────────────────────────────────┐
                              │   Manifest Generator                   │
                              │   (produces HLS m3u8 + DASH mpd)       │
                              │   stores in S3 + warms Redis cache     │
                              └────────────────────────────────────────┘

  PLAYBACK (continued):
    Client → API Gateway → Playback Service → returns manifest URL (CDN-fronted)
                                            → returns DRM license token if needed
    Client → CDN → fetches segments → ABR loop (player chooses bitrate per segment)

  WATCH PROGRESS:
    Player → /v1/watch_progress (every 30s) → Watch-Progress Service
                                            → Cassandra (RF=3, LOCAL_QUORUM)

  RECOMMENDATIONS (black-box):
    Client → API Gateway → Recs Service → returns video_ids
                                        → Playback Service hydrates metadata

  SEARCH:
    Client → API Gateway → Search Service → Elasticsearch
                                          → Hydrate from videos table
```

### Request walk-throughs

**Upload path:**
1. Client `POST /v1/uploads` with metadata. Upload Service allocates `upload_id` and a future `video_id`, initiates an S3 multipart upload, returns a signed URL pattern.
2. Client `PATCH`es chunks. Each chunk goes directly to S3; Upload Service merely records part numbers + ETags in `upload_sessions`.
3. Client `POST /v1/uploads/{id}/complete` → Upload Service finalizes the S3 multipart upload, marks the video record `upload_status = uploaded`, and emits `transcode.queue` event to Kafka with `video_id` and S3 path.
4. Total bytes through our service: zero (just metadata). All bytes go S3-direct via signed URLs.

**Transcode path:**
1. Transcoding Dispatcher consumes from Kafka. Looks up the **transcode profile** — fast pass (480p+720p, H.264 only) or HD pass (full ladder + multiple codecs).
2. Dispatcher fans out per-rendition jobs to a second Kafka topic, partitioned by `video_id`.
3. Transcoding Workers (CPU pool for H.264, GPU pool for H.265/AV1) pick up jobs:
   - Download source from S3.
   - Encode to target rendition (specific bitrate, resolution, codec).
   - Segment into 4–6 second chunks (HLS .ts or DASH .m4s).
   - Upload segments to S3 + populate manifest fragments.
   - Mark rendition complete in `transcode_jobs`.
4. When the **fast-pass renditions** are complete (typically 480p + 720p in H.264), Manifest Generator runs and the video is marked `ready`. Higher-res / additional-codec renditions complete later (lazy). The manifest is updated as renditions arrive.
5. SLA target: 90% of videos have `ready` status within 10 min of upload-complete.

**Playback path:**
1. Client `GET /v1/videos/{video_id}/manifest`. Playback Service:
   - Checks `geo_block` against viewer's region (rejects if blocked).
   - Returns CDN-fronted manifest URL.
2. Client fetches manifest from CDN edge (cached). Manifest lists segment URLs by rendition.
3. Client begins ABR loop:
   - Probes initial bandwidth with a low-bitrate segment.
   - Maintains a buffer of 30–60 sec ahead.
   - Per-segment, selects rendition based on (bandwidth_estimate, buffer_health, screen_size).
4. Each segment fetched via CDN. 90%+ hit at edge. Misses go to regional shield, then origin S3.
5. Client periodically posts watch-progress.

**Watch progress path:**
1. Player POSTs every 30 sec.
2. Watch-Progress Service writes to Cassandra (LOCAL_QUORUM). p99 < 200ms.
3. On player startup, GET `/v1/watch_progress?video_ids=[recent]` to hydrate "continue watching" rail.

### Why this layered architecture

- **Separation of upload + transcode + delivery** lets each scale independently. Upload is bursty (per-user); transcode is queue-shaped; delivery is steady-state.
- **CDN is a hard layer** — origin must never serve user traffic at scale. Origin is for cache-fill only.
- **Async transcoding** with priority profiles lets us deliver "watchable in 10 min" SLA without forcing all renditions to complete.
- **Manifest as static object** means manifest fetch hits CDN, not origin. The path from "user clicks play" → "first byte" is **all edge-served**.

---

## 8. Deep dives

### Deep dive 1: Transcoding farm

> **Interviewer:** "Walk me through the transcoding farm. How many machines, how do you handle failures, what's the codec/bitrate strategy?"

**Candidate:**

The transcoding farm is the **highest-OPEX compute system** in this design — and it's where the biggest engineering wins come from.

**The job decomposition.** A single uploaded video produces a matrix of outputs:

| Resolution | Codec | Container | Notes |
|---|---|---|---|
| 240p | H.264 | HLS + DASH | Mobile fallback; lowest priority |
| 480p | H.264 | HLS + DASH | **Fast-pass** — first delivered |
| 720p | H.264 | HLS + DASH | **Fast-pass** — first delivered |
| 1080p | H.264 + H.265 + AV1 | HLS + DASH | Full pass |
| 1440p | H.265 + AV1 | DASH | Full pass, popular videos only |
| 4K | H.265 + AV1 | DASH | Full pass, premium / popular only |

So one video → ~12–18 renditions × 2 containers = ~30 outputs in worst case.

**The pipeline:**
```
posts.uploaded event
       │
       ▼
TranscodeDispatcher
   - Looks up upload metadata
   - Picks profile (fast/full/premium) based on:
       * uploader's tier (premium creators get full pass)
       * predicted popularity (channel size, recent watch heatmap)
       * cost budget for this asset
   - Emits N rendition-jobs to Kafka
       │
       ▼
fanout-topic.transcode.rendition (partitioned by video_id)
       │
       ▼
TranscodeWorker pool
   ┌─ CPU pool (H.264 — universal, fast on CPU)
   ├─ GPU pool (H.265 — GPU encoders save 5–10× wallclock)
   └─ Specialized AV1 pool (AV1 is slow; SVT-AV1 + GPUs)
       │
       ▼
   Per-job:
   1. Download source from S3 (streaming)
   2. ffmpeg / svt-encoder → segments
   3. Upload segments to S3 (regional replication)
   4. Update transcode_jobs with rendition status
   5. Emit "rendition.complete" event
       │
       ▼
ManifestGenerator (consumes rendition.complete)
   - When fast-pass renditions complete → mark video "ready"
   - Continually update manifest as more renditions arrive
   - Push warm manifest to CDN
```

**Idempotency.** Workers crash mid-job. Critical: every job has a deterministic output S3 key. On retry, workers check for existing output (`HEAD` S3) before re-encoding. If a partial output exists from a prior crash, we delete and restart. Job state lives in `transcode_jobs.attempts`, capped at 5 retries → DLQ.

**Sharding the pipeline.** Kafka topic partitioned by `video_id`. Single video's jobs land on the same consumer group instance — useful if we want to coordinate (e.g., emit "all renditions complete" event). For maximum parallelism, we **fan out within a video** — each rendition is its own message.

**Profile selection** — the cost lever:

| Profile | Renditions | Cost | When |
|---|---|---|---|
| **Fast pass** | 480p + 720p, H.264 only | $0.30 | Long-tail UGC, low-traffic creators |
| **HD pass** | + 1080p, H.265 | $1.50 | Avg creator |
| **Full pass** | + 4K, AV1 | $5.00 | Premium creators, predicted-popular videos |
| **Re-pass** | Add AV1 to existing video | $1.00 | When AV1 becomes worth it (popularity crossing threshold) |

**Lazy higher-res transcoding.** Most UGC is never watched in 4K. Don't transcode 4K eagerly — do it on first 1080p+ playback request, or after the video crosses N views/hour. Saves ~70% of transcode cost on the long tail.

**GPU vs CPU.** H.264 encodes ~2–5× realtime on a modern CPU. H.265 is 3× slower. AV1 is ~10× slower than H.264. On GPU encoders (NVENC, Quick Sync, Apple Silicon), H.264 hits ~30× realtime, H.265 ~10×, AV1 still slow but improving. **Mixed pool** — route H.264 to CPU (cheaper), H.265 to GPU (faster wallclock per dollar), AV1 to specialized SVT-AV1 hosts.

**Codec mix tradeoff:**

| Codec | Pros | Cons | Strategy |
|---|---|---|---|
| **H.264** | Universal device support; mature | Largest files at given quality | Always produce; baseline |
| **H.265 (HEVC)** | 30–50% smaller than H.264 | Patent fees; not universally supported (web-mixed) | Produce for capable clients (mobile, TV) |
| **AV1** | 30% smaller than H.265, **royalty-free** | Slow encode; client decode support uneven (improving fast) | Produce for popular content; massive long-term egress savings |
| **VP9** | Royalty-free, web-friendly | Being superseded by AV1 | Legacy, phasing out |

**The cost-of-encode vs cost-of-egress tradeoff** is the core economic decision. AV1 takes ~$3 more to produce per video but saves ~30% on egress. Break-even: video must accumulate ~3 GB of egress savings to pay for itself = ~3,000 views at 1 MB/min × 1 hour. Long-tail UGC never crosses this; popular videos cross it 10⁶ ×. Therefore: **produce AV1 lazily, only after a popularity threshold.**

> **Interviewer:** "What if a transcoder crashes mid-job?"

**Candidate:** Idempotent retry. The job message stays in Kafka; the worker doesn't ack until it commits both the S3 output and the `transcode_jobs` row. On crash, another worker picks up. Output keys are deterministic (`s3://video-segments/{video_id}/{rendition}/{segment_n}.ts`), so we either find existing output and skip, or delete partial output and re-encode. We cap attempts at 5 and DLQ. DLQ is reviewed daily; common causes are corrupted source files (we mark the video `failed` and notify the user) or worker bugs (we re-process after fix).

> **Interviewer:** "How do you canary a new transcoder version?"

**Candidate:** Shadow encode. Take 1% of incoming jobs, encode with both old and new versions, compare PSNR/SSIM/VMAF (perceptual quality metrics) and file size. If new version regresses quality > 1 VMAF point on > 5% of jobs, auto-rollback. If size increases > 5% with no quality gain, also rollback. Promote gradually 1% → 10% → 100% over a week. **Never** deploy a new encoder fleet-wide without shadow validation — a regression here costs millions in storage + egress.

### Deep dive 2: CDN tiering and ABR delivery

> **Interviewer:** "Talk me through the CDN strategy and how the player adapts bitrate."

**Candidate:**

This is **the** problem in this domain. CDN egress is the dominant cost; rebuffer ratio is the dominant quality metric. Both are determined here.

**CDN architecture — three tiers:**

```
Client (player)
    │
    ▼
Tier 1: Edge POP (closest to user, ~10ms RTT)
   - 1000s of POPs globally
   - Akamai / CloudFront / Fastly + owned PoPs (YouTube GGC, Netflix Open Connect)
   - Hit rate target: 90%+ for popular content
    │ on miss
    ▼
Tier 2: Regional shield (per-region, ~50ms)
   - 10s of shields globally (one per major region)
   - Dedup misses across edge POPs to a single origin fetch
   - Hit rate target: 99% (catches everything but absolute cold content)
    │ on miss
    ▼
Tier 3: Origin (S3, multi-region replicated)
   - Final fallback
   - Origin egress goal: < 1% of total traffic (i.e., 99% caught upstream)
```

**Why three tiers?** Without the regional shield, every cold-content request from every edge POP would hit origin. With 1000 POPs and a viral video request from 100 different POPs simultaneously, that's 100 origin requests for the same segment. The shield deduplicates: 100 edge POPs → 1 shield → 1 origin fetch. **Origin shielding is the single most important cost lever after multi-CDN.**

**Multi-CDN strategy:**

| Provider | Use case | Why |
|---|---|---|
| Akamai | Premium markets, high-availability backbone | Most reliable, expensive |
| CloudFront | Bulk egress, AWS-integrated | Cheapest, AWS lock-in concerns |
| Fastly | Edge compute (image manipulation, manifest mods) | Programmable edge |
| **Owned PoPs (GGC / Open Connect)** | Hot content cached at ISPs directly | Eliminates transit cost; ISPs love it (saves them peering bandwidth) |
| Cloudflare | Tertiary fallback | Wide footprint |

**Active monitoring + auto-shift.** Each CDN reports per-region p99 latency and error rate. A real-time orchestrator shifts traffic if a CDN's p99 spikes or error rate > 0.5%. Done via DNS TTL (60s) and Anycast.

**Owned PoPs (YouTube GGC, Netflix Open Connect)** are the killer feature at scale. We embed appliances inside ISP networks. ISPs love it because they save on peering costs; we love it because the bytes never traverse the public internet on the way to the user. **At Netflix scale, Open Connect serves > 90% of traffic from ISP-embedded boxes.** Major engineering effort but the cost savings dwarf the build cost above ~50M streaming hours/day.

**Cache fill strategy — push vs pull:**

| Strategy | How | When |
|---|---|---|
| **Pull-through (default)** | First request to a POP fetches from upstream | Long-tail content; unpredictable demand |
| **Push (warming)** | Pre-populate POPs based on expected demand | New popular releases (Netflix), trending videos, regional premieres |
| **Per-region warming** | Push popular-in-region content to that region's POPs | Geo-diverse content (e.g., Indian Premier League content pushed to IN POPs) |

For Netflix-style: when a new season drops, push to all POPs an hour before release. For YouTube-style: pull-through is the default; only the global trending top-1K gets push-warming.

**Adaptive bitrate (ABR) — client-side:**

The player runs an ABR algorithm. The most common in production: **BOLA** (Buffer Occupancy-based Lyapunov Algorithm) or **MPC** (Model Predictive Control). Simplified logic:

```
on_segment_complete():
    measured_bandwidth = bytes_received / time_taken
    buffer_seconds = current_buffer_ahead

    if buffer_seconds < 5:      # danger zone
        bitrate = lowest_safe_bitrate
    elif buffer_seconds < 15:   # caution
        bitrate = max_bitrate where bitrate < 0.8 * measured_bandwidth
    else:                        # comfortable
        bitrate = max_bitrate where bitrate < 1.2 * measured_bandwidth
```

The server provides the **rendition ladder** in the manifest; the client picks. Server doesn't know client bandwidth — only the client measures the path.

**Server-side hints (newer):** The server can hint at "expected bandwidth for this user" based on historical data and time-of-day, sent via HTTP header. The client uses this to pick its initial bitrate (avoiding the cold-start "always start at 240p" problem). YouTube does this; it cut TTFF by ~200ms.

**ABR cap on inactive tab.** A backgrounded browser tab still requests segments. Cap to lowest bitrate when tab is inactive. Saves 30%+ egress on background-playing tabs (a real phenomenon).

**Segment length tradeoff:**

| Segment length | Pros | Cons |
|---|---|---|
| 2 sec | Fast bitrate adaptation, low TTFF | More HTTP overhead, more manifest churn |
| 6 sec | Better encoding efficiency, fewer requests | Slower adaptation, higher TTFF |
| 10 sec (DASH) | Most efficient encoding | Sluggish adaptation; only used for non-live |

**My pick:** 4-second segments. Standard industry choice; balances adaptivity and efficiency.

> **Interviewer:** "What happens when a video goes viral and the CDN gets overwhelmed?"

**Candidate:** This is the **origin shield miss storm** scenario. Viral video → 100M concurrent viewers → all hit edge POPs simultaneously → first-time POPs miss, all hit regional shield, shield possibly misses to origin. Three mitigations:

1. **Origin shield with request coalescing** — shield deduplicates concurrent identical requests. If 1000 edge POPs simultaneously request segment 5, shield does 1 origin fetch and fans out. Standard CDN feature; verify enabled.
2. **Pre-warming on trend detection** — anomaly detector watches per-video QPS gradient. When a video trends (say, 10× growth in 5 min), trigger pre-warming push to all POPs.
3. **Tiered origin** — origin S3 has a Read-Through Redis cache for hot segments. Even if shield fully misses, origin doesn't go to disk for the same segment 1000× — it serves from RAM after the first fetch.
4. **Bitrate cap on viral content** — under extreme load, the manifest service can return a manifest with the 4K rendition stripped, capping users at 1080p. Saves 50% of egress without users noticing on mobile.

> **Interviewer:** "If a CDN provider has a regional outage, how fast do you fail over?"

**Candidate:** Active monitoring observes p99 + error rate per CDN per region every 10 sec. On 30-sec sustained breach, traffic-shift via DNS (60s TTL) to alternate CDN. Not instant — users mid-stream may see a 30–60 sec rebuffer hiccup. Trade-off: if we make TTL too short (e.g., 10s) we crush our DNS infrastructure with constant lookups; 60s is the industry standard.

### Deep dive 3: Watch progress at scale

> **Interviewer:** "You mentioned watch progress. 3 million writes per second is a lot. Walk me through it."

**Candidate:**

**Workload:** 100M concurrent viewers × 1 write per 30s = ~3.3M writes/sec at peak. Each write is small (~100 bytes). The hot read pattern: "Continue Watching" rail at app open — fetch all in-progress videos for a user.

**Choice: Cassandra (or DynamoDB)** with `(user_id, video_id)` partitioning.

```
CREATE TABLE watch_progress (
    user_id     text,
    video_id    text,
    position_ms bigint,
    duration_ms bigint,
    last_updated timestamp,
    device_id   text,
    PRIMARY KEY (user_id, video_id)
) WITH default_time_to_live = 7776000;  -- 90 days
```

- Partition key `user_id`: per-user partition holds at most ~100 in-progress videos. Tiny partitions.
- Per-user reads: `SELECT * WHERE user_id = ?` returns the rail. Fast.
- Writes: `INSERT INTO watch_progress (...) USING TTL ...` — last-writer-wins, no read-before-write.

**Multi-region write strategy.** A user might play on mobile in NYC and pause; resume on TV in LA. Watch-progress must reach LA by then. Options:

| Option | Mechanism | Lag | Tradeoff |
|---|---|---|---|
| **Active-active Cassandra (multi-region)** | Writes go to local region; replicate async to others | 100ms–1s | Last-writer-wins on conflict (timestamp-based); rare since writes are fast |
| **Single-leader (write home region)** | All writes go to user's home region | RTT to home region | Cross-region latency for traveling users (NYC user in Tokyo: 200ms write) |
| **Edge + sync** | Player buffers writes locally, syncs on next online | 0 from client perspective | Resilient to network blips |

**My pick:** Active-active Cassandra + edge buffering on the player. Last-write-wins resolves conflicts (you'd notice if your phone's stale resume position overwrote your TV's, but it's a 1-in-millions edge case and the resolution is "you see resume position from the most recent device").

**Throttling.** Player checkpoints every 30s — sometimes more aggressive on mobile (every 10s during ad-supported tier so we don't lose progress on app crash). Backpressure: if the watch-progress service returns 503, player buffers and retries. We don't block playback ever.

**Alternative: in-memory write absorber.** Could put a Redis tier in front: writes go to Redis first, async-flushed to Cassandra in batches. Reduces Cassandra write QPS by 100×. Cost: Redis becomes a single point of data loss if it crashes. Acceptable because watch-progress has 5-min RPO tolerance. **I'd add this if the Cassandra write bill becomes a problem.**

### Deep dive 4: Cold-start playback and TTFF optimization

> **Interviewer:** "How do you make playback start fast — under 1 second?"

**Candidate:**

TTFF (time-to-first-frame) is dominated by:

1. **Manifest fetch** (1 RTT from client to nearest CDN POP serving manifest): ~50ms.
2. **First segment fetch** (1 RTT to first segment): ~50ms.
3. **Decoder warmup** (parsing + first frame decode): ~100ms.
4. **DRM license fetch** (if encrypted): another RTT, ~50–100ms.

Naive total: ~200–300ms. Real-world p99 is more like 1–2 seconds because of fat-tail issues:
- DNS resolution on cold app: +100ms
- TLS handshake: +1 RTT (mitigated by TLS 1.3 + 0-RTT for repeat sessions)
- TCP slow-start: first segment download throttled by TCP window
- DRM license server slow: +100ms+

**Optimizations:**
- **Manifest at edge.** Manifest is a small static object — cache aggressively. 30s TTL.
- **Low-latency first segment.** First segment of every rendition is encoded at the lowest bitrate to minimize first-byte download time. Player switches up after segment 2 if bandwidth allows.
- **Pre-fetch on hover** (web). When user hovers a thumbnail for > 500ms, pre-fetch manifest + first segment.
- **DRM license preloading.** On app open, batch-fetch DRM licenses for items in "Continue Watching" so playback is instant.
- **HTTP/3 (QUIC).** 0-RTT connection establishment for repeat playback in the same session.
- **Server-side initial bitrate hint.** Use historical bandwidth for this user/network to pick initial rendition; don't always start at 240p.

**Result:** with all of the above, p50 TTFF drops to ~500ms, p99 to ~1.5 sec.

### Deep dive 5: Popularity tiering and storage cost

> **Interviewer:** "You said storage is 3 EB/year. How do you not go bankrupt?"

**Candidate:**

**Popularity follows a power law.** Empirical YouTube data: top 0.1% of videos account for ~20% of views; long-tail (90% of videos) accounts for ~5% of views.

**Tiered storage:**

| Tier | Storage | Cost ratio | What lives here |
|---|---|---|---|
| **Hot** | NVMe-backed S3 / regional replicas | 1× (most expensive) | Top 1% of videos by 30-day views; new uploads (first 7 days) |
| **Warm** | Standard S3 | 0.5× | Top 10% by views; videos uploaded last 30 days |
| **Cold** | S3 Infrequent Access | 0.1× | Long-tail, > 30 days old, < 1 view/week |
| **Glacier / Deep Archive** | Glacier | 0.01× | Raw uploaded sources after transcoding (kept for re-encode); videos with 0 views in 1 year |

**Movement triggered by popularity model:** every video has a `last_30d_views` metric. Nightly job moves objects between tiers.

**The raw upload question.** We keep the raw uploaded source forever (or until user deletes). Why? Because in 3 years AV2 will be the new codec; we'll want to re-transcode. Raw goes to Glacier Deep Archive — costs ~$1/TB/month. 1 PB raw/day × 365 = 365 PB/yr × $0.001/GB/mo = ~$365K/month. Trivial.

**Don't keep stale low-bitrate renditions.** When AV1 renditions exist for a video and have > 80% of playback share, drop H.264 4K (the AV1 4K is 30% smaller and serves the same role). Saves storage + future egress.

**Geo-replication of hot content only.** Top 10% of videos replicated to all major regions. Long tail in 1 region only; cross-region request fetches once and caches at the destination. Saves replicated storage on the long tail.

---

## 9. Tradeoffs & alternatives

### Decision: Push transcoding (eager) vs Pull transcoding (lazy on first play)

| Option | Pros | Cons | When |
|---|---|---|---|
| **Pure eager** | Predictable latency for first viewer; simple flow | Wastes compute on never-viewed videos (~80% of long-tail UGC) | Curated catalogs (Netflix) |
| **Pure lazy** | No wasted compute | First viewer waits ~1 min for transcoding | Tiny systems |
| **✅ Hybrid: eager fast-pass, lazy higher-res** | Long-tail saved; first viewer always has 480p+720p | Implementation complexity | **YouTube/UGC at scale** |

I'd defend hybrid by saying: "fast-pass costs $0.30/video, full pass costs $5. 80% of UGC never gets > 100 views. Lazy higher-res saves ~70% of transcode spend without any user-visible latency."

### Decision: HLS vs DASH

| | HLS | DASH | Pick |
|---|---|---|---|
| Origin | Apple | MPEG (ISO standard) | |
| Container | .ts (legacy) or fragmented MP4 | fragmented MP4 | |
| Adoption | Apple devices, Safari (native) | Android, web, smart TV | |
| Manifest format | m3u8 (text) | mpd (XML) | |
| **Pick** | Required for Apple | Required for everything else | **Both, served from same segment files (CMAF)** |

**CMAF (Common Media Application Format)** — a unified container that works for both HLS and DASH from a single set of segment files. Standard since ~2018. Halves storage cost vs producing separate segment sets.

### Decision: H.264 + H.265 + AV1, not just H.264

| | H.264 only | + H.265 | + AV1 |
|---|---|---|---|
| Egress cost | Baseline | -30% on H.265-capable clients | -30% more on AV1 clients |
| Transcode cost | Baseline | +50% | +200% |
| Client compatibility | 100% | ~80% (no IE/old Safari) | ~50% (improving fast) |
| **Verdict** | Always produce | Produce for popular content | Produce lazily for highly popular content |

### Decision: Cassandra vs DynamoDB for `videos` and `watch_progress`

| | Cassandra | DynamoDB | Pick |
|---|---|---|---|
| Operational burden | High (we manage) | Low (managed) | DynamoDB if you have ops capacity issues |
| Cost at scale | Lower per GB at PB scale | Higher per RCU/WCU at high QPS | Cassandra at YouTube-scale |
| Multi-cloud | Yes | AWS-only | Cassandra if multi-cloud matters |
| Flexibility | Tunable consistency | Binary consistency | Cassandra |
| **Default pick** | At YouTube/Netflix scale | At < 100M user scale | Cassandra |

I'd defend Cassandra by saying: "We're a YouTube-scale company; per-RCU pricing at 3M writes/sec is prohibitive on Dynamo. We accept the ops cost. If we were a 5-engineer team starting greenfield, Dynamo wins."

### Decision: Kafka vs SQS for transcode pipeline

| | Kafka | SQS | Pick |
|---|---|---|---|
| Replay | Yes | No (or limited via DLQ) | Kafka — we need to re-process when we ship a new encoder |
| Ordering | Per-partition | FIFO variant | Kafka |
| Throughput | Massive (millions/sec) | Plenty (thousands/sec) | Both fine, Kafka has headroom |
| Ops cost | Higher | Zero (managed) | Kafka if you have it; SQS if greenfield AWS |

**My pick:** Kafka. Replay alone justifies it.

### Decision: Multi-CDN + owned PoPs vs single CDN

| | Single CDN | Multi-CDN | Multi-CDN + owned PoPs |
|---|---|---|---|
| Cost | Highest per byte | Mid (negotiating leverage) | Lowest |
| Reliability | Single failure domain | Failover possible | Best |
| Engineering effort | Low | Mid (orchestration) | High (build PoP boxes) |
| Pays back at | Always | ~10M DAU | ~50M DAU |
| **Pick** | — | Default at scale | At YouTube/Netflix scale |

---

## 10. Defending architectural choices — interviewer attacks

> **Interviewer:** "Why not just use a managed transcoding service like AWS Elemental MediaConvert?"

**Candidate:** For Netflix/YouTube scale, MediaConvert is 5–10× more expensive per minute than running our own farm. The break-even is around 1M hours/month of input — below that, MediaConvert wins on engineering simplicity. Above that, we save tens of millions/year by self-hosting. We also need fine-grained control: custom codec ladders, per-channel transcoding profiles, GPU/CPU mix tuning, and shadow encoding for canary releases. MediaConvert can't be canaried per-job. So we self-host the pipeline; we use MediaConvert only for spike capacity if we get behind.

> **Interviewer:** "Why DASH/HLS instead of WebRTC? WebRTC is lower-latency."

**Candidate:** WebRTC is for sub-second latency — good for real-time conferencing or live interactive streaming. VOD has no latency requirement on the playback start beyond TTFF (~1 sec). DASH/HLS over HTTP is **massively** more cacheable (it's just files at URLs); WebRTC is per-connection and uncacheable. CDN economics dictate HTTP-based delivery. If we did live interactive streaming, I'd reach for WebRTC or HLS-LL (low-latency HLS, ~3 sec glass-to-glass). For VOD, HTTP-based ABR wins on cost by 100×.

> **Interviewer:** "Why isn't the manifest dynamically generated per-user?"

**Candidate:** It can be — and for some features (per-user ad insertion, A/B-tested rendition ladders) it has to be. But making the manifest per-user kills CDN cacheability. The default is a static manifest cached at edge; per-user variants are an exception, served from a small per-user-manifest service that's still backed by static segment files. SSAI (server-side ad insertion) does manifest manipulation at edge using programmable CDNs (Fastly Compute@Edge, Cloudflare Workers) — that way we keep most of the cache benefit.

> **Interviewer:** "What if S3 has a partial outage?"

**Candidate:** Tiered failure response:
1. **Single-AZ failure**: S3 transparently routes around it. No user impact.
2. **Single-region S3 outage**: We have multi-region replicas of hot content; CDN keeps serving from cached edges (most users see no impact for the first 30 min). For cold content, we read from the replicated region.
3. **Multi-region outage**: Catastrophic. CDN edge caches keep serving hot content for hours (manifest TTL 30s but segment TTL is days). Uploads fail; users retry. New content can't transcode. Surface "video processing delayed" message. RTO is hours, RPO is minutes (uploads in flight may need to be retried).

> **Interviewer:** "How do you handle a 4-hour-long upload from a slow mobile connection that drops repeatedly?"

**Candidate:** This is exactly why we use TUS-style chunked resumable uploads. The client sends 5–10 MB chunks; each chunk is independently committed to S3 multipart. On disconnect, the client comes back, calls `HEAD /uploads/{id}` to learn where to resume from, and continues. The upload session has a 7-day TTL — they have a week to finish. We've decoupled "upload duration" from "any single TCP connection."

> **Interviewer:** "Why store watch progress in Cassandra instead of, say, just in the player as a cookie?"

**Candidate:** Watch progress must sync across devices — phone → TV, browser → tablet. Local-only loses this. Could put it in a small SQL store, but at 3M writes/sec a single-leader SQL doesn't scale even with sharding. Cassandra naturally partitions by user_id, replicates multi-region, and tolerates LWW conflicts (which we accept here). DynamoDB would also work; Cassandra wins on per-record cost at this volume.

> **Interviewer:** "Why don't you just transcode on-demand at first play? Saves storage."

**Candidate:** Two reasons. (1) First-viewer latency: transcoding 1 hour of 1080p AV1 takes ~10 minutes even on GPU. The first viewer would wait 10 minutes for playback to start — unacceptable. (2) Cache thrash: if one viewer triggers a transcode and 100K others arrive in the next minute, we'd serialize behind the transcode. We do **fast-pass eagerly** (so 480p/720p H.264 are always ready in 10 min) and **lazy higher-res**, accepting that the first 4K viewer may get 1080p served while we kick off the 4K encode in the background.

> **Interviewer:** "What about copyright? Someone uploads a copyrighted movie."

**Candidate:** Out of scope for this design — that's Content ID / fingerprinting, which is its own problem. The hook in this design is: a Content ID service consumes from `posts.uploaded` Kafka topic, runs perceptual fingerprinting on the raw upload, and emits a copyright-match event that updates the video's `visibility` to `restricted` or `blocked`. This runs in parallel with transcoding. If a match is found before transcoding completes, the video is blocked from publish. If a match arrives after publish, we mark and possibly remove. We'd build that as a separate ML pipeline.

---

## 11. Failure modes & mitigations

| Failure | Detection | Blast radius | Mitigation | Recovery |
|---|---|---|---|---|
| Transcoder worker crash mid-job | Job timeout (no progress in 5 min) | One job delayed | Idempotent retry — output keys deterministic, partial outputs purged | Auto-retry up to 5×, then DLQ |
| Transcode queue backlog (viral upload event) | Kafka consumer lag > 10 min | Transcode SLA breach | Auto-scale workers; deprioritize full-pass to fast-pass-only | Drain backlog, investigate root cause |
| S3 upload partial failure | Multipart upload not completed | Specific upload may be lost | Client retries via TUS resume; signed URL still valid | TUS session TTL 7 days |
| CDN edge POP failure | POP health check / latency spike | Users in that POP's region degraded | DNS failover to alternate POP/CDN | DNS TTL 60s; full failover < 2 min |
| CDN regional shield failure | Shield latency / error rate | Origin storm risk | Edge POPs failover to alternate shield; origin has Redis cache to absorb storm | Bring shield back online |
| Origin S3 partial outage | S3 error rate per region | Cold-content reads fail in that region | Multi-region S3; cross-region read on miss | Wait for AWS recovery; we have replicas |
| DRM license server down | License request errors | Encrypted content unplayable | Fallback to client-cached license (24h validity); short outage absorbed | Failover to standby region; on-call paged |
| Playback service down | API gateway 5xx on `/manifest` | New playbacks blocked; existing sessions continue | Multi-region active-active; DNS failover | Auto-rollback if recent deploy |
| Watch-progress service down | Write 5xx | Resume positions not saved during outage | Player buffers locally, syncs on recovery | Service restart; backlog processed; some loss bounded by buffer size |
| Cassandra `watch_progress` shard down | Per-shard latency / errors | 1/N of users have degraded resume | Replicas absorb (RF=3, LOCAL_QUORUM); writes route around | Replace node, stream-rebuild |
| Manifest cache (Redis) lost | Cache hit rate dip | Manifest service load spikes | Manifest is regenerable from S3 (it's a static file there); rebuild cache on miss | Re-warm from S3 reads |
| Encoder regression (new version produces bad output) | VMAF score regression in shadow encode | Could affect quality on canaried videos | Shadow encode + auto-rollback if VMAF drops > 1 point | Roll back to previous encoder; affected videos re-transcoded |
| Catastrophic data corruption (bad config wipes segments for a class of videos) | Sudden 404 spike from CDN | Subset of videos unplayable | Segments are regenerable from raw upload via re-transcode | Bulk re-transcode from raw S3; ~hours to days depending on volume |
| Sudden viral spike (10× traffic in 1 hour on one video) | Per-video QPS gradient alarm | Origin shield miss storm | Auto-prewarm POPs + bitrate cap on extreme load | Auto; alerts if origin egress > 5% of total |
| Geo-fencing leak (US-only content played in EU) | Audit log + compliance scan | Legal/contractual breach | Manifest service enforces geo at request time, not just CDN; rejected with 403 | Patch + audit |

---

## 12. Productionization

### Pre-launch
- [ ] Shadow traffic to new transcoders for 2 weeks; compare VMAF/PSNR/file size against current encoder.
- [ ] Load test playback to 5× peak QPS. Verify CDN tier hit rates hold.
- [ ] Chaos test: kill random transcoding workers under load; verify queue drains.
- [ ] Game day: simulate full CDN provider outage; verify DNS failover < 2 min.
- [ ] Game day: simulate S3 multi-region outage; verify graceful degradation.
- [ ] Soak test watch-progress at 5M writes/sec for 24h; verify no Cassandra hotspots.

### Rollout
- [ ] Feature flag per video-cohort. Start with 1% of videos using new encoder.
- [ ] 1% → 5% → 25% → 100% over 3 weeks; bake at each step.
- [ ] Per-region rollout for new CDN tier: lowest-traffic region first.
- [ ] Auto-rollback: VMAF regression > 1 point, or rebuffer ratio > 1%, or TTFF p99 > 3 sec, or transcode queue lag > 30 min.

### Capacity
- [ ] Reserve 50% headroom on transcoding farm — viral events and product launches.
- [ ] Auto-scale transcode workers on Kafka queue depth (target < 100K backlog).
- [ ] CDN: pre-negotiate burst capacity with each provider for known events (premieres, Super Bowl).
- [ ] Cassandra: provision watch-progress for 3× current write volume; resharding takes hours.
- [ ] S3: monitor bucket request rates; consider key-prefix sharding if hot prefixes appear.

### Migration (if replacing existing system)
- [ ] Dual-encode for 1 month: every new upload runs through old + new pipeline.
- [ ] Diff manifests, segment hashes, VMAF scores. Investigate any regression > 0.5%.
- [ ] Cut over playback traffic gradually: 1% → 100% over a month.
- [ ] Keep old segments for 90 days as rollback insurance.

### Compliance
- GDPR: user delete propagates to: videos owned, watch-progress, comments. Tombstone in metadata; async sweep removes media.
- DMCA: content takedown path with < 24h SLA from notice to takedown.
- Geo-rights: per-region rights metadata enforced at manifest service.
- Data residency: EU user uploads stored in EU buckets; watch-progress in EU region.
- COPPA (kids content): tag, suppress recommendations, hide engagement metrics.

### Cost
- Baseline $/DAU/month for storage+compute: ~$0.10 (rough public estimate). Egress dominates if users are heavy-viewers.
- Cost levers exercised:
  - Codec mix (AV1 phased rollout): -10% egress per year as AV1 adoption grows.
  - Storage tiering: -40% storage cost vs all-hot.
  - Lazy higher-res: -70% transcode cost on long tail.
  - Owned PoPs at ISP boundary: -50% peering cost.
  - ABR cap on inactive tabs: -5% egress.
  - Per-video popularity-aware CDN warming: improves hit rate from 88% → 93%.

---

## 13. Monitoring & metrics

### Business KPIs
- DAU, watch-hours/DAU, sessions/DAU
- Upload rate (videos/hour, hours/min)
- Engagement (watch-time per session, completion rate)
- Subscription churn (Netflix-variant)

### Service-level (SLIs)

#### Playback (the gold metrics)
- **Rebuffer ratio** — fraction of total playback time spent rebuffering. Target < 0.5%. Per-region, per-CDN.
- **Startup latency / TTFF** — time from "play tap" to first frame. p50, p95, p99. Target p99 < 2 sec.
- **Bitrate distribution served** — what % of bytes served at 240/480/720/1080/4K. Watch for downward shifts (network or CDN issues).
- **Playback failure rate** — % of sessions that error before first frame. Target < 0.1%.
- **Mid-stream failure rate** — % of sessions that error mid-playback. Target < 0.5%.

#### Upload
- p99 upload acceptance latency
- Upload success rate
- Time to "watchable" (upload complete → fast-pass renditions ready). p50, p99.

#### Transcode
- Queue depth (Kafka lag) — alert > 10 min
- Per-rendition transcode duration p99
- Job failure rate
- DLQ size (alert if growing)
- Shadow-encode VMAF delta

#### Watch progress
- Write p99 latency. Target < 200ms.
- Write success rate. Target > 99.9%.
- Read p99 latency for "Continue Watching" rail.

### Component metrics

#### CDN
- **Hit rate per tier** — edge target ≥ 90%, shield target ≥ 99%
- **Origin egress as % of total** — target < 1%
- **Per-CDN-provider latency, error rate** — drives auto-failover
- **Per-region cache fill rate** — first-time fetch volume

#### S3
- Request rate (per bucket, per prefix)
- 5xx rate
- Multi-region replication lag

#### Cassandra (watch_progress)
- Read/write latency per node
- Per-partition write skew (detect hot users — bots? account takeover?)
- Compaction queue depth
- Hint queue size
- Cross-region replication lag

#### Kafka (transcode pipeline)
- Producer publish rate, error rate
- Consumer lag per group
- Per-partition skew (detect uneven workload)
- Retention disk utilization

#### Transcoding workers
- Per-worker CPU / GPU utilization (target 70–80%)
- Wallclock-per-real-second encode rate (efficiency)
- Per-codec encode throughput
- Memory + temp-disk utilization

#### Redis (manifest cache, view-count buffer)
- Hit rate
- Eviction rate
- Memory utilization
- Slow log

### Alert routing
- **Page** (24/7): rebuffer ratio > 1% sustained, TTFF p99 > 3 sec, origin egress > 5%, transcode queue > 30 min lag, regional CDN outage, watch-progress write failure > 1%
- **Ticket**: cache hit rate degraded on a single CDN, single-node Cassandra issue, slow shadow-encode anomaly
- **Dashboard-only**: cost trends, popularity tier movement, codec adoption rates

### Dashboards
1. **Playback health** — rebuffer ratio, TTFF, error rates, by region & CDN
2. **CDN performance** — per-provider latency, hit rates, egress volume
3. **Transcode pipeline** — queue depth, per-rendition timing, DLQ
4. **Storage** — tier distribution, growth rate, cost trend
5. **Business** — DAU, watch-hours, upload rate
6. **Cost** — $/DAU, $/hour-watched, per-component cost share

---

## 14. Security & privacy

- **DRM** (Netflix variant required, YouTube optional for premium content) — Widevine (Android, Chrome), FairPlay (Apple), PlayReady (Microsoft). License server issues short-lived tokens; segment encryption keys delivered via license. Compromised devices have license revoked.
- **Token-based segment access** — even non-DRM, segment URLs are signed with user-specific tokens, expire in 1 hour. Prevents hotlinking.
- **Geo-fencing enforcement** — at manifest service, not just CDN. Trust no client-side check.
- **AuthZ on private uploads** — `private` videos require viewer match owner; `unlisted` requires URL knowledge.
- **Upload abuse prevention** — rate limit per user (X uploads/day), per IP, per device. ML-based spam classifier on metadata.
- **CSAM and prohibited content** — perceptual hash matching against PhotoDNA / Content ID at upload time. Auto-block matched content; human review queue for borderline.
- **Account takeover** — anomaly on upload velocity, geographic logins.
- **PII handling** — watch history is sensitive (it reveals interest profile). Encrypted at rest. Access logged. Per-region storage for GDPR.
- **Third-party data access** — auditing all internal queries against watch_progress.

---

## 15. Cost analysis

> **Candidate:** "Egress dominates. Everything else is rounding error."

Rough order-of-magnitude breakdown for YouTube-scale system (illustrative, not authoritative):

| Component | $ share | Driver | Lever |
|---|---|---|---|
| **CDN egress** | **~60%** | Bytes served × $/GB | Multi-CDN, owned PoPs (-50% on those bytes), AV1 (-30% per stream), per-popularity tier mix |
| **Storage (raw + transcoded)** | ~15% | EB-scale | Tiered storage (hot/warm/cold/archive); drop redundant codecs once usage shifts |
| **Transcoding compute** | ~10% | CPU/GPU hours | Lazy higher-res (-70% on long tail); GPU mix; popularity-driven full-pass selection |
| **Metadata / DB** | ~5% | Cassandra/Dynamo at scale | Compaction tuning; archival of old metadata |
| **Compute (services)** | ~5% | CPUs at services | Trivial; standard right-sizing |
| **Recommendations / ML** | ~5% | GPU inference | Batch inference, quantization, model distillation |

**The dominant cost lever is the codec/CDN combination.** AV1 + multi-CDN + owned PoPs together can reduce egress cost by 60% over a 3-year horizon. Engineering effort to build them is significant — but the math at hyperscale is overwhelming.

**Per-DAU cost target:** YouTube reportedly runs at ~$0.50–1.00/DAU/year for infrastructure. Most of that is egress. Driving this number down by 10% is a multi-million-dollar win at scale.

---

## 16. Open questions / what I'd validate with PM

1. **Geo-rights complexity** — for the YouTube variant, are there per-country rights restrictions? (Music rights are notorious here.) Affects manifest service and storage.
2. **Premium / paid tier?** — Affects DRM requirement, codec mix (premium gets AV1 first), and quality (bitrate ladder height).
3. **Live streaming** — confirmed out of scope, but is there a path to add it later? If yes, the Kafka + segmentation architecture must be designed with that future in mind (segments compatible with HLS-LL).
4. **Recommendations latency target** — does the homepage need < 100ms personalized recs? Affects whether to embed candidate generation in the playback service or call out.
5. **Maximum upload size / duration** — caps drive whether we need BPMP-class chunked storage or vanilla S3 multipart.
6. **Subtitles / captions** — multi-language, auto-generated via ASR? Adds a transcode-pipeline dependency on speech-to-text.
7. **Per-region SLA differentiation** — does India (low-bandwidth) get a different default bitrate ladder than US?
8. **Compliance scope** — specifically what flavors of GDPR/DPDP/PIPL apply at launch? Affects market-isolation timing.

---

## 17. Staff-level scorecard for this problem

| Signal | Did the candidate... |
|---|---|
| ✅ Picked YouTube (UGC) over Netflix and explained why | Said upload+transcode is the harder design surface |
| ✅ Identified CDN egress as the dominant cost early | Within first 10 min, in capacity estimation phase |
| ✅ Quantified the transcode workload | "8 hr/sec of new content × 5 renditions" math made explicit |
| ✅ Treated transcoding as a fanout problem with idempotent jobs | Discussed deterministic output keys, retry semantics |
| ✅ Multi-tier CDN with origin shield | Edge → shield → origin, with hit rate targets per tier |
| ✅ Brought up owned PoPs (GGC / Open Connect) at scale | Not just "use a CDN" |
| ✅ Discussed ABR as a client-side decision | Did not put bitrate selection on the server |
| ✅ Codec mix tradeoff with cost-of-encode vs cost-of-egress | AV1 break-even math made explicit |
| ✅ Storage tiering driven by popularity power-law | Hot/warm/cold/archive with movement triggers |
| ✅ Lazy higher-res transcoding | Recognized the $5/asset full pass is wasteful for long tail |
| ✅ Watch progress as a Cassandra/Dynamo problem at 3M w/s | Explained partition design, multi-region writes |
| ✅ TTFF optimizations | Manifest at edge, low-bitrate first segment, DRM preload, HTTP/3 |
| ✅ Rebuffer ratio as the gold metric | Volunteered, with target |
| ✅ Failure modes volunteered | Origin storm, encoder regression, CDN outage, S3 partial |
| ✅ Productionization with shadow encoding canary | Specifically called out VMAF/SSIM comparisons |
| ✅ Cost levers quantified | AV1, owned PoPs, lazy higher-res with ballpark $ savings |
| ✅ Cellular / market isolation surfaced | At least mentioned for compliance + blast radius |
| ✅ Closed with wrap-up speech | Restated the dominant constraints (egress + transcode), mitigations, what to deep-dive next |

### What separates Staff from Senior on this problem

A Senior candidate will:
- Correctly identify the upload → transcode → CDN → playback pipeline
- Pick reasonable databases
- Mention HLS/DASH and ABR

A Staff candidate will additionally:
- **Identify CDN egress as the dominant cost in capacity estimation**, not as an afterthought, and orient design choices around it
- Treat **codec mix as an economic decision** (cost-of-encode vs cost-of-egress break-even math)
- Distinguish **fast-pass vs full-pass transcoding** with explicit cost rationale
- Bring up **owned PoPs / ISP-embedded CDN** as a scale lever, not just "use Akamai"
- Volunteer **shadow-encode canary** as the rollout strategy for new encoders
- Identify **the origin shield miss storm** as a viral-event failure mode and pre-design mitigation
- Discuss **rebuffer ratio and TTFF** as primary SLOs, not afterthoughts
- Volunteer **multi-region active-active + market-isolation** for compliance
- Quantify **$/DAU** and identify the 60% egress dominance
- Discuss **storage tiering driven by popularity power-law** with explicit move triggers

---

## 18. Wrap-up speech (template)

> "To wrap up: the dominant constraints are **CDN egress cost (~60% of infrastructure spend)** and **transcode-pipeline scale (8 hours of video per second of wall-clock time, multi-codec, multi-rendition)**. The architectural choices I made to address those: (1) a three-tier CDN — edge, regional shield, origin — with multi-CDN active-active and owned PoPs at ISP boundaries; (2) a Kafka-driven transcoding farm with fast-pass renditions delivered eagerly and higher-res lazily; (3) codec mix prioritizing AV1 for popular content and H.264 universally; (4) storage tiered by popularity power-law from hot SSD down to Glacier archive; (5) Cassandra for both metadata and watch-progress with multi-region active-active.
>
> The two pieces I'd worry about most in production: (a) **transcoder regressions silently degrading quality** — mitigated with shadow-encode VMAF comparison and auto-rollback on regression; (b) **origin shield miss storms on viral content** — mitigated with request coalescing, anomaly-driven pre-warming, and emergency bitrate caps under extreme load.
>
> To productionize: shadow-encode every new encoder version for 2 weeks, gradual feature-flagged rollout per video-cohort with auto-rollback on rebuffer or VMAF regression, multi-CDN with active monitoring and 60-second failover. SLO-alert on rebuffer ratio (< 0.5%) and TTFF p99 (< 2 sec).
>
> If we had another 30 minutes I'd deep-dive (a) the recommendations / ranking system and how it integrates with the storage tier (popular videos end up promoted into hot storage by popularity) or (b) the live-streaming variant and how the same pipeline extends with HLS-LL and edge transcoding."

---

## 19. Market-based isolation (cellular architecture)

> Surface this in the last 5 minutes if there's time. For Netflix-style, market isolation is **the** architectural anchor — driven by content licensing. For YouTube, it's looser but still binding for compliance.

### What & why

A **market** here means a content-licensing region: e.g., US, EU, UK, India, Brazil, Japan, China (where applicable), each with its own content rights, regulatory regime, and possibly its own ranking models. A **cell** is a per-market replica of: metadata catalog (rights-filtered), watch-progress store, recommendations service, search index. The transcoding farm is **shared** because raw transcoded segments are content-neutral; what differs is which segments are *exposed* in each market.

Drivers:

1. **Content licensing** — Netflix's biggest design pressure. A movie licensed for US is illegal to serve to UK viewers. Per-market `region_catalog` is a hard requirement, not optimization.
2. **Compliance / data residency** — GDPR (EU), DPDP (India), PIPL (China), LGPD (Brazil). Watch history must stay in-region.
3. **Blast radius** — a bad metadata push to EU should not break US.
4. **Latency** — local manifest service, local watch-progress writes.
5. **Per-market features** — India needs aggressive bitrate-ladder lowering for 2G/3G; EU needs cookie consent banners; China needs real-name + content filtering.

### Architecture

```
              ┌─────────────── Global Edge ───────────────┐
              │  GeoDNS + CDN (Akamai / GGC / Open Connect)│
              │  Cache layer is GLOBAL (segments are       │
              │  content-neutral; geo-fence at manifest)   │
              └──────┬────────────────┬──────────────────┬─┘
                     │                │                  │
            ┌────────▼────┐    ┌──────▼─────┐    ┌──────▼─────┐
            │  US Cell    │    │  EU Cell   │    │  IN Cell   │
            │ ┌─────────┐ │    │┌─────────┐ │    │┌─────────┐ │
            │ │Manifest │ │    ││Manifest │ │    ││Manifest │ │
            │ │Playback │ │    ││Playback │ │    ││Playback │ │
            │ │Watch-   │ │    ││Watch-   │ │    ││Watch-   │ │
            │ │ progress│ │    ││ progress│ │    ││ progress│ │
            │ │Catalog  │ │    ││Catalog  │ │    ││Catalog  │ │
            │ │(filtered│ │    ││(filtered│ │    ││(filtered│ │
            │ │ rights) │ │    ││ rights) │ │    ││ rights) │ │
            │ │Recs     │ │    ││Recs     │ │    ││Recs     │ │
            │ │Search   │ │    ││Search   │ │    ││Search   │ │
            │ └─────────┘ │    │└─────────┘ │    │└─────────┘ │
            └─────────────┘    └────────────┘    └────────────┘
                     │                │                  │
                     └────────────────┼──────────────────┘
                                      │
                  ┌───────────────────▼───────────────────┐
                  │   GLOBAL TRANSCODE FARM               │
                  │   (segments shared; cell-specific     │
                  │    is just metadata + manifest mods)  │
                  │   ┌──────────────┐                    │
                  │   │   Kafka      │                    │
                  │   │ transcode.q  │                    │
                  │   └──────────────┘                    │
                  │   Workers, S3 raw + segments          │
                  └───────────────────────────────────────┘
                                      │
                  ┌───────────────────▼───────────────────┐
                  │   GLOBAL CONTENT METADATA (rights DB) │
                  │   Per-region rights metadata,         │
                  │   replicated to cells with per-market │
                  │   filter applied                      │
                  └───────────────────────────────────────┘
```

### Routing

- User home cell determined by registration country + current geo at app open.
- A **traveling user** (US user in EU) — what do they see?
  - Netflix: must show EU catalog (per licensing); their watch history follows them via a cross-cell read.
  - YouTube: shows EU-allowed videos; watch history is global.
- VPN-detected users: depending on policy, either fall back to home market or show a notice (Netflix has explicit anti-VPN measures).

### Cross-cell interactions

The interesting question: a video uploaded by a US user becomes available to EU. How?

- **Transcoded segments** are global (they live in S3, replicated across regions for delivery). Segments don't care about market.
- **Metadata** is replicated to all cells with per-market `geo_block` filter applied.
- **The cell decides whether to expose the video**, not the upload.
- This means: one upload → one set of segments, exposed in N cells with N different rights filters.

For Netflix-style curated catalog: more complex, because licensing varies per movie per country per month. The `rights_db` tracks `(content_id, region, license_window)` and the cell's catalog is a continually-refreshing filtered view. A movie can leave a region overnight (license expires); the catalog removes it; segments stay in S3 for the regions where it's still licensed.

### What's shared, what's per-cell

| Component | Sharing |
|---|---|
| Transcode farm | Shared globally |
| Raw + transcoded segments (S3) | Shared (replicated by region for delivery) |
| CDN | Shared (segments are content-neutral; manifest enforces geo) |
| Metadata DB | Per-cell (rights-filtered view of global rights DB) |
| Search index | Per-cell |
| Recommendations service + model | Per-cell (different models for different markets) |
| Watch-progress store | Per-cell (data residency) |
| Manifest service | Per-cell (enforces geo) |
| Playback service | Per-cell |
| Upload service | Per-cell (residency for raw uploads) |

### Failover

If EU cell goes down: do we serve EU users from US cell? **No** for residency (EU watch-progress can't be written to US-resident database). But we **can** serve **stateless reads** (catalog browse, manifest fetch for already-licensed-globally content) from a fallback cell with cached data. Watch-progress is degraded — players buffer locally and sync when EU cell recovers. This is the cell-as-failure-domain principle: intra-cell HA protects users; cross-cell failover is for stateless paths only.

### What changes operationally

- **Per-cell deploy pipelines.** Encoder change rolls cell-by-cell.
- **Per-cell on-call.** EU cell paging EU on-call, etc. (Or follow-the-sun.)
- **Per-cell capacity planning.** India growth doesn't draft on US capacity.
- **Per-cell cost dashboards.** PMs want to know "what does Brazil cost vs Japan."
- **Per-cell A/B testing.** A new ranking model can test in IN before rolling globally.

### When NOT to do this

- < 50M users globally, no regulatory pressure: skip it; the operational tax is too high.
- Pure-UGC with global content rights and no licensing: lighter-weight regional replication suffices.
- Single-team operation with no cell-aware tooling: deploy hell.

### Staff-signal phrasing

> "For Netflix-variant, cellular by market is the architectural anchor — content licensing dictates it before any other concern. The cell IS the rights-enforcement boundary, the residency boundary, and the blast-radius boundary. The transcode farm is shared globally because segments are content-neutral; what's per-cell is the metadata, watch-progress, recs, and the manifest service that enforces geo. The hard part is the rights DB — a continually-refreshing per-region license window — and the cross-cell interactions when users travel. I'd accept ~30% extra infrastructure cost because it's the only way to survive a licensing audit and a regional outage at the same time."

---
