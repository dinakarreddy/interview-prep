# 06 — Design Slack / Discord (workspace messaging with channels, threads, search, real-time)

> The thing the interviewer is testing here is whether you can hold three concurrent design problems in your head at once: a **WebSocket fanout system** (millions of long-lived connections), a **multi-tenant message store** (workspace-isolated, append-heavy), and a **search system** (per-workspace ACL-aware index). The trap is to over-design any one of them. Staff signal is recognizing that **workspace-as-cell** is the core architectural primitive, and connection management is the dominant operational headache.

---

## 0. Problem statement (as the interviewer would drop it)

> **Interviewer:** "Design a workspace messaging product like Slack or Discord. Users join workspaces, post messages in channels (public, private, DMs), reply in threads, and see messages delivered in real time. They can also search the history. Walk me through the design."

That's all you get. The interviewer is implicitly inviting you to scope — there are 10 sub-systems hiding in that paragraph. Pick what to design and what to defer.

---

## 1. Clarifying questions

Pick 5–6, interleaved with the interviewer's responses. Don't ask all of them.

> **Candidate:** "Before I scope, a few questions:"
>
> 1. **"Slack-style or Discord-style? Workspace-isolated, or large open communities?"** — *Drives multi-tenancy model. Slack: thousands of users per workspace, strong isolation. Discord: 100K+ members per server, more open. Assume hybrid.*
> 2. **"What's the scale — DAU, concurrent connections, messages per day?"** — *Expect: 30M DAU, 20M concurrent connections, 10B messages/day. The concurrent-connection number is the headline.*
> 3. **"What's the largest channel size we need to support — fanout target?"** — *Drives the channel-fanout design. Expect: top channels have 100K+ members.*
> 4. **"Are voice/video, file storage internals, integrations in scope?"** — *Push out of scope explicitly. They're each their own design problem.*
> 5. **"What's the latency target for message-send-to-deliver?"** — *Expect: p99 < 500ms end-to-end (sender hits send → receiver's client renders).*
> 6. **"Is search a P0 launch feature, and what's the recall vs. freshness tradeoff?"** — *Search lag of seconds is fine; missing a result is bad. Drives Elasticsearch indexer design.*
> 7. **"Multi-region? Data residency requirements?"** — *Slack Enterprise customers have hard regional pinning. EU workspaces in EU.*
>
> **Out of scope (state explicitly):**
> - Voice/video (separate problem — see `07-video-call.md`)
> - File storage internals (use object storage as a black box, gloss it)
> - Screen sharing
> - Third-party integrations / bots / webhooks (mention as integration surface, don't design)
> - Authentication (OAuth/OIDC, assume it exists)
> - Compliance retention / e-discovery deep-dive (mention, don't design)
>
> **Candidate:** "I'm going to design **(a) the real-time message-send-and-deliver path** including channel fanout, **(b) message persistence and ordering**, and **(c) search**. Presence and notifications I'll cover as a deep-dive secondary. Sound right?"

> **Interviewer:** "Yes."

---

## 2. Functional requirements

| Priority | Requirement |
|---|---|
| P0 | User connects to the service via persistent connection (WebSocket) and receives messages in real time |
| P0 | User posts a message in a channel (public, private, DM) and it's persisted + fanned out to channel members |
| P0 | User replies in a thread (parent message + ordered replies) |
| P0 | User receives messages they missed while offline (catch-up on reconnect) |
| P0 | Workspace isolation — messages in workspace A never leak to workspace B |
| P0 | Message search across a workspace, ACL-aware (private channel results filtered) |
| P0 | Mentions extracted, notifications fired (desktop + mobile push) |
| P1 | Edit / delete a message (last 24h editable) |
| P1 | Reactions (emoji on a message) |
| P1 | Presence indicators (online / away / offline) |
| P1 | Typing indicators |
| P2 | Message pinning, bookmarks, reminders |
| Out | Voice/video, file internals, screen-share, integrations, e-discovery |

---

## 3. Non-functional requirements

| Dimension | Target | Justification |
|---|---|---|
| Availability | 99.95% (~22 min/month) for connect+send; 99.99% for read | Real-time tools have low tolerance for outages — Slack down = whole company down |
| Send-to-deliver latency | p50 < 100ms, p99 < 500ms | Conversation feels broken past 1s |
| Connect latency (WS handshake) | p99 < 1s | Reconnect storms must complete fast |
| Search latency | p95 < 1s | Users tolerate search slower than messaging |
| Consistency | Per-channel total order; eventual cross-channel | Messages in #general must be in agreed order; "did A post before B in #random?" — don't care |
| Durability | No message loss after 200 OK to sender | User-generated content; legal/compliance store |
| Scale | 100M MAU, 30M DAU, 20M concurrent WS, 10B messages/day | Hybrid Slack+Discord scale |
| Workspaces | 5M total, avg 50 users; top 1% have 100K+ users (large enterprises) | Long-tail distribution drives celebrity-channel-style problems |
| Geography | Global, multi-region with workspace-pinning | Workspace-as-cell — EU workspaces in EU |

---

## 4. Capacity estimation

> **Candidate:** "Let me work the numbers — they'll drive most of the choices."

### Users / connections
- 30M DAU
- 20M concurrent WebSocket connections at peak (mobile + desktop clients held open)
- Connection lifetime: hours (workday for Slack, evenings for Discord)

### Messages
- 10B messages/day
- Avg per-second: 10B / 86,400 ≈ **115K messages/sec**
- Peak (3×): **~350K messages/sec**

### Fanout work (the punchline)
- Avg channel size: ~50 members. Median DM = 2.
- But heavy-tail: largest channels have 100K members. A handful of #general channels in 100K-user enterprises = 100K-member channels.
- Avg fanout per message: ~20 (most messages go to small channels and DMs).
- **Avg fanout writes/sec: 350K × 20 = 7M deliveries/sec** at peak.
- **Tail:** one message in a 100K-member enterprise #general = 100K deliveries.
- This is the channel-as-celebrity problem — not as extreme as Twitter celebs, but the same shape.

### Storage
- Avg message: 200 bytes text + 100 bytes metadata = ~300 bytes
- 10B/day × 300 bytes = **3 TB/day** for message records
- Per year: ~1.1 PB (replication 3× = ~3.3 PB/yr)
- Plus search index (Elasticsearch): ~1.5–2× the raw text size = ~2 TB/day index growth
- Plus media (files, images): out of scope for storage internals, but messaging metadata references them

### Bandwidth (WebSocket frames)
- Avg sent message + delivered message round-trip: ~1 KB on the wire (with JSON envelope)
- 7M deliveries/sec × 1 KB = **7 GB/sec egress** at peak (just for message delivery)
- Plus presence heartbeats (every 30s × 20M = 660K/sec → 30 MB/s)
- Plus typing indicators, reaction events, edit events — call it 10 GB/s peak total egress to clients

### Concurrent connection cost
- 20M connections / 10K connections per gateway host = **2,000 gateway hosts**
- Each holds long-lived TCP — RAM dominated, not CPU. ~100 KB per connection (kernel buffers + per-connection state) = 1 GB / 10K connections = manageable.
- **The cost driver here is gateway compute, sized for connection count not message rate.**

### Decision drivers from the numbers
- 20M concurrent WS → dedicated stateful Gateway tier, sticky, sharded by user_id
- 7M deliveries/sec → in-memory pub/sub (Redis, or proprietary) — not persisted per delivery
- 10B msg/day → Cassandra (write-heavy, append-only, partitioned by channel)
- Search at scale → Elasticsearch, async-indexed via Kafka, per-workspace shard

---

## 5. API design

> **Candidate:** "Two surfaces: a REST/HTTP control plane, and a WebSocket data plane. I'll keep them separate."

### Connection (WebSocket)

```
GET /v1/gateway → 200 { gateway_url, session_token }
   then: client opens WS to gateway_url with session_token

WS frames (JSON envelope):
  Client → Server:
    { type: "MESSAGE_CREATE", channel_id, content, idempotency_key, parent_message_id?, client_msg_id }
    { type: "TYPING", channel_id }
    { type: "PRESENCE", status: "online"|"away" }
    { type: "ACK", up_to_message_id }   # client confirms it has rendered up to this id
    { type: "PING" }                     # heartbeat

  Server → Client:
    { type: "MESSAGE_NEW", channel_id, message: {...} }
    { type: "MESSAGE_EDIT", channel_id, message_id, new_content, edited_at }
    { type: "MESSAGE_DELETE", channel_id, message_id }
    { type: "REACTION_ADD", channel_id, message_id, emoji, user_id }
    { type: "TYPING", channel_id, user_id }
    { type: "PRESENCE_UPDATE", user_id, status }
    { type: "READY", session_id, missed_messages_cursor }  # initial post-connect snapshot
    { type: "RECONNECT", reason }                          # graceful drain
    { type: "PONG" }
```

### REST (HTTP/2)

```
POST /v1/messages
  body: { workspace_id, channel_id, content, parent_message_id?, idempotency_key }
  → 201 { message_id, ts }
  # Used as fallback when WS unavailable; primary path is WS for established sessions.

GET /v1/channels/{channel_id}/messages?before={message_id}&limit=50
  → 200 { messages: [...], has_more }

GET /v1/channels/{channel_id}/messages/{message_id}/thread?cursor=&limit=
  → 200 { parent: {...}, replies: [...], next_cursor }

POST /v1/channels { workspace_id, name, type: "public"|"private"|"dm", members[] }
POST /v1/channels/{channel_id}/members { user_id }
DELETE /v1/channels/{channel_id}/members/{user_id}

GET /v1/search?workspace_id=&q=&channel_id?=&from?=&to?=&user_id?=
  → 200 { hits: [{message_id, channel_id, snippet, score}], total, next_cursor }

POST /v1/messages/{message_id}/reactions { emoji }
PATCH /v1/messages/{message_id} { content }      # edit
DELETE /v1/messages/{message_id}                  # delete
```

### Anti-patterns to avoid
- Don't expose **server-issued message IDs** as the only identifier in the WS write path — clients send `client_msg_id` for optimistic UI, server returns server `message_id` in the ack.
- Don't make search a synchronous fanout to all workspace shards — pre-shard by workspace_id.
- Don't return raw timestamps as ordering keys — use server-issued sequence + ts (see §6 ordering).

---

## 6. Data model

### Tables / collections

#### `workspaces` (sharded PostgreSQL — small, transactional)
| Column | Type | |
|---|---|---|
| workspace_id | bigint (Snowflake) | PK |
| name | string | |
| plan | enum | free / paid / enterprise |
| home_region | string | drives cell pinning |
| created_at | timestamp | |

#### `users` (sharded PG)
| Column | Type | |
|---|---|---|
| user_id | bigint | PK |
| primary_workspace_id | bigint | FK |
| email | string | unique |
| profile_meta | json | |

#### `workspace_memberships` (sharded by workspace_id)
- (workspace_id, user_id) → role, joined_at
- Hot read path: "is user X in workspace W?"

#### `channels` (sharded by workspace_id, co-located)
| Column | Type | |
|---|---|---|
| channel_id | bigint | PK |
| workspace_id | bigint | partition key (co-locate with workspace) |
| type | enum | public / private / dm / mpdm |
| name | string | |
| created_at | timestamp | |

#### `channel_memberships` (sharded by workspace_id, indexed by both channel_id and user_id)
- (channel_id, user_id) — needed both ways:
  - Fanout: "give me all members of channel X" (by channel_id)
  - Permission check: "is user U in channel X?" (by user_id × channel_id)

#### `messages` (Cassandra — the hot table)
- Partition key: `channel_id`
- Clustering key: `(server_ts DESC, server_seq DESC)` — gives per-channel total order
- Wide row per channel; range-scan reads page through history
- Columns: `message_id`, `author_id`, `content`, `parent_message_id` (null for root), `edited_at`, `deleted_at`

```
CREATE TABLE messages (
  channel_id bigint,
  server_ts timestamp,
  server_seq bigint,
  message_id bigint,
  author_id bigint,
  content text,
  parent_message_id bigint,
  edited_at timestamp,
  deleted_at timestamp,
  PRIMARY KEY ((channel_id), server_ts, server_seq)
) WITH CLUSTERING ORDER BY (server_ts DESC, server_seq DESC);
```

- **Why partition by channel_id, not workspace_id?** Channels are the unit of read access (history scroll). Workspace as partition would create giant rows for big enterprises. But we co-locate channels of one workspace on the same Cassandra DC for cell isolation.
- **Hot partition risk:** a 100K-member channel posting 100/sec is a hot row. Mitigated below in §8.

#### `threads` — implicit, not a separate table
- A thread is just messages with `parent_message_id != null`.
- Per-thread read: `WHERE channel_id = X AND parent_message_id = Y` — secondary index.
- Avoid a fully separate "thread" partition unless threads are huge (>1000 replies).

#### `reactions` (Cassandra)
- (channel_id, message_id, emoji, user_id) — append-only, separate from message body
- Keeps message immutable; reactions evolve

#### `read_state` (per user, per channel — Redis, with PG fallback)
- (user_id, channel_id) → last_read_message_id, last_read_ts
- Used for unread badges and "mark as read" sync across devices

#### Search index (Elasticsearch)
- Per-workspace index (or per-tier-of-workspaces if 5M is too many indices)
- Document = message; fields: workspace_id, channel_id, author_id, content, ts, channel_visibility
- Indexed asynchronously via Kafka (search lag tolerable, hard durability isn't)

#### Presence (Redis)
- (user_id) → status, last_heartbeat_ts
- TTL of 60s; refreshed by heartbeat from gateway

### Why these choices

- **Cassandra for messages**: append-heavy, time-ordered, partition-by-channel maps perfectly to the read pattern (channel scroll). Tunable consistency. Linear scale to 10B/day. PG would buckle without significant manual sharding work.
- **PG for workspace/user/channel metadata**: low-volume, transactional needs (membership changes are rare and ACID matters), joins useful for admin tooling.
- **Redis for hot state** (presence, read-state, channel-fanout pub/sub): in-memory speed, pub/sub primitive, TTL evictions free.
- **Elasticsearch for search**: nothing else in the stack does inverted-index ranked search at this scale. Per-workspace shard isolates noisy neighbors.

### Message ordering — the hard part

Per-channel total order requires a **server-issued sequence**. Two clients can write at the same wall-clock time. Approach:

- The Messaging Service that owns a channel partition assigns `(server_ts, server_seq)` to every message. server_seq monotonically increases per channel.
- This requires either: (a) a single writer per channel (partition by channel, sticky to a node), or (b) a coordination primitive (Redis INCR per channel — costs a round trip).
- We pick (a): channel-affine writers. The Messaging Service is sharded by channel_id; messages for the same channel land on the same writer. server_seq is a local counter, persisted alongside the message.
- Failover: if the writer dies mid-flight, a backup takes over. Brief gap in seq is acceptable; we just never reuse a seq.

This is the kind of design call where you pre-empt the question: "I'm going with single-writer-per-channel because total order is required and the alternative — a global seq generator — adds 1ms RTT per message and a single point of failure. The tradeoff is that channel writers can be hotspots, mitigated by sub-channel sharding for very busy channels."

---

## 7. High-level architecture

```
                 ┌─────────────────────────────────────────────────────────────┐
                 │            CDN / Anycast Edge / Layer-7 LB                  │
                 │            (TLS termination, WS-aware, sticky)              │
                 └──────────────────┬──────────────────────┬───────────────────┘
                                    │ WebSocket            │ HTTP REST
                                    ▼                      ▼
                          ┌──────────────────┐   ┌──────────────────┐
                          │  Gateway tier    │   │  REST API tier   │
                          │ (stateful, WS)   │   │  (stateless)     │
                          │ shard by user_id │   │                  │
                          │ ~2000 hosts      │   │                  │
                          └────────┬─────────┘   └────────┬─────────┘
                                   │                       │
                                   │  ingest message       │ ingest message
                                   ▼                       ▼
                          ┌────────────────────────────────────────┐
                          │         Messaging Service              │
                          │  (stateless ingress; channel-affine    │
                          │   writers via shard router)            │
                          │  - assigns server_ts + server_seq      │
                          │  - persists to Cassandra               │
                          │  - emits to channel pub/sub bus        │
                          │  - emits to Kafka for downstream       │
                          └──┬───────────────┬─────────────┬───────┘
                             │               │             │
                             ▼               ▼             ▼
                  ┌──────────────┐  ┌─────────────┐  ┌────────────┐
                  │  Cassandra   │  │ Channel     │  │ Kafka      │
                  │  messages    │  │ pub/sub     │  │ topic:     │
                  │  (per-       │  │ (Redis or   │  │ messages.  │
                  │   channel    │  │  proprietary│  │ created    │
                  │   wide row)  │  │  mesh)      │  │            │
                  └──────────────┘  └──────┬──────┘  └─────┬──────┘
                                           │                │
                                           │ delivers to    │ consumed by
                                           │ subscribed     │ - Search Indexer
                                           │ Gateway hosts  │ - Notification Svc
                                           ▼                │ - Analytics
                                  ┌──────────────────┐      │
                                  │  Gateway tier    │      ▼
                                  │  (push frames    │   ┌────────────────┐
                                  │   to subscribed  │   │ Search Indexer │
                                  │   user sockets)  │   │  → Elasticsearch│
                                  └──────────────────┘   └────────────────┘

      ┌──────────────────────────────────────────────────────────────────┐
      │  Side services:                                                  │
      │   - Channel Service (membership, permissions; PG-backed)         │
      │   - Presence Service (Redis; heartbeat-driven)                   │
      │   - Notification Service (mobile/desktop; APNs/FCM out)          │
      │   - Object Storage (S3 — files, glossed)                         │
      └──────────────────────────────────────────────────────────────────┘
```

### Request walk-through

**Send + deliver path (the hot path):**

1. Client A has an open WebSocket to Gateway G1. Sends `{ type: MESSAGE_CREATE, channel_id: C, content, idempotency_key }`.
2. G1 forwards to Messaging Service via internal RPC. (It does NOT write directly — it's only a connection terminator.)
3. Messaging Service routes to the channel-affine writer for channel C (consistent hash on channel_id).
4. Writer:
   - Permission check: is user A a member of C? (Cache: 99% hit; miss → Channel Service.)
   - Assigns (server_ts, server_seq) — incrementing local counter.
   - Writes to Cassandra (RF=3, CL=QUORUM). Idempotency key dedupes if client retries.
   - On Cassandra ack: returns 201 to G1, which sends `{ type: MESSAGE_ACK, client_msg_id, server_message_id }` back to client A.
   - Asynchronously: publishes to channel pub/sub topic `channel:C` (Redis pub/sub or proprietary).
   - Asynchronously: emits to Kafka `messages.created` (for search indexer, notifications, analytics).
5. **Channel pub/sub**: every Gateway host that has at least one subscriber for channel C is subscribed to topic `channel:C`. The pub/sub layer fans the message out to all such hosts.
6. Each Gateway host receives the message once, then **fans out within the host** to all connected sockets that subscribe to channel C. (Per-process fanout is cheap; per-network-hop fanout is the expensive part.)
7. Each subscribed client receives `{ type: MESSAGE_NEW, ... }` and renders.

**Total latency budget:**
- Client → Gateway: ~5ms
- Gateway → Messaging Service: ~2ms
- Cassandra QUORUM write: ~10–30ms
- Pub/sub hop: ~5ms
- Gateway → Client: ~5ms
- **End-to-end p50: ~50–80ms; p99: 300–500ms.**

**Reconnect / catch-up path:**

1. Client reconnects after network blip. Sends WS handshake with `last_ack_message_id` per channel (or a single "global cursor" the server maintains).
2. Gateway forwards to Messaging Service. Messaging Service queries Cassandra for messages with `server_seq > last_ack` per channel the user is in.
3. Sends them as a `READY` frame, then subscribes the gateway socket to live channel topics.
4. Bounded by N channels × replay window. To prevent abuse, replay capped at N hours; longer = client must use REST history endpoint.

**Search path:**

1. Client `GET /v1/search?workspace_id=W&q=foo` → REST API tier → Search Service.
2. Search Service identifies workspace's Elasticsearch shard (by routing key = workspace_id).
3. Adds ACL filter: `channel_id IN (channels user is a member of)`.
4. Submits ES query, returns top N hits with snippets.
5. Hydrates message bodies from Cassandra (or just trusts ES copy for snippets, fetches authoritative body on click).

### Why this shape

- **Gateway is stateful, everything else is stateless.** The Gateway holds connections; if it dies, clients reconnect to a different one. Messaging Service, Search, etc., are stateless and trivially horizontally scaled.
- **Channel-affine writers** for ordering, with **channel-as-topic pub/sub** for fanout. This separates the write path (durable, ordered, slow-ish) from the deliver path (best-effort, fast, fan-out).
- **Kafka is the system of record for downstream consumers** (search, notifications, analytics). The pub/sub bus is for live delivery only — it's allowed to drop on partition (clients reconnect and pull from Cassandra).

---

## 8. Deep dives

### Deep dive 1: WebSocket gateway — the connection problem

> **Interviewer:** "20 million concurrent connections. Walk me through the gateway."

**Candidate:**

This is the operationally hardest part of the system. Let me lay it out.

**Sharding:**
- Gateway shards by `user_id` (hash). One user's connections (potentially several — phone, laptop) land on the same shard, simplifying per-user state (presence, read-state caches).
- ~2,000 gateway hosts, each holding ~10K connections. RAM-bound, not CPU-bound (~100KB / connection × 10K = 1GB plus overhead).
- Sticky sessions via consistent hashing of user_id. Edge LB does the hashing — clients hit any LB, get routed deterministically.

**Connection establishment:**
- Client first calls `GET /v1/gateway` over HTTPS — small, stateless, returns the actual WS URL with a routing hint.
- Client opens WS to that URL. Gateway accepts, validates session token, registers in connection table.
- **Gracefully handle TLS handshake load:** TLS resumption, session tickets — keep the cost of reconnects down.

**Subscribing to channels:**
- After connect, gateway loads the user's channel membership (from Channel Service cache).
- Gateway subscribes to pub/sub topics for each of those channels. With 100 channels per user × 20M users = 2B subscriptions in aggregate, but per-host it's ~1M subscriptions (10K users × ~100 channels). Manageable.
- We use **channel-as-topic** semantics: subscribe to `channel:C`, get every message for that channel. Compare to the alternative — per-user inbox topic — which scales worse for big channels (the publisher would have to enumerate all subscribers and write to N inboxes).

**Heartbeat / liveness:**
- Client sends PING every 30s. Server replies PONG.
- Server tracks last-PING; if >90s, considers connection dead, closes it, removes presence.
- TCP keepalive as backstop, but don't rely on it (TCP keepalives are minutes by default).

**The reconnect storm — gateway crashes:**
- A gateway host dies. 10K connections drop simultaneously.
- Clients see WS close. **Without backoff**, they all reconnect within seconds — a thundering herd.
- Mitigation: clients implement **jittered exponential backoff** (1s + random(0–1s), then 2s + jitter, etc.). Capped at 30s.
- Edge LB's consistent hashing reroutes the affected user_ids to other gateways. Other gateways see a 1% bump in connections — absorbable if we're sized for 30% headroom.
- **Operational rule:** never let total gateway utilization go above 70% — leave 30% for failover.

**Graceful drain (for deploys):**
- Old gateway version → server sends `{ type: RECONNECT, reason: "drain" }` to each connected client, with a small jitter (0–60s) so clients reconnect spread over a minute.
- Client closes the connection after rendering pending messages, opens a new one — lands on either the same or a new gateway via consistent hashing.
- Drain takes 5–10 min per host; rolling deploy across 2,000 hosts at 2% concurrency = ~10 hours for full fleet rollover. Acceptable for non-emergency deploys.

> **Interviewer:** "What if a single gateway shard becomes hot — say one user has 1000 channels they're in?"

**Candidate:**
- Per-host limit: degraded experience for that user (some channels deliver slow), but not other users on the same host.
- Hard cap on max channel subscriptions per user (Slack actually does this — ~1000-channel limit).
- Could shard the gateway differently for power users — but the operational complexity isn't worth it.

### Deep dive 2: Channel fanout — pub/sub architecture

> **Interviewer:** "How does a message in #general (50K members) actually get to all 50K clients?"

**Candidate:**

Two layers:

**Layer 1: Inter-host fanout via pub/sub.**
- The Messaging Service writes the message to Cassandra, then publishes to `channel:C`.
- Pub/sub backend: Redis pub/sub for early-stage, **proprietary mesh** at scale (Slack built one called Edgar; Discord built theirs; the principle is the same).
- All gateway hosts that have ≥1 subscriber to channel C get the message exactly once.
- For #general with 50K members spread across 2,000 gateway hosts: roughly all 2,000 hosts receive one copy. So the pub/sub publish fans out to ~2,000 hosts.

**Layer 2: Intra-host fanout.**
- Each receiving gateway host has, on avg, 25 sockets subscribed to channel C (50K / 2000).
- The host's per-channel subscriber list is held in memory; on receipt, iterate and write to each socket.
- Per-socket write is ~1KB serialization + 1 network frame. At 50K subscribers spread over 2K hosts → 25 writes/host = trivial.

**Why this scales better than per-user inbox model:**
- Alternative: publisher writes to per-user inbox topics. For 50K-member channel = 50K writes. Doesn't scale to large channels.
- Channel-as-topic = N writes where N = number of gateway hosts with a subscriber, which is bounded by the gateway fleet size (2K), not the channel size (50K, or 100K).
- This is **the central fanout decision** for large channels.

**Rate limiting on fanout:**
- Per-channel publish rate cap (e.g., 100 messages/sec per channel). Beyond that, queue up — visible as small delivery lag.
- Prevents a runaway bot from melting #general for 100K users.

**The 100K-member channel:**
- Same approach scales: 2K hosts each get one copy, each host has ~50 local subscribers. Still manageable.
- The throughput cost is in the Cassandra hot partition (see below), not the fanout itself.

> **Interviewer:** "What if Redis pub/sub partitions — split-brain?"

**Candidate:**
- Redis pub/sub is best-effort. If a partition occurs, some subscribers don't get the message.
- **Self-healing via Cassandra:** clients periodically reconcile via REST `GET /messages?since=...` if they detect missing seqs (server_seq has a gap → fetch the gap).
- This is also the recovery mechanism for "client was offline; what did I miss."
- For stronger guarantees we'd use Kafka as the bus (persistent, ordered) — but Kafka publish latency is ~10–30ms vs Redis pub/sub's <1ms, and we need delivery to feel instant. **Tradeoff: speed vs delivery guarantee.** We pick speed + reconciliation.

### Deep dive 3: Message ordering and the channel-affine writer

> **Interviewer:** "How do you guarantee per-channel total order?"

**Candidate:**

The hard part: two clients post to the same channel at the same wall-clock millisecond. We need a single agreed order.

**Approach: channel-affine writer.**
- Messaging Service is sharded by `channel_id`. Each shard is a single primary writer (with backups for failover).
- The writer holds an in-memory per-channel sequence counter, persisted to Cassandra alongside the message.
- All writes to channel C funnel through one writer at a time, so server_seq is monotonic.

**Failover:**
- Writer dies. Backup takes over. Reads max(server_seq) from Cassandra for channel C, resumes counter from there + 1.
- Brief unavailability for that channel's writes (say 5–10s during failover detection).
- During failover, in-flight writes that didn't ack get retried by the Gateway → land on the new writer. Idempotency key dedupes.

**Why not a single global sequencer?**
- Single point of failure, hot.
- We don't need global order, just per-channel.

**Why not just use timestamp + tiebreaker (e.g., (ts, message_uuid))?**
- Clock skew means two writers could disagree on the order of two near-simultaneous messages.
- Workable, but the user-visible bug is "Alice's reply appears before the question she's replying to" if their writers' clocks are skewed.
- Server_seq from a single writer eliminates this.

**Cassandra hot partition (busy channel):**
- A 100K-member channel posting 100/sec is a hot partition.
- Mitigations:
  - **Sub-partition scheme**: instead of partition key = channel_id, use (channel_id, ts_bucket) where ts_bucket = floor(ts / 1hr). Reads now range across recent buckets but writes spread.
  - **Cassandra row-cache** for the most recent N messages per channel.
  - At extreme scale (Discord-sized public servers with 1000+ msg/sec sustained), shift to a different store (Discord moved to ScyllaDB + custom routing).

**Threads:**
- Threads don't need their own ordering scheme — they inherit the channel's. parent_message_id is just a foreign-key-like reference.
- Thread reads are: query messages where parent_message_id = P, ordered by server_seq.
- For very large threads (>1K replies), consider a separate sub-partition.

### Deep dive 4: Search — Elasticsearch with workspace ACLs

> **Interviewer:** "How does search work, and how do you keep it from leaking private channel content?"

**Candidate:**

Async pipeline via Kafka:

1. Messaging Service emits `messages.created` to Kafka (already in our diagram for fanout).
2. Search Indexer (Kafka consumer) reads the event, indexes the message into Elasticsearch.
3. Indexing lag: typically 1–5s p99. Visible to users as "search may take a moment to find your latest message."

**Index sharding:**
- Per-workspace index (or per-tier-of-workspaces if 5M is too many indices to manage — group small workspaces, dedicated index for large ones).
- Within an index, ES sharded by hash; we can route queries with a `routing` parameter (= workspace_id) so all docs in one workspace hit one shard.

**ACL enforcement (the crux):**
- Doc fields include `channel_id` and `channel_visibility` (public / private / dm).
- Query-time filter:
  - For public channels: anyone in the workspace can see → filter `workspace_id = W AND channel_visibility = public`
  - For private channels: only members → fetch user's private-channel-membership list (cache it), filter `channel_id IN (member-list)` for private.
- **Why filter at query time, not at index time?** Memberships change. Re-indexing on every membership change is expensive. Filter at query time, accept the latency cost.
- Common cache: per-user "channels I have access to" list, cached for 60s; refresh on membership change events.

**Permission revocation:**
- User removed from private channel. They search → ACL filter no longer includes that channel. Past results disappear.
- Cache invalidation on membership change events (Kafka topic `memberships.changed`).

**Storage / retention:**
- Elasticsearch storage is expensive. Limit free-tier workspaces to last 90 days of search history (Slack-style); paid tiers get longer or unlimited.
- Cold archive: messages older than retention indexed in a cheaper "cold search" index (separate cluster with ColdHDD storage).

**Search query latency:**
- p50 < 300ms, p95 < 1s.
- Caching: query result cache for repeated queries (TTL 60s).

**Capacity:**
- 10B msg/day × 1KB indexed = 10TB/day raw. Compressed in ES: ~3–5TB/day index growth.
- Per-year: ~1.5 PB indexed. Hot tier (last 90 days): ~450 TB. Manageable across a fleet.

> **Interviewer:** "What if ES indexing falls behind?"

**Candidate:**
- Lag is observable: Kafka consumer lag on the indexer.
- Alert at >30s lag. User-visible as "search doesn't have the message you just sent yet."
- Backpressure: scale indexer worker pool. ES side: more nodes if write throughput maxed.
- Worst case (multi-hour lag): users live with stale search; it's a graceful degradation, not an outage.

### Deep dive 5: Presence — the cheap-but-not-cheap problem

> **Interviewer:** "How do you compute online presence for 20M concurrent users?"

**Candidate:**

Two models:

**Naive: Redis + heartbeat per user.**
- Each gateway pings `SET presence:user_id status EX 60` every heartbeat.
- 20M users × 1 write / 30s = 660K writes/sec to Redis. Doable but expensive.
- Reads: when one user views another's profile, GET presence:user_id.

**Smarter: Probabilistic / sampled presence.**
- For DMs, accurate presence matters. For 100K-member channel sidebar, "who's online" can be approximate.
- Sample-and-aggregate: gateway emits "X out of Y users in channel C heartbeated in last 60s" to a per-channel presence summary. Updates every 30s.
- Reduces fanout: no per-user presence read for the 99% of times the answer is "doesn't matter."

**Push presence diffs, not full state:**
- When user goes online → emit `PRESENCE_UPDATE` event to that user's channel topics. Other users' clients update locally.
- When user goes offline → after 60s heartbeat timeout, emit a single offline event.
- Avoid 20M × 200 channels = 4B presence subscriptions globally. Slack handles this with explicit "subscribe to presence for these N user_ids" — only when you can see them on screen.

**Client-side optimization:**
- Don't send presence for users not visible. Only when sidebar / DM list / member modal opens.
- Drives "presence is calculated lazily."

This is one of those features where naive scales until ~1M concurrent and then gets expensive. The right move is "model presence as best-effort, sampled, push-when-changed, not pull-when-asked."

### Deep dive 6: Notifications

> **Interviewer:** "If I'm offline, how do I get a push notification on my phone?"

**Candidate:**

Flow:
1. Messaging Service emits to Kafka `messages.created`.
2. Notification Service consumes the event.
3. For each recipient (channel members):
   - Check connection state: are they currently connected via WS? (Check presence cache.)
   - If yes and the message was delivered live → no push.
   - If no, OR the user has "always notify" rules, OR user is mentioned → fire push.
4. Push routing:
   - Mobile → APNs (iOS) / FCM (Android) with the user's device tokens.
   - Desktop (when app backgrounded) → web-push or native APNs/FCM.
5. Badge counts: Notification Service maintains per-user unread count; pushes badge update with each push.

**Mention extraction:**
- Done at message-create time (Messaging Service parses content for `@user`, `@channel`, `@here`).
- Mention list goes into the message metadata; Notification Service uses it to scope per-user behavior (mentioned users always get push).

**Rate limiting:**
- Per-user push cap (e.g., 100/day) to prevent notification storm in a busy channel.
- Coalesce: "5 new messages in #general from 3 people" — bundle, don't send 5 separate pushes.

**APNs/FCM are unreliable:**
- Treat as best-effort — they don't ack durably.
- Fallback: in-app inbox shows missed messages on next open, regardless of push delivery.

---

## 9. Tradeoffs & alternatives

### Decision: Channel-as-topic pub/sub vs per-user inbox

| Option | Pros | Cons | When to pick |
|---|---|---|---|
| **✅ Channel-as-topic (Discord-style)** | Scales to giant channels — fanout cost = O(gateway hosts) not O(channel size) | Pub/sub is best-effort; needs reconciliation for missed events | Public communities, large channels |
| **Per-user inbox (Slack-style historic)** | Each user has a durable inbox queue; deliveries are individually tracked | Doesn't scale: 100K-member channel = 100K inbox writes per message | Small workspaces, where per-user audit matters |
| **Hybrid** | Per-user inbox for DMs, channel-as-topic for big channels | Two code paths | Real-world systems |

I picked channel-as-topic for the channel path, with a per-user notification queue for **mentions and DMs** (where durability matters more than fanout efficiency).

### Decision: Cassandra for messages

| Option | Pros | Cons | When to pick |
|---|---|---|---|
| **PostgreSQL (sharded)** | Familiar, transactions | Manual sharding pain at 10B/day | Smaller scale (<1B msg/day) |
| **DynamoDB** | Managed, scales | Vendor lock-in, costly at 10B/day, secondary indexes weak | AWS-native, ops-thin team |
| **MongoDB** | Schema flexibility | Write throughput weaker than Cassandra; rebalancing pain | Document-heavy use cases |
| **✅ Cassandra / ScyllaDB** | Wide-row optimal for channel append, linear scale, tunable consistency | Operational complexity | Wide-row write-heavy workloads at hyperscale |
| **Kafka as primary store** | Already on critical path | Not a query store; awkward for history reads | Not a fit for this read pattern |

Discord publicly migrated from Cassandra to ScyllaDB at scale — same data model, better latency/throughput. I'd start with Cassandra and migrate if hot-partition latency becomes the limit.

### Decision: Redis pub/sub vs Kafka for live delivery

| Option | Pros | Cons | When to pick |
|---|---|---|---|
| **✅ Redis pub/sub (or proprietary mesh)** | <1ms latency, simple, in-memory | Best-effort, no replay, partitions can drop messages | Live-delivery, low-latency required |
| **Kafka** | Persistent, ordered, replayable | 10–30ms publish latency; overkill for live | Durable bus for downstream consumers (where we DO use it) |
| **Direct service-to-service RPC** | Minimal infra | Doesn't fan out across gateway hosts; coupling | Small scale only |

Use Redis pub/sub for live, Kafka for downstream durability. Two buses, two purposes.

### Decision: WebSocket vs SSE vs Long-polling

| Option | Pros | Cons | When |
|---|---|---|---|
| **Polling** | Trivially simple | Wasteful, slow, scales poorly | Don't |
| **Long-polling** | HTTP-friendly, works through proxies | Server-side connection cost, latency penalty | Fallback for WS-blocked networks |
| **SSE (Server-Sent Events)** | Server → client one-way, HTTP | One-way only; client → server uses separate POST | Read-only feeds |
| **✅ WebSockets** | Bidirectional, low-overhead, mature | Sticky sessions, harder LB, some networks block | This use case |

We default to WebSockets, fall back to long-polling for the 1–2% of clients on networks blocking WS.

### Decision: Workspace-as-cell (cellular)

Slack Enterprise Grid is exactly this: each enterprise customer has a dedicated stack. Cellular by tenant, not by region.

| Option | Pros | Cons |
|---|---|---|
| Single global multi-tenant stack | Cheaper infra, simple deploys | Blast radius = 100% of customers; data residency painful |
| Per-region cells | Latency wins; residency easier | Cross-region workspaces complex |
| **✅ Per-workspace cells (for enterprise)** | Strong isolation; per-tenant capacity; per-tenant compliance | Operational overhead — cells per customer, not per region |

For consumer / small-business tier: shared multi-tenant cells per region.
For enterprise tier: dedicated cells per customer (Slack Enterprise Grid model).
Cross-cell DMs (Slack Connect) are the cross-cell case — see §19.

### Consistency model summary

| Scenario | Consistency | Mechanism |
|---|---|---|
| Messages within a channel | Per-channel total order | Channel-affine writer + server_seq |
| Read-your-writes (sender sees own message) | Yes | Optimistic UI + server ACK |
| User sees message they sent on second device | Eventual (~1s) | Pub/sub fans out to all of user's connections |
| Message edits / reactions | Eventual | Separate event stream merged at client |
| Search results include latest messages | Eventual (1–5s lag) | Async indexing |
| Permission changes (user removed from channel) | Eventual (~30s) | Membership cache TTL |
| Workspace isolation | Strong | Hard partition by workspace_id at every layer |

---

## 10. Defending architectural choices — interviewer attacks

> **Interviewer:** "Why two buses — Kafka AND Redis pub/sub? Isn't that overkill?"

**Candidate:** Different SLAs. Kafka guarantees delivery + replay for downstream consumers (search indexer, notifications, analytics) where durability beats latency — losing a message means it's missing from search forever. Redis pub/sub is best-effort and <1ms — for the live-delivery path where latency dominates and we have a reconciliation backstop (clients detect server_seq gaps and pull). One bus can't be both: Kafka's 10–30ms publish latency would push end-to-end deliver above our 500ms SLO, and Redis pub/sub's no-replay would lose search updates. Two buses, each fit-for-purpose.

> **Interviewer:** "Why partition messages by channel_id, not workspace_id?"

**Candidate:** Channels are the unit of read access. Users scroll history one channel at a time. Partition by workspace_id and a 100K-user enterprise's `messages` becomes a single giant row spanning every channel — read amplification on history reads, hot-partition risk on writes. Channel partitioning matches the access pattern. Workspace co-location happens at the **deployment** layer (a workspace's channels live on the same Cassandra DC for cell isolation), not at the partition layer.

> **Interviewer:** "What if a single gateway host crashes? 10K users disconnect."

**Candidate:** Detection (~30s via heartbeat), blast radius (10K users — 0.05% of fleet), automatic mitigation (consistent hashing reroutes user_ids to other gateways, clients reconnect with jittered backoff, presence updates eventually), data loss (zero — gateway is stateless re: messages; messages live in Cassandra), recovery (1–2 min for 10K connections to reconnect). The reconnect storm risk is mitigated by client backoff + the 30% headroom we keep on remaining gateways. We DON'T page on this — it's expected behavior. We page if 10+ gateways die simultaneously (fleet-level event).

> **Interviewer:** "Why not store messages in Kafka and skip Cassandra?"

**Candidate:** Kafka isn't a query store. History reads — "show me #general from yesterday" — need ranged scans by channel+time, which Kafka can do (offset by partition + time index) but ergonomically and cost-wise, a wide-column store is purpose-built. Kafka also has retention pressure: 10B messages/day × infinite retention is a different shape than Kafka was designed for. Cassandra-as-primary, Kafka-as-bus is the standard split.

> **Interviewer:** "Why server-issued sequence vs client timestamp?"

**Candidate:** Two writers' clocks can be skewed by milliseconds — and at 350K msgs/sec, two messages within the same millisecond are common. Without a server-side total order, messages in the same channel can shuffle on the receiver. The user-visible bug is a reply appearing before the question. Client timestamp + server tiebreak is workable but more complex; channel-affine writer with a local counter is simpler and the failure mode (brief unavailability during writer failover) is acceptable.

> **Interviewer:** "How do you handle a viral channel — 100K members, 100/sec posts?"

**Candidate:** Three pressures: (1) Cassandra partition becomes hot → mitigation: sub-partition by hour bucket; (2) Channel pub/sub topic has 2K gateway subscribers → fine, gateway count is the bound, not channel size; (3) Per-host fanout — each gateway pushes to ~50 sockets per message, 100/sec × 50 = 5K writes/sec per host, well within capacity. Rate limit at 200 messages/sec per channel as a hard cap to prevent runaway bots.

> **Interviewer:** "What if Cassandra goes down?"

**Candidate:** Multi-region replication: messages durably replicated to ≥2 regions. If primary DC fails, traffic shifts to replica (read-only at first; writer election happens within seconds). User-visible: brief write unavailability (10–30s) for affected channels, then writes resume in the failover region. We never serve stale reads on history; if Cassandra is fully partitioned, the service degrades to "live messages work but history scroll fails." It's unusual to lose Cassandra entirely; far more common is a single shard. RF=3 means we lose one replica and continue at QUORUM.

---

## 11. Failure modes & mitigations

| Failure | Detection | Blast radius | Mitigation | Recovery |
|---|---|---|---|---|
| Gateway host crash | Health check / connection drop alarm | ~10K users disconnect, reconnect ≤2 min | Consistent-hash reroute + jittered client backoff + 30% headroom | Auto-replace via ASG; on-call only if multiple |
| Reconnect storm (mass gateway redeploy) | Spike in connection rate | Whole fleet | Stagger graceful drains over 60-min window with jittered RECONNECT frames | N/A (planned) |
| Messaging Service writer (channel) crash | Per-channel write lag alarm | All writes for that channel paused 5–10s | Backup takes over; reads max(seq) from Cassandra, resumes | Auto-failover; idempotency keys dedupe in-flight writes |
| Cassandra hot partition (busy channel) | Per-partition write latency p99 spike | Slow writes for that channel; risk of cascading lag | Sub-partition by ts_bucket; row cache hot recent messages | Re-partition the channel offline if persistent |
| Cassandra node down | Per-shard read/write error rate | 1/N channels degraded reads | RF=3, QUORUM tolerates 1 down; writes go to remaining 2 + hint | Replace node, stream-rebuild |
| Redis pub/sub partition | Pub/sub subscriber count drops; missed-message reports | Some live deliveries dropped | Clients detect seq gap via REST `since=`, fetch and reconcile | Redis cluster repair; we never relied solely on pub/sub |
| Elasticsearch indexer lag | Kafka consumer lag on `messages.created` | Search results stale | Auto-scale indexer workers; ES side scale write capacity | Catch up; user-visible only as "search may not have your latest message" |
| Elasticsearch shard down | Per-shard query error rate | 1/N workspaces' search degraded | Replica shard takes over; query routing avoids dead shard | ES self-heal; on-call investigates if persistent |
| Notification Service backlog | Kafka consumer lag on notifications | Push notifications delayed | Auto-scale; coalesce more aggressively | Drain backlog |
| Presence Redis failure | Heartbeat write error rate | "Online" indicators wrong / missing | Show last-known status; degrade to "unknown" | Restore Redis cluster; presence is best-effort anyway |
| Workspace cross-talk (worst case) | Audit | Privacy breach — workspace A sees workspace B | Hard partition: every query keyed by workspace_id; integration tests guard | Page security on detection; postmortem mandatory |
| Gateway version mismatch (mid-deploy) | Client reports protocol error | Some users disconnected during deploy | Forward+backward compatible WS protocol; version negotiate at handshake | N/A — design for it |
| Schema change in messages | Pre-deploy | Service crash on deploy | Additive-only schema; new fields nullable; stagger rollout | Rollback if needed |
| Massive enterprise onboarding (10K users join in a minute) | Connection rate spike | Gateway saturation | Pre-provisioning capacity planned with sales; per-workspace rate limit on connect rate | Manual scale-up if missed |

---

## 12. Productionization

### Pre-launch
- [ ] Load test gateway tier to 2× projected concurrent connections. Verify reconnect storm behavior (kill 10% of hosts, watch recovery).
- [ ] Soak test message ingest at 5× peak QPS for 4 hours. Watch Cassandra latency drift.
- [ ] Chaos test: kill random gateway hosts, kill Redis pub/sub nodes, kill Cassandra nodes. Verify graceful degradation.
- [ ] Game day: simulate full Cassandra cluster failover. Time the recovery.
- [ ] Permission audit: synthetic test attempts cross-workspace queries — must all return 403.

### Rollout
- [ ] Workspace-level feature flags. Roll out to internal workspaces first (dogfooding).
- [ ] Gradual: 1% → 10% → 50% → 100% **of workspaces**, not of users. We A/B at workspace level so users in the same workspace share an experience.
- [ ] Per-region rollout: smallest market first. If we have an EU residency cell, start there since it's smaller volume.
- [ ] Auto-rollback if: send-to-deliver p99 > 1s for 10 min, or 5xx rate > 1%, or Kafka indexer lag > 10 min.

### Capacity
- [ ] Reserve 50% headroom on gateway tier (connection failover demands it).
- [ ] Cassandra: provision 3× current write volume — adding capacity is a multi-day operation.
- [ ] Kafka: 5× current partition count — repartitioning is expensive; over-provision early.
- [ ] Auto-scale Messaging Service on QPS, target 60% CPU. Gateway scaled on connection count, not CPU.

### Migration (if replacing existing system)
- [ ] Dual-write to old + new for 4 weeks. Sample 0.1% of traffic for diff-checking.
- [ ] Read traffic cutover by workspace cohort (start with internal workspaces).
- [ ] Old system stays warm for 4 weeks of rollback insurance.
- [ ] Decommission old after 8 weeks of clean operation.

### Schema changes
- [ ] **Months long.** Message format changes touch 5+ services and the wire protocol.
- [ ] Two-phase: deploy code that READS new format → wait full week → deploy code that WRITES new format. Bake old + new in parallel.
- [ ] Compatibility tests in CI: old client + new server, new client + old server.

### Cost
- $/DAU rough order: $0.05–0.15/DAU/month for messaging + storage + search. Egress for live delivery and Elasticsearch storage are top line items.
- Cost levers:
  - Limit free-tier search history (90 days)
  - Tier old messages to cold storage (S3 Glacier-class) after 1 year
  - Aggressive Redis eviction for inactive users' presence
  - Connection density per gateway host — engineering effort here pays back linearly

### Compliance
- GDPR: user deletion → tombstone in `messages` (anonymize author_id, delete content); search index re-indexed without the deleted content. Async sweep.
- Data residency: workspace `home_region` field drives cell pinning. Hard requirement, not soft preference.
- E-discovery / legal hold: separate read API for compliance admins; audited.

---

## 13. Monitoring & metrics

### Business KPIs
- DAU, weekly retention
- Messages sent per DAU per day
- Concurrent connections (peak)
- Workspace size distribution
- Search QPS as % of session

### Service-level (SLIs)

- **Send-to-deliver end-to-end** (sender hits send → receiver client renders): p50, p95, p99, p99.9. THE headline metric.
- **Connection establish latency** (WS handshake → READY frame): p99 < 1s.
- **Catch-up latency on reconnect**: time for client with N missed messages to be live.
- **Search query latency**: p95.
- **Indexing freshness**: time from message_create → searchable.
- **SLO**: 99.95% of sends < 500ms; 99% of searches < 1s; 99.9% of WS connects < 1s.

### Component metrics

#### Gateway
- Connection count per host (alarm at >12K — running hot)
- Connection accept rate (storms manifest here)
- Per-channel subscription count distribution (detect users with too many channels)
- Heartbeat-driven disconnects per minute (network blips)
- Pub/sub consumer rate (hint at throughput)
- Per-host RAM (connection density)

#### Messaging Service
- Write throughput per channel (detect hot channels)
- Server_seq gap rate (failover events)
- Idempotency dedup hit rate (client retry rate)
- Permission-check cache hit rate (target ≥99%)

#### Cassandra
- Per-shard read/write latency p50/p99
- Hot partition detection (per-key write rate)
- Compaction queue depth
- Replica lag cross-region
- Hint queue (during failover)

#### Channel pub/sub (Redis or proprietary)
- Subscribers per topic (detect mega-channels)
- Publish rate
- Drop rate (Redis cluster failover symptoms)
- Memory utilization

#### Kafka
- Producer rate per topic
- Consumer lag per group (alert >30s for indexer, >5min for analytics)
- Partition skew (hot channels manifest as hot partitions)

#### Elasticsearch
- Query latency p50/p99 per index
- Indexing throughput
- Indexing lag
- Per-index storage size + growth rate
- Shard rebalancing events

#### Presence
- Redis QPS for presence operations
- Heartbeat completion rate
- Presence sub fanout (subscribers per online-user channel)

### Alert routing
- **Page** (24/7): SLO breach (multi-window burn rate); region-level outage; Cassandra cluster degraded; gateway fleet utilization >85%
- **Ticket** (next business day): single-host issues, indexer lag <15min, cache hit rate degraded
- **Dashboard-only**: cost trends, capacity headroom, business KPIs

### Dashboards
1. **Gateway health** — connection count, accept rate, heartbeat losses, fleet headroom
2. **Send pipeline** — Send → Cassandra → pub/sub → deliver, end-to-end timing per percentile
3. **Workspace health** — per-cell, top workspaces by message rate
4. **Cassandra** — per-shard, per-keyspace
5. **Search** — query latency, indexer lag, ES cluster health
6. **Business** — DAU, messages/day, sessions/DAU

---

## 14. Security & privacy

- **Workspace isolation** — every query, every cache key, every Kafka message keyed by workspace_id. Integration tests verify cross-workspace queries return 403. The single most important invariant in the system.
- **Channel ACLs** — public channel: any workspace member; private: only members. Enforced at:
  - Send time (Messaging Service permission check)
  - Read time (history endpoint)
  - Search time (query-time ACL filter)
  - Presence (channel sidebar shows only members)
- **DM privacy** — DMs are private channels with exactly 2 (or N for group DM) members. No admin readback by default. (Enterprise compliance admins may have legal-hold view; logged.)
- **Data residency** — workspace_id → home_region binding is hard. Region-scoped Cassandra. Region-scoped ES. Cross-region only via explicit Slack-Connect-style federation.
- **Audit logging** — every admin action (user removed from channel, workspace settings changed, message deleted by admin) logged immutably.
- **Encryption** — TLS in transit (WS over WSS); at-rest encryption for Cassandra, ES, S3.
- **Bot/integration tokens** — scoped: read:channel:X, write:channel:Y. Validated at API layer.
- **Spam / abuse** — per-user message rate limit (~10 msg/sec hard cap); auto-mute on flood; admin tools for workspace-level moderation.
- **Account takeover** — anomaly detection on session geography / device fingerprint; force re-auth on suspicious activity.
- **Compliance** — SOC2 Type II, HIPAA (paid tiers), GDPR, CCPA. Data deletion path hooks into user lifecycle.

---

## 15. Cost analysis

| Component | Rough cost driver | Optimization lever |
|---|---|---|
| Gateway compute | Connection count × 24/7 uptime | Connection density per host; reserved instances for baseline; spot for burst |
| Cassandra (messages) | Storage × replication × retention | Tiered: hot (last 90 days SSD) + cold (1+yr cheap HDD); compress |
| Elasticsearch (search) | Storage + memory | Limit free-tier retention; cold-index for old data; per-workspace sharding lets us tier by workspace |
| Redis (pub/sub + presence) | Memory + ops/s | Aggressive eviction; sample presence; not durable so no replication cost |
| Kafka | Disk × retention | Short retention (7d enough for replay); compression |
| Egress (clients) | Bytes delivered to clients | Compress WS frames (per-message-deflate); coalesce high-frequency events (typing, presence) |
| APNs/FCM push | Per-push cost | Coalesce; rate limit per user |

Largest costs typically: gateway compute (always-on connection cost) and Elasticsearch storage. Egress to mobile clients is significant but manageable with compression.

$/DAU rough order: **$0.05–0.15/DAU/month** at the scale modeled. Enterprise Grid customers (dedicated cells) cost 3–5× per-DAU but charged accordingly.

---

## 16. Open questions / what I'd validate with PM

1. **Search retention by tier** — what's the free vs paid retention cliff? Drives ES storage cost.
2. **Largest-channel SLO** — what's the worst-case channel size we promise to support? Drives fanout architecture choices.
3. **Compliance** — is e-discovery a P0 (means a separate immutable archive) or P1?
4. **Slack Connect / federation** — cross-workspace DMs. Drives the cross-cell architecture (§19).
5. **Mobile background delivery SLA** — push notification freshness target on mobile vs desktop. Affects notification service design.
6. **Voice/video roadmap** — if voice is coming, the connection layer should anticipate WebRTC signaling.
7. **Bot/integration scale** — how many bot messages per workspace? Could dwarf human messages.
8. **Edits/delete retention** — do edited messages keep a history trail (compliance) or replace in place?

---

## 17. Staff-level scorecard for this problem

| Signal | Did the candidate... |
|---|---|
| ✅ Identified the connection problem as the operational dominant | Within first 10 min, before any other deep dive |
| ✅ Picked channel-as-topic over per-user inbox with explicit reasoning | Quantified: 100K-member channel = O(2K hosts) vs O(100K writes) |
| ✅ Quantified concurrent-connection cost | "20M connections / 10K per host = 2K hosts; sized at 30% headroom" |
| ✅ Volunteered reconnect storm | Discussed jittered backoff + consistent hashing reroute |
| ✅ Explained per-channel total order via channel-affine writer | Defended single-writer-per-channel, named failover semantics |
| ✅ Designed search ACL filter at query time, not index time | Explained the cache-invalidation tradeoff |
| ✅ Treated workspace as a first-class isolation boundary | Mentioned workspace-as-cell early, in scoping |
| ✅ Volunteered failure modes before being asked | Reconnect storm, Cassandra hot partition, ES indexer lag |
| ✅ Talked about cost concretely | Per-DAU number; named gateway compute and ES storage as drivers |
| ✅ Discussed productionization | Workspace-level rollout, schema changes are months-long |
| ✅ Closed with a wrap-up speech | Restated dominant constraints, named what they'd worry about |

### What separates Staff from Senior on *this* problem

A Senior candidate will:
- Draw the gateway + messaging + persistence layout
- Pick Cassandra for messages
- Mention WebSockets

A Staff candidate will additionally:
- Identify channel-as-topic vs per-user inbox as the central fanout decision and pick the right one for the workload
- Articulate the channel-affine writer + server_seq design for total order
- Quantify the concurrent-connection cost AND the reconnect-storm risk
- Distinguish two buses (Redis pub/sub for live, Kafka for downstream) with explicit rationale
- Treat workspace isolation as the strongest invariant and mention how it's enforced at every layer
- Discuss workspace-as-cell — and Slack Connect as the cross-cell case
- Anticipate that schema changes touching message format are multi-month
- A/B test at workspace, not user, level
- Volunteer presence as a "best-effort, sampled, push-when-changed" feature and not naive heartbeats

---

## 18. Wrap-up speech (template)

> "To wrap up: the dominant constraints here are (a) 20 million concurrent WebSocket connections — that's a connection-management problem more than a throughput problem — and (b) channel fanout to mega-channels with 100K members. The architecture answers both: a stateful Gateway tier sharded by user_id with consistent hashing for rerouting; channel-as-topic pub/sub via Redis (or proprietary mesh) for live delivery, where fanout cost is O(gateway hosts) not O(channel size); Cassandra partitioned by channel_id for the durable message store, with a channel-affine writer for per-channel total order; Elasticsearch async-indexed via Kafka for search, with query-time ACL filtering. The two pieces I'd worry about in production are (1) the reconnect storm when a gateway host dies — mitigated by jittered exponential backoff, consistent-hashing reroute, and 30% fleet headroom — and (2) Cassandra hot partitions for the busiest channels — mitigated by sub-partitioning by hour bucket and rate limits per channel. Workspace is the unit of isolation: every query keyed by workspace_id, and at the architectural level workspaces are pinned to a region for compliance. To productionize: workspace-level feature flags (we A/B at workspace, not user, granularity), staged rollout with auto-rollback on send-to-deliver p99 breach, and a four-week dual-write migration if replacing an existing system. If we had another 30 minutes, I'd deep-dive (a) the Slack-Connect / cross-cell federation problem, or (b) the Elasticsearch capacity model and ACL-cache design."

---

## 19. Workspace-as-cell isolation (cellular architecture) — the natural fit

> Bring this up in the last 5 minutes, or earlier if data residency comes up. For workspace messaging this isn't a bonus topic — it's the most natural cell boundary in the entire HLD library, and Slack Enterprise Grid is a real-world example.

### Why workspace, not region

For most products, the cell axis is geographic — US cell, EU cell, IN cell. For workspace messaging, the natural cell is the **workspace** itself (or a group of workspaces sharing a tier). Reasons:

1. **Compliance** — large enterprises require dedicated infra for SOC2/HIPAA/FedRAMP audits. They don't share a Cassandra cluster with anyone else.
2. **Blast radius** — a config bug rolled to one cell affects 10K workspaces, not 5M.
3. **Per-customer SLA** — Enterprise customers pay for and demand 99.99% uptime; freemium gets 99.9%. Different cells = different operational rigor.
4. **Capacity isolation** — Workspace W spawns a viral channel; doesn't degrade everyone else.
5. **Independent rollouts** — Customer X requires that we not deploy on Black Friday; we can hold their cell back.
6. **Cost attribution** — per-cell P&L. Enterprise customer pays $X/seat; we know exactly what their cell costs.

### Architecture

```
            ┌────────────────── Global Edge ───────────────────┐
            │ User → Workspace Directory → routes to home cell │
            └─────────┬─────────────┬─────────────┬────────────┘
                      │             │             │
            ┌─────────▼───────┐ ┌──▼──────────┐ ┌─▼────────────────┐
            │ Shared Cell (US) │ │ Shared (EU) │ │ Enterprise X     │
            │ ┌──────────────┐ │ │             │ │ (dedicated)      │
            │ │ Gateway      │ │ │             │ │                  │
            │ │ Messaging    │ │ │             │ │                  │
            │ │ Cassandra    │ │ │             │ │                  │
            │ │ Redis        │ │ │             │ │                  │
            │ │ ES (per-ws)  │ │ │             │ │                  │
            │ │ Kafka        │ │ │             │ │                  │
            │ └──────────────┘ │ │             │ │                  │
            │ Holds 10K        │ │             │ │ Holds 1 customer │
            │ small workspaces │ │             │ │ (e.g. 100K users)│
            └──────────────────┘ └─────────────┘ └──────────────────┘
                              │              │             │
                              └──────────────┴─────────────┘
                                             │
                              ┌──────────────▼───────────────┐
                              │   Cross-Cell Bridge          │
                              │ - Slack Connect (cross-      │
                              │   workspace DMs)             │
                              │ - User home-cell directory   │
                              │ - Federation events          │
                              └──────────────────────────────┘
```

### Routing

- **Workspace Directory** (replicated KV at the edge): `workspace_id → cell_id`.
- API Gateway looks up cell, routes the request. Sub-millisecond at the edge.
- WS connect: client `GET /v1/gateway` returns a cell-specific gateway URL. Client connects there.
- Mis-routed requests (rare) get a 307 redirect to the correct cell.

### Slack Connect / cross-workspace DMs — the cross-cell case

Two users in different workspaces (different cells) want to DM each other. Approaches:

| Approach | How | Pros | Cons |
|---|---|---|---|
| **Federation (cross-cell calls)** | DM channel exists in BOTH cells; each cell's writer mirrors writes to the other via Bridge | Each cell stays authoritative for its half; no replication of full data | Complex; ordering across cells requires a federation protocol |
| **Pinned cell for the DM** | DM lives in one cell (say, the inviter's); the other workspace's users connect cross-cell to read/write | Simpler ordering | Cross-cell read latency on every message; failure-coupling |
| **Bridge as the DM owner** | Bridge service holds cross-workspace DMs in its own store | Clean isolation | Bridge becomes a giant service; cost |

Slack does **federation with the DM channel mirrored**. Each side's users write to their local cell; Bridge replicates to the other side. Ordering is per-side total order, with a tiebreaker for cross-cell views. Complexity, but it's the right shape: most workspace DMs (99%) stay within the workspace, only 1% are cross-workspace.

### Cell-level operations

- **Per-cell deploys** — same binary, deployed cell-by-cell. Customer X's cell deploys on Tuesdays; cell-A on Wednesdays.
- **Per-cell on-call** — large enterprise cells have dedicated on-call rotations.
- **Per-cell capacity** — sized for that customer's expected use. They onboarded 1000 employees this quarter? We grow their cell.
- **Per-cell schema migrations** — can run staggered. Cell-A on v3 while cell-B is still on v2 (with bridge translating).
- **Workspace migration** — customer upgraded from Standard to Enterprise → migrate their workspace to a dedicated cell. Multi-step process: snapshot Cassandra, restore in target cell, dual-write window, cutover, decommission old. **Often a months-long project for a big customer.**

### When NOT to do per-workspace cells

- Pre-scale: if you have <10K workspaces, shared multi-tenant cells are simpler.
- Free / freemium tier: per-workspace cells are too expensive; share aggressively.
- The product doesn't have enterprise customers paying for it — no ROI on dedicated infra.

### What "Staff signal" sounds like here

> "The default architecture is shared multi-tenant cells per geography — say one US cell, one EU cell. The premium tier is dedicated per-workspace cells for enterprise customers, where the cell IS the unit of compliance audit, SLA negotiation, and blast radius. The interesting design problem is cross-cell interactions — Slack Connect DMs — which I'd model as federation with per-side authoritative writes and a bridge that replicates. Workspace migration is a real operational story; you can't ignore it because customers buy you over the upgrade path. The tax is significant — operational complexity scales with cell count, not user count — but it's the only architecture that lets you tell a Fortune 500 customer 'your data is on dedicated infrastructure with these specific availability and audit guarantees,' which is what the largest contracts require."
