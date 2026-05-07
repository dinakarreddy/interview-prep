# 05 — Design a Cloud File Storage + Sync System (Dropbox / Google Drive)

> A storage-and-sync problem disguised as a file system problem. The interviewer cares less about REST endpoints and more about whether you correctly identify that **chunking + content-addressed dedup** and **multi-device sync with conflict handling** are the only things that matter — and you spend your time there.

---

## 0. Problem statement (as the interviewer would drop it)

> **Interviewer:** "Design a cloud file storage and sync system like Dropbox or Google Drive. Users have files on their devices, the files sync up to the cloud, and across all of their other devices. They can also share files and folders with other users. Walk me through how you'd build it."

That's the prompt. Notice what's *not* there: no scale numbers, no ranking, no real-time chat, no collaborative editing. Your job is to scope it, not solve it whole.

---

## 1. Clarifying questions

Pick 5–6. Don't fire them as a checklist — interleave with the interviewer's responses.

> **Candidate:** "Before I design, let me scope this. A few questions:"
>
> 1. **"Read-heavy or write-heavy?"** — *Expect: write-bytes-heavy on the upload path (bulk first-time syncs are big), read-heavy on the metadata path (clients constantly poll for changes). Roughly balanced in events.*
> 2. **"What kind of files? Tiny text files or 10GB videos?"** — *Critical for chunking decisions. Assume the full distribution: median ~100KB, p99 ~1GB, max ~100GB.*
> 3. **"How many devices per user, and is offline-first a requirement?"** — *Drives the sync protocol. Expect 3 devices avg, offline-first is a hard yes for desktop clients.*
> 4. **"Is collaborative document editing in scope, or is that a separate problem?"** — *Force the interviewer to commit. OT/CRDT is a whole other problem (Google Docs–style); I'll assume it's out and treat the file as an opaque blob.*
> 5. **"What's the sharing model? Per-file, per-folder, or both? Public links?"** — *Drives the ACL model. Assume per-file and per-folder, transitive on folders, plus public link sharing.*
> 6. **"Compliance — EU data residency, HIPAA, FedRAMP?"** — *EU residency is the realistic FAANG-scale constraint; it forces the cellular question.*
> 7. **"How long do we keep version history?"** — *Drives the versioning storage cost. Assume 30 days for free tier, longer for paid.*
>
> **Out of scope (state explicitly):**
> - Collaborative real-time editing (mention OT / CRDTs, defer to a separate problem)
> - Full-text search of file content (separate ingestion + Elasticsearch problem)
> - Mobile photo-roll auto-backup (a specialized client feature, not core)
> - End-to-end encryption (mention as a tradeoff against dedup; default to server-side encryption at rest)
> - Authentication / SSO (assume an external IdP)
>
> **Candidate:** "I'll design the upload/download pipeline, multi-device sync (push+pull), conflict handling, and the sharing/permissions evaluation path. I'll black-box ranking and search. Sound right?"

> **Interviewer:** "Yes. Go."

---

## 2. Functional requirements

| Priority | Requirement |
|---|---|
| P0 | User uploads a file from a device; it's durable in the cloud |
| P0 | User downloads a file (cold or fresh) from any device |
| P0 | Multi-device sync: a change on device A is reflected on devices B, C within seconds when online |
| P0 | Folder hierarchy: create, rename, move, delete |
| P0 | Conflict detection + resolution when two devices change the same file offline |
| P0 | Sharing: per-file and per-folder ACL; transitive permission on folders |
| P1 | Version history (last 30 days, restore any version) |
| P1 | Trash / recover deleted file |
| P1 | Public share links (read-only or read-write) |
| P1 | Resumable upload for large files / flaky networks |
| P2 | Selective sync (don't sync this folder to this device) |
| P2 | Offline access on mobile (LRU cache) |
| Out | Real-time collaborative editing (Docs/Sheets) |
| Out | Full-text content search |
| Out | Photo gallery / media auto-tagging |

---

## 3. Non-functional requirements

| Dimension | Target | Justification |
|---|---|---|
| Durability | 11 nines (10⁻¹¹ annual loss probability) for committed bytes | This is the user contract — losing a wedding photo is unacceptable; matches S3 SLA |
| Availability | 99.95% on read; 99.9% on write | Read path is desktop-app front door; write absorbed by client retry buffer |
| Sync latency | p50 < 5s, p99 < 30s end-to-end (save on A → visible on B) | Anything past 30s feels broken to a user actively switching devices |
| Upload throughput | Saturate user's uplink (bottleneck at user, not server) | Service must not be the slow part |
| Read latency | First-byte p99 < 200ms for cached chunks; p99 < 2s for cold-tier | Match perceived "snappy" desktop file open |
| Consistency | Strong on metadata (file tree); eventual on cross-device sync | Tree corruption is unrecoverable; sync delay is forgivable |
| Scale | 700M registered users, 200M DAU, 3 devices avg, 100GB avg per user, total ~50EB stored, 10B file events/day | Dropbox-class |
| Geography | Global, with hard EU data residency for opted-in markets | GDPR + customer asks |

---

## 4. Capacity estimation

> **Candidate:** "Round numbers, no calculator. Stop me if you want a deeper number anywhere."

### Users & devices
- 700M registered, 200M DAU
- Avg 3 devices per active user → ~600M online client connections at peak
- Concurrent active devices (in-foreground): ~50–100M

### Storage
- 100GB avg/user × 700M users = **70 EB raw**
- Dedup ratio ~3–4× across the population (OS files, app installers, shared docs) → effective stored bytes **~17–20 EB**
- 3× replication on object storage (or erasure-coded equivalent ~1.4×) → **~25–30 EB physical disk**
- This is the dominant cost line item. Storage tiering is non-negotiable.

### File events
- 10B events/day across all users → ~115K events/sec avg, ~350K events/sec at peak (3× peak factor)
- An "event" = one chunk upload, one rename, one delete, one share, etc.

### Upload bandwidth
- Assume 10% of DAU does a meaningful upload session per day, avg 50MB
- 200M × 0.1 × 50MB = **1 PB/day uploaded** (post-dedup, this drops to ~250–300 TB/day persisted)
- Per second avg: ~12 GB/s sustained; peak ~36 GB/s

### Download bandwidth
- Reads are skewed: a file is uploaded once, read N times across devices
- Heuristic: 3× the upload bandwidth (3 devices pulling each upload) = **~36 GB/s sustained** through download path
- This is *internal* — Dropbox owns the PoPs and serves clients via private peering. Public-internet egress is a separate, much smaller line.

### Chunking
- 4MB chunks (Dropbox-style)
- 50EB / 4MB = ~10¹⁶ chunks. SHA-256 hash = 32 bytes → 320 PB just for the hash index. **Hashes go in a sharded KV, not a single index.**

### Metadata
- File rows: ~10B files (avg ~14 files per user × 700M)
- Per-file metadata row: ~500 bytes → **5 TB raw metadata**
- Multiplied by version history (avg 5 versions/file): ~25 TB
- This fits in a sharded SQL fleet without exotic infra. The hot path is "list folder", which is 100s of rows, well-indexed.

### Sync events log
- 10B events/day × 30-day retention = **300B rows**
- Avg row 200 bytes → **60 TB**
- Cassandra/DynamoDB territory. Sharded by user_id, time-clustered.

**The two punchline numbers:** 50EB total bytes (storage cost dominates) and 10B events/day (sync infrastructure must be event-driven, not poll-based at scale).

---

## 5. API design

> **Candidate:** "Public REST + signed URLs. The hot byte path is direct-to-blob-store; the API tier only handles metadata."

### Block (chunk) upload — bytes go directly to object storage

```
POST /v1/blocks/upload-init
  body: { sha256, size }
→ 200 { exists: bool, upload_url, upload_expires_at }
   - exists=true means another user already has this chunk; client skips upload
   - upload_url is a signed URL (S3/GCS) good for ~1 hour
```

```
PUT <upload_url>   (direct to object storage)
  body: <bytes>
→ 200
```

```
POST /v1/blocks/commit
  body: { sha256, size }
→ 200 { block_id }
```

### File commit — metadata only

```
POST /v1/files/commit
  body: {
    path: "/foo/bar/x.docx",
    parent_rev: "<previous server rev or null>",
    blocks: [sha256, sha256, ...],   // ordered list of chunk hashes
    file_size, mtime, mime_type
  }
→ 200 { file_id, rev: "<new server rev>", server_modified: <ts> }
→ 409 { conflict, current_server_rev, current_blocks } when parent_rev mismatch
```

- `parent_rev` is the optimistic concurrency token. If the server's current rev doesn't match the client's `parent_rev`, we return 409 and let the client decide (typically: create a "conflicted copy").

### Sync — delta query

```
GET /v1/changes/longpoll?cursor=<opaque>&timeout=30
→ 200 { changes: [], cursor: <opaque>, has_more: false, backoff: 0 }
   - long-poll: server holds the connection up to `timeout` sec
   - returns immediately if there are pending changes for this user
```

```
GET /v1/changes/list?cursor=<opaque>&limit=1000
→ 200 { changes: [{file_id, op: created|modified|deleted|moved, rev, path, ...}], cursor, has_more }
```

### Download

```
GET /v1/files/{file_id}?rev=<rev>
→ 302 redirect to signed download URL OR 200 with chunk list
```

```
GET /v1/blocks/{sha256}
→ 302 redirect to signed download URL (CDN-cacheable for hot blocks)
```

### Sharing

```
POST /v1/sharing/folders/{folder_id}/members
  body: { email, role: "viewer"|"editor" }
→ 200

POST /v1/sharing/links
  body: { file_id_or_folder_id, role, expiry, password? }
→ 200 { link_url }
```

### Anti-patterns to avoid
- **Don't** send file bytes through your API tier — they go straight to blob storage via signed URL. Saves enormous bandwidth on your own services.
- **Don't** return a "list of files" with bytes — return file IDs + chunk lists.
- **Don't** make the sync endpoint return all changes in one shot — paginate by cursor.
- **Don't** use timestamps for cursors — opaque tokens (encoding a (shard, monotonic_id) pair) survive shard rebalance.

---

## 6. Data model

### `users` (sharded by user_id, MySQL or Spanner)
| Column | Type | |
|---|---|---|
| user_id | bigint | PK |
| email | string | unique |
| home_region | enum | EU / US / APAC — drives data residency |
| storage_quota | bigint | bytes |
| created_at | timestamp | |

### `files` — the file tree, sharded by user_id (MySQL or Spanner)
| Column | Type | |
|---|---|---|
| file_id | bigint (Snowflake) | PK |
| user_id | bigint | shard key |
| parent_folder_id | bigint | indexed |
| name | string | |
| is_folder | bool | |
| current_rev | string | latest committed revision |
| size | bigint | |
| mime_type | string | |
| created_at, modified_at | timestamp | |
| deleted_at | timestamp nullable | for trash |

- Composite index on `(user_id, parent_folder_id, name)` — uniqueness per folder, hot path for "list folder".
- A folder rename is a single-row update — *if* the design avoids materializing the full path.

> **Important design choice:** I'm storing **parent pointer**, not full path. A folder rename of `/a/b` to `/a/c` then becomes O(1), not O(descendants). The price: computing a file's path requires walking up the parent chain. We mitigate with a per-user path cache (Redis) keyed by `file_id`.

### `file_versions` — chunk lists per revision
| Column | Type | |
|---|---|---|
| file_id | bigint | shard with `files` |
| rev | string | PK with file_id |
| chunk_list | array<sha256> | ordered |
| size | bigint | |
| created_at | timestamp | |
| author_device_id | string | |

### `chunks` — content-addressed dedup index (sharded by hash prefix, DynamoDB or sharded MySQL)
| Column | Type | |
|---|---|---|
| sha256 | binary(32) | PK |
| size | int | |
| storage_region | enum | EU / US / APAC — where the bytes live |
| storage_path | string | object-store key (or ID) |
| refcount | int | for GC |
| first_seen_at | timestamp | |

- Sharded by hash prefix (first 2 bytes) → 65,536 logical shards, mapped to a few hundred physical shards.
- Refcount: incremented when any user references this chunk in a `file_versions` row; decremented on deletion / version expiry. Chunks with refcount=0 → eligible for GC.

### `acl` — per-resource ACL (sharded by resource_id)
| Column | Type | |
|---|---|---|
| resource_id | bigint | file or folder |
| principal_id | bigint | user or group |
| role | enum | owner / editor / viewer |
| inherited_from | bigint nullable | parent folder if inherited |
| granted_at | timestamp | |

### `sync_events` — per-user activity log (Cassandra / DynamoDB)
| Column | Type | |
|---|---|---|
| user_id | bigint | partition key |
| event_id | bigint (monotonic per user) | clustering key |
| event_type | enum | file_created / file_modified / file_deleted / file_moved / share_added / etc. |
| file_id | bigint | |
| rev | string | |
| timestamp | timestamp | |
| origin_device_id | string | so the originating device can skip the event |

- Partition by user_id, clustered by event_id ascending. The cursor is `(user_id, last_seen_event_id)`.
- TTL: 30 days. Long-paused devices that miss the window do a "full re-scan" via `/v1/files` enumeration.

### `device_sessions` — active device → user mapping
| Column | Type | |
|---|---|---|
| device_id | string | PK |
| user_id | bigint | |
| last_seen_event_id | bigint | what cursor we last delivered |
| websocket_gateway_host | string | which notification host owns this conn |
| last_heartbeat | timestamp | |

### Why these choices
- **Sharded MySQL / Spanner for `files` and `file_versions`** — this is the "tree" and we need transactional moves and renames. A folder move where the destination already exists must be atomic. Cassandra would force us to serialize tree mutations through a per-user lock; SQL gives it for free per shard.
- **DynamoDB for the chunk dedup index** — pure key-value, billions of keys, single-digit-ms reads. SQL would be possible but DynamoDB's auto-sharding is the killer feature for a 10¹⁶-key namespace.
- **Cassandra for `sync_events`** — wide-row, write-heavy, time-partitioned. Exactly the pattern.
- **Object storage (S3 / GCS) for chunk bytes** — durability is their problem, not ours. We pay for storage and get 11 nines.

---

## 7. High-level architecture

```
   Desktop / Mobile / Web client
   ┌────────────────────────────────┐
   │  Client SDK                    │
   │  - watches local filesystem    │
   │  - chunks files                │
   │  - hashes (SHA-256)            │
   │  - maintains local index       │
   │  - long-polls / WS             │
   │  - resolves conflicts          │
   └────────────────┬───────────────┘
                    │ HTTPS
                    ▼
          ┌────────────────────┐
          │   Edge (PoP)       │  TLS termination, WAF, rate limit
          │   + private CDN    │  Dropbox owns PoPs → cheap egress
          └─────┬────────┬─────┘
                │        │
       bytes ▼        ▼ metadata / control
   ┌──────────────┐  ┌─────────────────────────────────────────┐
   │ Block        │  │       Metadata API Gateway              │
   │ Service      │  │  authn (OAuth/PoP), rate-limit, routing │
   │ - presigns   │  └────┬────────┬─────────┬─────────────────┘
   │   upload URL │       │        │         │
   │ - dedup      │   files│   sharing│   sync│
   │   check      │       ▼        ▼         ▼
   │ - commit     │  ┌─────────┐ ┌────────┐ ┌────────────────┐
   │ - presigns   │  │ File    │ │ Sharing│ │ Sync /         │
   │   download   │  │ Service │ │ Service│ │ Notification   │
   │              │  │ (tree)  │ │ (ACL)  │ │ Service        │
   └──────┬───────┘  └────┬────┘ └────┬───┘ └─────┬──────────┘
          │               │           │           │
          │               │           │           │ long-poll / WS
          ▼               ▼           ▼           │
   ┌──────────────┐  ┌──────────┐ ┌──────────┐    │
   │  Object      │  │ files    │ │ acl      │    │ writes events
   │  Storage     │  │ DB       │ │ DB       │    ▼
   │  (S3/GCS)    │  │(sharded  │ │(sharded  │  ┌────────────────┐
   │  + CDN cache │  │ SQL)     │ │ SQL)     │  │ sync_events    │
   │              │  └────┬─────┘ └──────────┘  │ (Cassandra)    │
   │ chunk dedup  │       │                     └────────────────┘
   │ index        │       │ optimistic concurrency
   │ (DynamoDB)   │       ▼
   └──────────────┘  ┌─────────────────────┐
                     │  Kafka              │
                     │  topic: file.events │
                     └────┬────────────────┘
                          │
                          ▼
                  ┌─────────────────┐
                  │ Notification    │  ── pushes to per-device WS
                  │ Fanout Service  │
                  └─────────────────┘
```

### Request walk-through

**Upload path (client saves file `report.pdf`, 12MB):**
1. Client SDK detects local fs change. Splits file into 3 × 4MB chunks. Hashes each.
2. Client SDK calls `POST /v1/blocks/upload-init` for each hash. Block Service checks dedup index (DynamoDB).
   - Hash 1 already exists globally → server returns `exists: true`. **Client skips upload entirely.**
   - Hash 2, 3 are new → server returns signed `upload_url`s.
3. Client uploads new chunks directly to S3 via signed URL. Bytes never touch our API tier.
4. Client calls `POST /v1/files/commit` with `parent_rev` (current local rev) and the ordered chunk hash list.
5. File Service:
   - Validates chunks exist (cheap dedup-index lookup).
   - Begins txn on user's metadata shard.
   - Checks `current_rev == parent_rev`. If mismatch → return 409 (conflict).
   - Writes new `file_versions` row, updates `files.current_rev`.
   - Increments refcount on each chunk.
   - Writes `sync_events` row.
   - Publishes `file.committed` event to Kafka.
   - Returns 200 with new `rev`.
6. Notification Fanout Service consumes Kafka event, looks up user's other devices, pushes to their open WebSocket connections.
7. Other devices receive push, immediately call `/v1/changes/list` for the deltas they just got notified about, and download the new chunks.

**Download path (device B sees file is new, fetches it):**
1. Device B calls `/v1/files/{file_id}?rev=<rev>`. File Service returns chunk list (sha256 sequence).
2. Device B checks local cache for chunks. For chunks it already has (from other files — dedup wins client-side too!), skip.
3. For missing chunks, device B calls `GET /v1/blocks/{sha256}` → redirects to a CDN-cacheable signed URL.
4. Device B fetches bytes, validates hash, assembles file.

### Why this split (data plane vs control plane)

- **Bytes never traverse our API tier.** This is the single most important architectural choice. At 12 GB/s sustained upload, putting bytes through a service tier would require an extra ~$10M/yr in compute and bandwidth. Signed URLs to object storage cost nothing extra.
- **Metadata operations are small and transactional.** They live on a SQL fleet that's optimized for the "list folder" and "atomic move" hot paths.
- **The dedup check is the bridge.** It's the only metadata lookup on the byte path, and it's a single-key DynamoDB read — fast and cheap.

---

## 8. Deep dives

### Deep dive 1: Chunking strategy

> **Interviewer:** "Walk me through how you split a file into chunks and why."

**Candidate:**

Two main families:

**(A) Fixed-size chunking** — split every 4MB. Simple, fast (no hashing needed for split decisions). The pathology: if the user inserts 1 byte at the start of a 1GB file, every chunk's content shifts → every hash changes → re-upload the whole file. No dedup benefit on shifted content.

**(B) Content-defined chunking (CDC)** — use a rolling hash (Rabin fingerprint, FastCDC, Buzhash) to find natural "anchor" points in the byte stream. Chunk boundaries shift with content. The 1-byte insert above only affects the chunk that contains it; downstream chunks are stable.

| Aspect | Fixed | CDC |
|---|---|---|
| Splitting CPU | ~0 | ~5–10% of a core for SHA-rolling hash at line rate |
| Dedup on edits in middle of file | Bad | Excellent |
| Implementation | Trivial | Moderate (boundary tuning, min/max chunk size guardrails) |
| Compatibility | Same chunks across clients only if file is identical | Same chunks across clients regardless of where file came from |

**My pick:** **Fixed 4MB chunks** — same choice Dropbox made historically.

Why? Three reasons:
1. The dedup win from CDC mostly applies to *edits* of large files. The biggest dedup wins in practice are *full-file dedup* — OS images, app installers, shared documents — where both algorithms perform identically.
2. Fixed-size lets us pre-compute chunk boundaries without buffering the entire file. CDC requires a sliding window pass.
3. Fixed-size means a file's chunk list is deterministic from `(file_size)` alone — easier validation, easier debugging, easier to recover an in-flight upload after a crash.

If the workload were *append-heavy logs* or *frequently edited large binaries*, I'd flip to CDC. For general-purpose cloud storage, fixed wins on simplicity.

> **Interviewer:** "Why 4MB and not 1MB or 64MB?"

**Candidate:** It's a tradeoff between:
- **Small chunks** → better dedup granularity, more parallel upload streams, but higher per-chunk overhead (every chunk has a hash row, a refcount, a metadata round-trip).
- **Large chunks** → lower overhead, but worse dedup, slower retransmit on a single failed chunk.

4MB is the empirical sweet spot. At 4MB:
- 100GB file = 25K chunks → 800KB of metadata per file, fine.
- A failed upload retransmits 4MB, not 64MB.
- Parallel upload of 8 chunks saturates a 1 Gbps link in ~250ms.

### Deep dive 2: Content-addressed dedup — and the privacy footgun

> **Interviewer:** "You mentioned dedup. Where does it happen — client or server?"

**Candidate:**

This is a subtle and often-fumbled question. There are three options:

**(1) Server-side dedup, blind:** Client uploads every chunk. Server hashes, looks up, drops the duplicate. Simple. Wastes bandwidth — the whole point of dedup is to avoid the upload.

**(2) Client-side dedup with global hash check (what Dropbox actually did):** Client computes SHA-256 of chunk, asks server "do you have this?", uploads only if no. Saves enormous bandwidth.

**(3) Per-user-only dedup:** Client-side, but server only confirms dedup against chunks owned by *this user*.

Option (2) has a known privacy / security flaw. **An attacker can probe whether a specific file exists in the system** by computing its hash and asking. If yes-without-upload, someone has it. Pre-2011 Dropbox was attacked this way — researchers proved you could detect specific copyrighted files in the system.

The fix is **client-side encryption per-user OR proof-of-possession (PoP)**:
- **Convergent encryption** — encrypt with `key = H(plaintext)`. Identical plaintexts still produce identical ciphertexts → dedup still works. But it's a known weakness against confirmation-of-file attacks.
- **Proof-of-possession** — client must prove it has the bytes (e.g., respond to a challenge over a server-chosen byte range) before the server confirms dedup. Defeats hash-only probing.

**My pick:** Option (2) with **per-account dedup as the default tier** and **global dedup as an opt-in** for accounts that explicitly accept the privacy cost. Or: option (2) with proof-of-possession on every dedup hit. The dedup ratio will drop from ~4× to maybe ~2.5× under per-account, but cross-tenant leak risk is gone.

For end-to-end encrypted accounts (paid Business / Advanced tier), **dedup is impossible across users** because ciphertexts differ. Customers pay for that privacy; the storage cost is the price.

> **Interviewer:** "What's the dedup ratio you'd expect?"

**Candidate:** Industry public numbers cluster at 3–5× for general consumer file storage. The wins concentrate in:
- macOS/Windows install files in user-uploaded backups (~50% of bytes for some users)
- Shared corporate documents (one PDF emailed to a 1000-person team, all backed up)
- App data, library files, common media (memes, viral images)

If we deduped only within an account, ratio drops to ~2×. The cross-user dedup is where the savings live. **At 50EB raw, a 4× dedup ratio saves ~37EB of physical storage.** At ~$0.01/GB/month retail object storage, that's $370M/yr saved. This single optimization is the difference between a profitable and unprofitable cloud storage business.

### Deep dive 3: The sync protocol

> **Interviewer:** "How does device B find out that device A made a change?"

**Candidate:**

Pure poll → too expensive. 600M devices × 1 poll/sec = 600M QPS to the metadata fleet just for "is anything new?" answers.

Pure push → harder than it looks (NAT, mobile sleep, connection churn).

**Hybrid: long-polling + WebSocket push, with poll-fallback.**

Layered like this:

1. **Steady state:** each connected device holds a long-poll HTTP request open to the **Notification Service** with `?cursor=<last_seen_event>&timeout=30`. Server holds the connection until either:
   - A new event arrives for this user → respond with the event(s)
   - 30s elapse → respond with `no_changes`, client immediately re-issues
2. **Active session (mobile foreground / desktop):** upgrade to WebSocket. Server pushes events as they happen, sub-second.
3. **Sleeping mobile:** Apple/Google push notification (APNs/FCM) wakes the app on event. App reconnects, syncs delta, sleeps again. Uses platform-native backgrounding.
4. **Hard offline / extended absence:** when device reconnects after >30 days (the `sync_events` retention), the cursor is invalid. Client falls back to **full re-scan** — recursively list folders, diff against local index, sync diffs.

The "delta sync" itself is straightforward once we have a cursor:
- Client says "I've seen up to event X for me." Server returns events X+1..X+N.
- Each event has enough detail for the client to take action without further metadata calls (chunk list included for `file_modified`).

**Why long-poll, not pure WebSocket?**
- WebSocket through corporate proxies and odd network paths is unreliable at the long tail.
- Long-poll is HTTP — works through every proxy, every CDN, every firewall.
- WebSocket is a *latency optimization* on top of long-poll, not a replacement. Both code paths exist.

**What goes into the cursor?**
The cursor is opaque, but encodes `(user_shard_id, last_event_id, schema_version)`. We do *not* expose timestamps — that would couple clients to clock skew and make migrations harder.

> **Interviewer:** "How many open connections does that mean?"

**Candidate:** ~50–100M concurrent active devices. Long-poll holds an HTTP request open ~30s; WS holds it indefinitely. At ~10K concurrent connections per Notification Service host, we need ~5K–10K hosts globally. Sharded by `user_id`, sticky session per device. Auto-scale on connection count.

### Deep dive 4: Conflict resolution

> **Interviewer:** "Two devices edit the same file offline. Both come back online. What happens?"

**Candidate:**

This is the most user-visible part of the entire system. Get it wrong and users lose work. Three options:

**(A) Last-writer-wins (LWW).** Newer commit wins; older edit is lost. **Wrong choice.** Silent data loss. Never do this for user files.

**(B) Operational transform / CRDT.** Merge the two edits into a unified result. Works only for *structured* file types (text, JSON). Useless for binaries (PNG, ZIP, PDF).

**(C) Conflicted copy.** Server detects the parent_rev mismatch on commit. Accepts the second commit, but renames it to `report (Conflicted copy from <device_name>) <date>.pdf`. Both versions preserved; user resolves manually.

**My pick:** **Conflicted copy as the default, with optional CRDT for known-structured types.**

The conflict-detection mechanism is the `parent_rev` optimistic-concurrency token in `POST /v1/files/commit`:

```
Device A: parent_rev = "v3"  → commits → server now at "v4"
Device B: parent_rev = "v3"  → commits → server returns 409 because current_rev = "v4"
Device B (client SDK): receives 409, generates new filename "report (Conflicted copy ...).pdf",
                       commits as a new file with parent_rev = null.
```

Now both `report.pdf` (with A's edits) and `report (Conflicted copy ...).pdf` (with B's edits) exist. User opens the conflicted copy, decides to keep or merge.

> **Interviewer:** "What about a folder rename collision? Two devices rename the same folder simultaneously."

**Candidate:** Same mechanism. The folder row has a `current_rev` too. Whoever commits second sees 409. Client SDK applies the loser's rename as a no-op (the folder is already renamed; just to a different name) — or keeps the local name and creates a divergent state, depending on policy. We'd typically prefer "first writer wins for renames; second client adopts the new name silently" — folder renames are less destructive than file content edits.

> **Interviewer:** "Why not vector clocks?"

**Candidate:** Vector clocks would give us strict causal-order detection — "is event A's history a strict prefix of event B's, or are they concurrent?" That's more precise than parent_rev (which is just one Lamport-like counter). But vector clocks scale with the number of writers per file. For a folder with 50 collaborators, every file row has a 50-entry vector. The bookkeeping cost outweighs the benefit because *the user-visible action is the same either way* — show conflicted copy. We don't need to *characterize* the conflict; we just need to *detect* it.

### Deep dive 5: Sharing & permission evaluation

> **Interviewer:** "User shares a folder with 1000 collaborators. One of them tries to read a file 5 levels deep in that folder. How do we authorize the read?"

**Candidate:**

Two options:

**(A) Materialize ACLs onto every descendant.** When you share folder F with user U as editor, write 1 ACL row per descendant file/folder. Pros: read-time check is O(1) — single lookup on the file. Cons: share/unshare on a folder with 1M descendants writes 1M rows. And rename/move requires re-materializing.

**(B) Check ancestor chain at read time.** When user U asks for file F, walk up the folder chain checking ACLs at each level. Pros: share/unshare is O(1). Cons: read-time O(depth) — typically 5–10 levels.

**My pick:** **Hybrid — option (B) for the source of truth, with a materialized cache for hot paths.**

- ACL writes go to the canonical row (option B). Cheap.
- Read-time evaluation walks ancestors. With a Redis cache keyed by `(file_id, user_id) → role`, the second access to the same path is O(1).
- Cache TTL: 60s. Permission changes are not real-time; users can wait a minute for revoke to propagate. (Sensitive workflows — e.g., a "kick out everyone NOW" admin action — invalidate the cache via a fanout topic.)

Hot path optimization: on the desktop client, the SDK **knows the folder it's asking about**. Server can precompute the ACL of that folder once per session (~1 walk) and reuse for all file accesses inside.

> **Interviewer:** "What if the cache is stale and a revoked user accesses a file?"

**Candidate:** That's the explicit tradeoff. We accept up to 60s of stale access. For higher-stakes workflows (paid Business tier with "instant revoke"), we publish revocation events on a low-latency topic that all gateways subscribe to and invalidate immediately. The default 60s TTL is a cost-vs-correctness knob.

### Deep dive 6: Versioning and garbage collection

> **Interviewer:** "How do you handle version history without exploding storage costs?"

**Candidate:**

Two levers:

**(1) Per-chunk versioning.** A new file revision only stores chunks that changed. Chunks unchanged across revs share storage via the dedup mechanism (refcount += N). Editing one paragraph of a 1GB document stores ~4MB of new bytes, not 1GB.

**(2) Aggressive expiry.** Free tier: 30-day version history. Paid tier: 180 days or unlimited. After expiry, old `file_versions` rows are purged, refcounts decremented, GC catches orphaned chunks.

**Garbage collection** is the operational risk:
- A chunk with refcount=0 is eligible for GC.
- We don't delete immediately — race condition with concurrent commits (a user might be committing right now with this chunk hash).
- Mark-and-sweep with a 7-day cooldown: when refcount hits 0, mark with a tombstone timestamp. GC sweeper runs daily, deletes only tombstoned chunks older than 7 days.
- Alternative we don't use: synchronous deletion at refcount 0. Too risky; a single race condition causes user data loss.

> **Interviewer:** "What's the GC backlog look like?"

**Candidate:** At 10B events/day, we're churning a meaningful chunk volume — call it 100M chunk deletions/day on average (after dedup, after expiry). GC is async, sharded by chunk hash prefix. We monitor `gc_backlog_age` — alert if oldest pending tombstone is >14 days. (Should be close to 7 days steady state.) Small backlog is fine; runaway backlog means our refcount math is wrong somewhere — that's a P1 alert.

### Deep dive 7: Resumable upload / partial failure

> **Interviewer:** "User uploads a 10GB file on a flaky home Wi-Fi. Connection drops at 30%. What happens?"

**Candidate:**

The chunk-based upload model **already** handles this:
1. Client splits the file into 2500 chunks of 4MB.
2. Uploads them in parallel (say 8 at a time).
3. Each chunk's upload is independent — a network drop affects only the chunks in flight.
4. On reconnect, client SDK checks which chunks were `commit`-ed. Re-uploads only the missing ones.
5. Once all 2500 chunks are committed to dedup index, client calls `/v1/files/commit`.

The file is never "half-uploaded" from the metadata point of view. Either commit happened (file visible at new rev) or it didn't. Bytes that were uploaded for an aborted attempt are still in the dedup index and may be reused — refcount stays at 0 until something references them, GC eventually cleans up.

No special "resumable upload" protocol needed. **The architecture is inherently resumable by virtue of being chunked.**

For multi-day uploads (terabyte-scale), the only addition is chunk hash persistence on the client — so a client crash doesn't lose track of which chunks already succeeded.

---

## 9. Tradeoffs & alternatives

### Decision: Fixed-size chunking

| Option | Pros | Cons | When to pick |
|---|---|---|---|
| **✅ Fixed 4MB** | Trivial to implement, deterministic, fast splitting | Insertion in middle of large file = full re-upload of subsequent chunks | General consumer cloud storage |
| **CDC (Rabin / FastCDC)** | Stable boundaries → great dedup on edits | More CPU, complex tuning, harder debugging | Backup of edited binaries, log archives |
| **Variable per-file-type** | Adapt to content semantics | Massive complexity, schema explosion | Specialized backup product |

### Decision: Client-side dedup with global cross-user check

| Option | Pros | Cons | When to pick |
|---|---|---|---|
| **No dedup** | Simple, perfect privacy | 4× the storage cost; impossible at our scale | Toy systems |
| **Per-user dedup** | OK privacy, ~2× ratio | Misses the biggest wins (shared docs, OS files) | Privacy-first products |
| **✅ Cross-user dedup with PoP** | ~4× ratio, no probing attack | Modest CPU on PoP challenge | Default consumer tier |
| **Cross-user dedup, no PoP** | ~4× ratio, simpler | Confirmation-of-file privacy attack | Avoid; this is the historical Dropbox flaw |
| **E2E encrypted, no dedup** | Maximum privacy | No dedup, cost passed to user | Premium tier |

### Decision: Sharded SQL (or Spanner) for the metadata tree

| Option | Pros | Cons | When to pick |
|---|---|---|---|
| **PostgreSQL, single primary** | Familiar, ACID | Doesn't scale to 700M users / 10B files | Smaller scale |
| **✅ Sharded MySQL by user_id** | Linear scale, transactional moves within a user, mature | App-side sharding logic, no cross-user transactions | Hyperscale with per-user-isolated workloads |
| **Spanner / CockroachDB** | Global consistency, no sharding logic | More expensive, slightly higher latency | If we wanted multi-region transactions |
| **Cassandra / DynamoDB** | Linear scale, simple ops | No transactions → atomic moves require app-side coordination | When write volume dominates and trees aren't deep |

For the **chunk dedup index**, I'd flip to DynamoDB: pure KV pattern, billions of keys, no transactions needed. SQL would also work but DynamoDB's auto-sharding eliminates ops.

### Decision: Long-poll + WebSocket for sync notifications

| Option | Pros | Cons | When to pick |
|---|---|---|---|
| **Pure polling** | Trivial | 600M QPS for nothing-changed answers; mobile battery drain | Tiny scale |
| **Pure WebSocket** | Lowest latency | NAT/proxy issues at the long tail; harder to scale to 100M concurrent | Inside controlled networks |
| **Pure server push (APNs / FCM)** | Battery-friendly on mobile | High latency on desktop, vendor-dependent | Mobile-only |
| **✅ Hybrid (long-poll + WS + APNs/FCM)** | Each path covers the others' gaps | Complexity (3 transports) | Multi-platform consumer products |

### Decision: Conflicted copy (not LWW, not OT/CRDT)

| Option | Pros | Cons | When to pick |
|---|---|---|---|
| **Last-writer-wins** | Simplest | **Silent data loss — never** | Caches, not user files |
| **OT / CRDT merge** | Single canonical version | Only works for structured content | Collaborative editing (Docs) |
| **✅ Conflicted copy** | Never lose data; user-resolvable | Two files where there should be one | Opaque-blob file storage |
| **Block edit (return error)** | No conflict ever | Bad UX (offline edits become useless) | Critical financial documents |

### Decision: Object storage + private CDN for byte path

Alternative: run our own object store. This is what Dropbox eventually did (project "Magic Pocket") to escape S3's $/GB. At their scale ($MM/month on egress), it paid back. For an interview answer, I'd say: "S3 to start, build an internal store once it's clearly cheaper at scale (~$50M/yr in storage spend)." The architecture is the same — just swap the backend.

### Decision: Eventual consistency on cross-device sync; strong on metadata

| Scenario | Model | Mechanism |
|---|---|---|
| Device A commits, device B reads | Eventual (≤30s) | sync_events + push |
| Atomic folder move | Strong | DB txn within user shard |
| ACL change → file access | Eventual (≤60s) | Cache TTL; explicit invalidation for high-stakes |
| Quota check | Strong | DB-backed counter |
| File version history | Strong | Write-through |

---

## 10. Defending architectural choices — interviewer attacks

> **Interviewer:** "Why not just use S3 directly? Why have a Block Service at all?"

**Candidate:** Three things the Block Service does that raw S3 doesn't:
1. **Dedup index lookup** — S3 doesn't know if "this hash already exists". We need a separate index regardless.
2. **Signed URL minting** — clients can't be trusted with raw S3 credentials. The Block Service mints scoped signed URLs.
3. **Refcounting and GC orchestration.** S3 doesn't know our chunks have refs from multiple users; we manage that.

Without it, every client would need cross-tenant S3 IAM — operationally a nightmare and a security disaster.

> **Interviewer:** "Why not put metadata in DynamoDB instead of sharded SQL?"

**Candidate:** The deal-breaker is atomic folder operations. A folder move with 10K descendants needs to be transactionally atomic *or* the system has to handle partial-state user-visible weirdness ("half my photos moved"). DynamoDB's transactions cap at 100 items; our move could touch 10K. SQL's per-shard transactions handle it natively. We pay with sharding pain (which is well-understood). I'd flip to Spanner if we wanted global transactions, but DynamoDB is the wrong shape for the tree.

> **Interviewer:** "What if a user has 10 million files in one folder?"

**Candidate:** That's pathological — Dropbox/Drive would have thrown a UI warning long before. Practical limits: ~100K files per folder, enforced at API level. For users who need more, recommend they nest into subfolders. The DB would handle 10M rows in a folder fine; the *client* listing it would not.

> **Interviewer:** "Why not show a version every time the file changes — keep all versions forever?"

**Candidate:** Storage cost. Even with per-chunk versioning, every edit creates new metadata rows. At 10B events/day × infinite retention, the metadata grows linearly forever. Beyond cost, users can't navigate 10K versions of a file. 30 days is the empirical "useful window" — long enough to undo accidents, short enough to bound storage.

> **Interviewer:** "Your conflict resolution creates a duplicate file. Doesn't that just push the problem to the user?"

**Candidate:** Yes — and that's the right call for opaque files. The system can't merge a PNG. The system can preserve both versions and tell the user "you have two; pick one or merge manually." The alternative (LWW) silently destroys data. **Better to push a manual-merge UI burden than to lose work.** For *structured* file types, we'd add merge automation as a layered feature, not a foundation.

> **Interviewer:** "What if the Notification Service is down?"

**Candidate:** Devices fall back to long-poll on the metadata API tier (the same `/v1/changes/longpoll` endpoint, served by a different cluster). Latency degrades — sync goes from sub-second to 30s — but **the system stays up**. Notification Service is a latency optimization; loss of it should never block a sync.

> **Interviewer:** "What's the worst case for upload of a brand-new 100GB file?"

**Candidate:**
- 25,000 chunks at 4MB each.
- Client uploads 8 chunks in parallel; each chunk is ~4MB / link_speed = ~32s on a 1 Mbps uplink, ~32ms on 1 Gbps.
- On 1 Gbps: ~25K chunks / 8 concurrent = 3,125 sequential rounds × 32ms = ~100s, ~1.7 minutes. Bottleneck is the network, not us.
- On 1 Mbps: ~25K chunks / 8 concurrent × 32s = ~100K seconds = ~28 hours. User's link is the bottleneck, *not us*. We don't time out — chunks succeed independently.
- Metadata ops are negligible: 25K dedup checks at 5ms each = 125 sec total, parallel with upload, dominated by upload.

The system never re-uploads from scratch; client SDK persists the chunk-success log to local disk and resumes after any failure.

---

## 11. Failure modes & mitigations

| Failure | Detection | Blast radius | Mitigation | Recovery |
|---|---|---|---|---|
| Object storage bucket region outage | S3 region health | Reads fail in that region | Multi-region replication for chunks; serve from secondary region | AWS recovery, automatic |
| Metadata shard primary down | Per-shard write error rate | 1/N of users blocked from writes | Sync replica promotion (~30s); reads continue from replicas | Auto-failover |
| Dedup-index hot key (a viral OS image) | Per-key QPS | Every install of macOS hits the same key | Cache the popular hashes in Redis at every gateway; absorb 99% of reads | Promote to permanent cache |
| Notification Service host loss | Per-host conn count drops | Devices on that host disconnect | Devices reconnect, get re-sharded; sticky session reassigned | LB health-check, automatic |
| Client uploads partial chunks then dies | No commit happens | Bytes orphaned (refcount 0) | GC cleanup after 7 days | Background sweep |
| Conflict storm (many clients editing same file) | Per-file commit rejection rate | One file's edits keep losing | Backoff with jitter on retry; surface "merge required" UI | User-driven |
| Share permission cache stale after revoke | Audit log of access vs ACL state | Up to 60s of unauthorized reads | Explicit invalidation on revoke for high-stakes accounts; SLO on ACL prop | Manual page if SLO breaches |
| Kafka file.events lag | Consumer lag metric | Cross-device sync delayed | Auto-scale Notification Service consumers; alert at >30s lag | Drain backlog |
| Sync-events table approaches retention | Disk utilization, oldest event age | Long-paused devices fall off cursor | Force full re-scan at next reconnect (always-correct fallback) | Capacity bump |
| Zombie device with stale auth | OAuth token validation | Reads continue under revoked credentials until token expiry | Short-lived tokens (1h); refresh-token revocation list | Token rotation |
| Metadata tree corruption (rare bug) | Consistency checker | A user's files vanish from listing | Per-user backup of `files` table snapshots; replay from `sync_events` | Manual recovery |
| Bad dedup hash collision | Hash mismatch on read-back | Wrong file content returned (one in 2¹²⁸ — won't happen with SHA-256) | Verify hash on download; fallback to upload-from-source if mismatch | Detection >> recovery |
| Cross-region link cut | Inter-region latency | Cross-region sharing degrades | Local-region cache of remote chunks; clients tolerate temp degradation | Network team |
| Client SDK bug corrupts local index | Crash / hash mismatch | Single device's sync state broken | "Reset and re-sync" command; client downloads full tree, throws away local index | User-triggered |

---

## 12. Productionization

### Pre-launch
- [ ] Dark launch: shadow real production sync events through new pipeline for 4 weeks. Compare event ordering, conflict outcomes against legacy.
- [ ] Load test to 5× projected peak (50B events/day equivalent). Verify p99 sync end-to-end < 60s.
- [ ] Chaos: kill random metadata shards during load. Verify recovery without lost commits.
- [ ] Game day: simulate full S3 region outage. Verify fail-over to secondary region completes within 5 min, no data loss.
- [ ] Privacy review on dedup confirmation-of-file vector — sign off from security team on the proof-of-possession scheme.

### Rollout
- [ ] Client SDK: feature-flagged by user-bucket. **Critical:** an SDK bug can corrupt local data — start at 0.1%, not 1%.
- [ ] Bake 7 days at 0.1% before going to 1%. Then 1% → 5% → 25% → 100% over 4–6 weeks.
- [ ] Per-region rollout: smallest data-residency market first.
- [ ] Auto-rollback if: sync latency p99 > 60s for 10 min, or commit error rate > 0.5%, or any single user reports data loss attributable to new path.
- [ ] **Server-side flag separate from client-side.** Server can disable the new path even if SDK has it. Belt and suspenders.

### Capacity
- [ ] Storage planning: plan for 18 months of growth (storage migration is a 12+ month project).
- [ ] Object-storage cost projection: $/GB × dedup ratio × growth rate. Tie to FinOps quarterly review.
- [ ] Metadata DB: pre-shard for 5× current. Resharding live SQL is hard.
- [ ] Notification Service: auto-scale on connection count; reserve 50% headroom for incident-driven reconnect storms.

### Migration (if replacing existing system)
- [ ] **Critical:** old client SDK and new client SDK must coexist for 12+ months. Old clients in the wild for years.
- [ ] Schema must be backward-compatible. Adding fields, never renaming or removing.
- [ ] Dual-write to old and new chunk stores for 6 months. Background reconciler.
- [ ] Read traffic shifts gradually; old stays for rollback.
- [ ] Decommission old only after old-SDK MAU drops below 1%.

### Cost
- Estimated $/DAU: storage dominates. At 100GB/user × $0.01/GB-month (after dedup) = $1/user/month *gross*. Net (after dedup, tiering, deduction of inactive users) closer to $0.20–$0.40/DAU/month. Most of that is paid by paid-tier subscribers.
- **Cost levers (in priority):**
  1. Dedup ratio — every 0.1× improvement saves $30M+/year.
  2. Cold-tier migration (chunks not accessed in 90 days → S3 Glacier or in-house cold). Saves ~70% on the 80% of bytes that are cold.
  3. Compression on metadata (zstd) — ~3× on metadata DB.
  4. Aggressive version expiry on free tier.
  5. PoP-based egress savings via owned peering — Dropbox's "Project Magic Pocket" reduced egress cost by ~75%.

### Compliance
- **GDPR delete:** user delete must remove all data within 30 days. Fan out delete request to (a) metadata DB rows, (b) sync_events, (c) chunks where this user was the *only* referent. Chunks shared with other users keep the chunk; just decrement refcount.
- **Data residency:** EU users' data lives in EU cell; cross-cell sharing is its own design problem (see §19).
- **Audit log:** every share, every access on Business-tier accounts is logged for 7 years.
- **Right to portability (GDPR Art. 20):** user can download a ZIP of all their files. Implemented as a long-running job → pre-signed URL.

---

## 13. Monitoring & metrics

### Business KPIs
- DAU, MAU, paid-tier conversion
- Bytes uploaded per DAU
- Dedup ratio (storage-wide and per-tier)
- Storage growth rate (TB/day)
- Trash and version restore rate (proxy for "system reliability for users")

### Service-level (SLIs)
- **Upload init**: p50/p99 latency, error rate
- **File commit**: p50/p99 latency, conflict (409) rate, error rate
- **Changes long-poll**: p99 time-to-first-event, connection error rate
- **Block download** (signed URL latency): p99 first-byte
- **End-to-end sync latency**: time from device-A commit to device-B "received event" (synthetic probe)
  - **Primary SLO**: p99 < 30s end-to-end
- **ACL eval p99**: cache hit ratio + cold-path latency
- **Conflict rate**: 409 commits / total commits (target <0.5%)

### Component metrics

#### Block Service
- Dedup hit rate (target ~80%+)
- Signed URL mint rate, error rate
- Refcount increment / decrement rate
- GC backlog age (oldest pending tombstone)

#### File / Metadata Service
- Per-shard write QPS, p99 latency
- Replication lag per shard
- Conflict rate per user (detect "stuck client" patterns)
- Move/rename count vs commit count

#### Sync / Notification Service
- Concurrent connections (per host, per region)
- Long-poll fanout latency
- WebSocket disconnect rate
- Kafka consumer lag on file.events

#### Sharing Service
- ACL cache hit rate (target ≥99%)
- ACL lookup p99
- Permission revoke propagation time

#### Object Storage / CDN
- Per-region read/write latency
- 5xx rate from S3
- CDN cache hit rate on chunk reads (target ≥85%)
- Cross-region replication lag

#### DBs
- DynamoDB throttling rate on chunks index
- MySQL/Spanner replication lag, slow queries
- Cassandra (sync_events) compaction queue

### Alert routing
- **Page (24/7):** any data-loss signal (commit lost, chunk hash mismatch, refcount went negative), regional outage, sync end-to-end p99 > 60s for 10 min
- **Ticket (next business day):** dedup ratio drift, ACL cache hit rate degraded, GC backlog growing, single-shard issues
- **Dashboard-only:** cost trends, dedup ratio per tier, capacity planning

### Dashboards
1. **Sync health** — RED metrics, end-to-end latency, conflict rate
2. **Storage health** — bytes stored, dedup ratio, GC backlog, cold-tier migration rate
3. **Metadata DB** — per-shard, replication lag, slow queries
4. **Notification gateway** — connection count, push latency, regional split
5. **Sharing/ACL** — cache hit, ACL eval p99, revoke propagation
6. **Business** — DAU, paid conversion, bytes/user, top file types

---

## 14. Security & privacy

- **Encryption in transit:** TLS 1.3 everywhere; mTLS service-to-service.
- **Encryption at rest:** AES-256 server-side. Per-account KMS keys for paid tier.
- **End-to-end encryption (paid tier):** client-side encryption with user-held key. **Disables cross-user dedup.** Customers accept the storage cost.
- **OAuth2 + device-bound tokens.** Short-lived access tokens (1h); refresh tokens stored in OS keychain, revocable server-side.
- **Confirmation-of-file attack mitigation:** proof-of-possession challenge before dedup-skip is granted (random byte-range MAC).
- **Public link sharing:** unguessable token (≥128 bits entropy); optional password; optional expiry; revocable.
- **Sensitive content:** ML scanner for CSAM (legally required), malware (signature scan + heuristics), copyrighted content (DMCA pipeline).
- **Audit log:** every share, ACL change, public-link create, admin action — append-only, 7-year retention for Business tier.
- **Compliance:** GDPR (right to erasure, right to portability, data residency), HIPAA (Business Associate Agreement available on Business tier), SOC2, ISO27001.

---

## 15. Cost analysis

Rough order-of-magnitude:

| Component | Cost driver | Optimization lever |
|---|---|---|
| Object storage (chunks) | $/GB/month × physical bytes | Dedup ratio, cold-tier migration, erasure coding (vs 3× replicate) |
| Metadata DB | DB-host instances + IOPS | Compression, TTL on old versions |
| Sync events log | Cassandra instances | 30-day TTL, compression |
| Egress (downloads to clients) | $/GB transferred | Owned PoPs (Dropbox Magic Pocket); private peering with ISPs |
| CDN (chunk distribution) | $/GB cached + $/GB egress | Long TTL (chunks immutable by hash); aggressive prefetch for popular hashes |
| Compute (Block / File / Sharing) | CPU-hours | Auto-scale on QPS |
| Notification gateway | Connection-hours | Right-size; reduce keepalive frequency |

**Largest single line item: bulk object storage (~50–60% of infra cost).** Dedup ratio is the single biggest lever — every 0.5× of dedup improvement saves $50M+/yr at 50EB scale.

**Second largest: egress** (downloads to user devices). Dropbox famously eliminated this by owning their PoPs and peering directly with consumer ISPs. Public-cloud egress would cost ~$0.05/GB outbound; private peering brings it to ~$0.005/GB or free.

**$/DAU rough envelope:** $0.30–$0.80/DAU/month, dominated by paying-customer storage. The free tier is a customer-acquisition cost for the paid tier.

---

## 16. Open questions / what I'd validate with PM

1. **Free vs paid tier split.** Free users drive most QPS but drive zero revenue; what's the cost ceiling per free user before we throttle their device count or quota?
2. **Real-time collaboration scope.** Are we shipping Google Docs–style editing on top of this in a year? If yes, the file model needs an "operations log" hook *now* — adding it later is hard.
3. **Compliance roadmap.** Which specific regulations? GDPR, HIPAA, FedRAMP all have radically different infra implications. FedRAMP in particular forces a separate cell.
4. **Mobile auto-backup.** If photo backup is a big driver of bytes uploaded, we may want a separate, optimized ingest path (HEIC-aware chunking, image-content dedup).
5. **Selective sync UX.** Is "don't sync this folder to this device" a P0? It changes the metadata model — we'd need per-device sync state, not just per-user.
6. **End-user encryption at scale.** Is it a small premium feature or moving toward default? Has massive implications on dedup ratio (and thus storage cost).

---

## 17. Staff-level scorecard for this problem

| Signal | Did the candidate... |
|---|---|
| ✅ Identified chunking + dedup as the central insight | Within first 10 min, with the dedup ratio quantified |
| ✅ Quantified storage cost and dedup leverage | "4× dedup saves 37EB and ~$370M/yr" |
| ✅ Explained the dedup privacy attack (and the fix) | Confirmation-of-file → proof-of-possession |
| ✅ Picked databases with defended reasons | SQL for tree (transactions); DynamoDB for chunks (KV scale); Cassandra for events (wide-row) |
| ✅ Described conflict resolution as "conflicted copy", not LWW | And explained why LWW is wrong |
| ✅ Designed bytes path to bypass API tier | Signed URLs to S3 — the single most important architectural choice |
| ✅ Treated sync as long-poll + WS hybrid, not just WS | And described the fall-through path |
| ✅ Volunteered failure modes before being asked | Region outage, hot key, GC backlog, conflict storm |
| ✅ Discussed productionization unprompted | SDK rollout caution, dual-write migration, version coexistence |
| ✅ Discussed cost in concrete dollar terms | Storage as 50–60% of infra; dedup ratio as #1 lever; egress via owned PoPs |
| ✅ Brought up cellular / data residency unprompted | EU cell, cross-cell share complexity |

### What separates Staff from Senior on *this* problem

A Senior candidate will:
- Correctly draw the chunked upload to S3 with a signed URL
- Pick reasonable databases
- Mention the dedup pattern

A Staff candidate will additionally:
- Quantify the dedup ratio and tie it to a dollar figure
- Identify the dedup privacy attack and propose proof-of-possession
- Pick "conflicted copy" with conviction and articulate why LWW is wrong
- Make the conflict resolution policy a per-file-type decision (opaque vs structured)
- Identify SDK rollout as the highest-risk production change (not server)
- Surface the cellular / data-residency question without prompting
- Discuss the multi-year SDK coexistence problem (clients in the wild)
- Identify GC and refcount as a non-trivial reliability surface

---

## 18. Wrap-up speech (template)

> "To wrap up: the dominant constraints here are storage cost (50EB raw, dominated by 100GB/user × 700M users) and sync correctness across devices. The architectural answers: 4MB fixed-size chunks with content-addressed cross-user dedup behind a proof-of-possession check, signed URLs for the byte path so bytes never traverse our API tier, sharded SQL for the metadata tree to get atomic moves, DynamoDB for the dedup index because it's pure KV at billions of keys, Cassandra for the sync events log, and a hybrid long-poll + WebSocket sync notification path with APNs/FCM for mobile. Conflict resolution is 'conflicted copy', never last-writer-wins. The two pieces I'd worry about in production are (1) the dedup-index hot-key problem on viral chunks — mitigated by a Redis cache layer in front and aggressive CDN — and (2) the multi-year SDK rollout, where bad client code can corrupt local user data, mitigated by 0.1%-bake rollouts and dual-flagging server-side. To productionize: feature-flag at the SDK and server independently, dark-launch the new pipeline shadowing legacy for 4 weeks, SLO-alert on end-to-end sync latency p99 < 30s. If we had another 30 minutes, I'd deep-dive (a) the cellular architecture for EU data residency and cross-cell sharing, or (b) the cold-tier migration policy and refcount-driven GC."

---

## 19. Market-based isolation (cellular architecture) — bonus / "if time permits"

> Bring this up in the last 5 minutes if there's time. For Dropbox/Drive, this isn't optional — Dropbox operated a separate "Dropbox EU" deployment for years specifically because of GDPR. Frame as: *"If we had more time, the next architectural layer I'd add — and one Dropbox actually built — is cellular isolation by market."*

### What & why

A **cell** for cloud storage is a self-contained per-region replica: its own object storage region, its own metadata DBs, its own chunk dedup index, its own Notification Service, its own Sharing Service. A **market** is the unit of business+geographic isolation: US, EU (often a separate cell because of GDPR), APAC, India, etc.

Drivers (in priority order at FAANG scale):

1. **Data residency / compliance** — GDPR for EU, India's DPDP Act, China's PIPL, FedRAMP for US government tenants. These are hard legal requirements, not preferences. Dropbox-EU exists for this exact reason.
2. **Blast radius reduction** — a corruption bug in the EU cell does not affect US users.
3. **Latency** — EU users hit EU object storage; saves the trans-Atlantic 80–150ms RTT on cold-chunk reads.
4. **Operational independence** — capacity, deploys, on-call, schema migrations all per-cell.
5. **Customer commitments** — "Your data never leaves the EU" is a sales line item enterprises pay extra for.
6. **Geopolitical risk** — a market may need to be cleanly separated for regulatory or political reasons (e.g., Russia, China).

### Architecture

```
        ┌──────────────────── Global Edge / DNS ────────────────────┐
        │  GeoDNS routes user to home cell at signup               │
        └────────┬─────────────────┬──────────────────┬─────────────┘
                 │                 │                  │
       ┌─────────▼────────┐ ┌─────▼────────┐ ┌──────▼────────┐
       │   US Cell        │ │   EU Cell    │ │  APAC Cell    │
       │ ┌──────────────┐ │ │ ┌──────────┐ │ │ ┌──────────┐  │
       │ │ Block Svc    │ │ │ │Block Svc │ │ │ │Block Svc │  │
       │ │ File Svc     │ │ │ │File Svc  │ │ │ │File Svc  │  │
       │ │ Sharing Svc  │ │ │ │Sharing   │ │ │ │Sharing   │  │
       │ │ Sync Svc     │ │ │ │Sync      │ │ │ │Sync      │  │
       │ │ Object store │ │ │ │Object    │ │ │ │Object    │  │
       │ │ Metadata DB  │ │ │ │Metadata  │ │ │ │Metadata  │  │
       │ │ Dedup index  │ │ │ │Dedup idx │ │ │ │Dedup idx │  │
       │ └──────────────┘ │ │ └──────────┘ │ │ └──────────┘  │
       └─────────┬────────┘ └─────┬────────┘ └──────┬────────┘
                 │                │                  │
                 └────────────────┴──────────────────┘
                              │
                  ┌───────────▼───────────┐
                  │ Cross-Cell Bridge     │
                  │ - User → home cell    │
                  │   directory           │
                  │ - Cross-cell shares   │
                  │   (ACL + chunk fetch) │
                  └───────────────────────┘
```

### Routing: how a request lands in the right cell

- Every user has a `home_cell` set at signup (based on country, optionally migratable).
- A small global **User Directory** (replicated KV cache, sub-ms reads at the edge) maps `user_id → home_cell`.
- Authentication and routing happens at the global edge; once the cell is known, all storage and metadata operations stay in-cell.
- Public sharing links contain a cell hint (`https://drop.example/cell-eu/<token>`) so they go directly to the right cell.

### The hard part: cross-cell interactions

The interesting design question is **what happens when a US user shares a folder with an EU user?**

The asymmetric question:
- **Permission grant** (the ACL row) is small and globally relevant — it should be replicated to both cells.
- **Bytes** are large and may be residency-restricted — they may *not* be replicated to the recipient's cell.

Three approaches:

| Approach | How | Pros | Cons |
|---|---|---|---|
| **Replicate chunks to recipient's cell** | When share happens, copy all bytes to the recipient's cell | Reads are local; dedup works in recipient cell | Massive replication cost; may violate residency in some markets |
| **Cross-cell read (federation)** | Recipient's read goes to source cell via Bridge | No bytes replicated; clean residency story | Cross-region latency on every chunk fetch; failure-coupling |
| **Selective replication** | Replicate chunks lazily on first access; cache in recipient cell with TTL | Pays only for accessed bytes; respects residency for source-restricted files | Complex eviction; may not respect residency if recipient is in restricted market |

**My pick:** **Cross-cell read with aggressive caching** as the default. Replicate the *metadata* (file tree row, version row, chunk hash list) but not the bytes. On first chunk read, fetch from source cell, cache in recipient cell's CDN with TTL = 30 days. Cap cache size per share. For residency-restricted content (EU-only flagged files), the recipient *must* read cross-cell every time — no caching outside source cell allowed.

The Bridge service is the chokepoint:
- Owns the user-directory cache.
- Mediates cross-cell ACL evaluation.
- Manages the cross-cell chunk fetch (with auth, with rate limiting, with residency policy enforcement).

### Dedup across cells — the painful tradeoff

Within a cell, dedup ratio is ~4×. Across cells, **dedup is impossible by design** — that would require a global chunk store, which violates residency.

The cost: a popular OS image is duplicated once per cell. With 5 cells, dedup ratio drops effectively to (4× per cell × duplication-across-cells) — net maybe 3.5× instead of 4×. That's a real cost (tens of millions/yr at scale) but residency dominates.

**Don't try to do cross-cell dedup.** It's the right call but it's a real money line.

### Cell tradeoffs vs single global system

| Dimension | Global | Cellular | Net |
|---|---|---|---|
| Total storage cost | Lower (full dedup) | Higher (~10–15% more bytes) | Cellular pays a tax |
| Engineering complexity | Lower | Higher (cell awareness everywhere) | Significant tax — pays back at compliance |
| Compliance | Often impossible | Easy | Cellular wins, often non-negotiable |
| Blast radius | 100% on bug | 1/N per cell | Cellular wins |
| Latency for in-cell reads | Higher | Lower | Cellular wins for ≥80% of traffic |
| Cross-cell sharing UX | Native | New problem to solve | Cellular pays a tax |

### What changes in the design when going cellular

- **User Directory** — new global service, replicated KV, sub-ms at edge.
- **Cross-Cell Bridge** — service handling cross-cell ACL eval, chunk fetch, share grant replication.
- **Per-cell dedup index** — no global hash check; chunks may exist multiple times globally.
- **Per-cell Kafka topics** — file.events stays in-cell. Cross-cell events go through Bridge.
- **Per-cell rollouts** — feature flags now have a `cell` dimension. Roll new SDK to APAC first (smallest cell), then EU, then US.
- **Failover** — a US cell outage does NOT route to EU (residency). Intra-cell HA (multi-AZ) is what protects users; cross-cell failover is not a thing for storage.
- **Shared-link routing** — public links carry a cell hint or hit a global router that resolves to the right cell.

### Operational implications

- **Deploy pipelines** per cell. CI/CD must understand cells.
- **Schema migrations** per cell, staggered. A cell is the unit of "stable" while the next migrates.
- **On-call** per cell. Pages route to cell-aware oncall. EU cell outage at 03:00 UTC pages the EU oncall, not Pacific.
- **Capacity planning** per cell. EU storage growth ≠ US.
- **Cost dashboards** per cell.
- **Customer support** per cell — EU enterprise support sees only EU customers' data.

### When NOT to do cellular

- Pre-scale: under ~10M users globally, the operational tax exceeds the benefit.
- No regulatory pressure and no enterprise customers asking.
- Single team unable to operate N cells.
- Workload is fundamentally cross-market with high cross-cell read ratio (e.g., a globally collaborative product where the average share crosses cells).

### What "Staff signal" sounds like here

> "I'd default to a single global system until either (a) we have an enterprise GDPR-compliance ask, or (b) we cross 50M DAU. The first cell I'd carve out is EU, exactly the way Dropbox did with Dropbox-EU. The hard part of cellular for storage isn't the cells themselves — it's cross-cell sharing, and the right answer there is replicate-metadata-globally, fetch-bytes-on-demand-with-caching, and accept that cross-cell dedup is impossible. The tax is real — ~10–15% more storage, a Bridge service to maintain forever, schema migrations that take quarters not weeks — but it's the only design that survives an EU regulator audit, a regional AWS outage, and a 'we lost trust in vendor X in market Y' geopolitical event without losing all your customers at once."
