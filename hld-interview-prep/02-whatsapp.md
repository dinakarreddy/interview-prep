# 02 — Design WhatsApp / Messenger (1:1 chat, group chat, presence, multi-device, media)

> Canonical real-time messaging HLD problem. The interviewer cares less about whether you can spell "WebSocket" and more about whether you correctly identify that **the connection layer**, **fanout to per-device queues**, and **end-to-end encryption's effect on server-side features** are the only things that matter — and you spend your time there.

---

## 0. Problem statement (as the interviewer would drop it)

> **Interviewer:** "Design a chat system like WhatsApp or Messenger. Users can send 1:1 and group messages. Messages should be delivered in real-time when both parties are online, queued when the recipient is offline, and synced across multiple devices. Show me how you'd build it."

That's all you get. Your job: scope it, not solve it whole. The trap here is that the surface area is enormous (calls, status, payments, contacts sync, notifications) — Staff candidates cut hard on day one.

---

## 1. Clarifying questions

Pick 5–6. Don't fire them as a checklist — interleave with the interviewer's responses.

> **Candidate:** "Before I draw anything, let me scope this. A few questions."
>
> **Candidate:** "First — are voice and video calls in scope? They share the signaling layer with chat but the media-plane is a totally different system (SFU, STUN/TURN, jitter buffers). I'd rather time-box one of them."
>
> **Interviewer:** "Skip calls. Focus on text+media chat."
>
> **Candidate:** "Good. Second — is end-to-end encryption a hard requirement, or are we okay with server-readable messages? It changes a lot: server-side search becomes impossible, ML moderation has to move to the client, multi-device key sync becomes its own subsystem."
>
> **Interviewer:** "E2EE is a hard requirement. Signal Protocol-style. Server stores ciphertext only."
>
> **Candidate:** "Then I'll budget time for E2E in a deep dive. Third — what's the rough scale? DAU, message volume, peak-to-average ratio?"
>
> **Interviewer:** "Assume 1B DAU, 2B MAU, ~100B messages/day. Group chat up to 1024 members. ~4 devices per user is typical."
>
> **Candidate:** "OK — that's WhatsApp scale. Fourth — message delivery semantics? At-least-once with client dedup, or do we need exactly-once?"
>
> **Interviewer:** "At-least-once is fine; client dedups on `message_id`."
>
> **Candidate:** "Fifth — read receipts and presence (last-seen, online-now) — are those P0 features or P2?"
>
> **Interviewer:** "P1. They have to work but the design shouldn't bend itself around presence."
>
> **Candidate:** "Sixth — multi-region active-active or single-region with edge POPs? Specifically, are there data-residency constraints (EU, India)?"
>
> **Interviewer:** "Multi-region. Assume EU and India have data residency requirements."
>
> **Candidate:** "Great. So I'll design:"
>
> **In scope (P0):**
> - 1:1 messaging with sent / delivered / read states
> - Group messaging up to 1024 members
> - Multi-device sync (4 devices/user, all-active)
> - Media attachments (image, voice note, short video) via signed-URL upload to object storage
> - Offline message queueing (30-day retention)
> - End-to-end encryption (Signal Protocol)
>
> **In scope (P1):**
> - Presence (online / last-seen) with privacy controls
> - Read receipts (per-conversation opt-in/out)
> - Reactions and replies
> - Push notifications for offline devices
>
> **Out of scope (state explicitly):**
> - Voice / video calls (separate problem — different media plane)
> - Status / Stories
> - Payments (WhatsApp Pay) — different consistency profile
> - Channels / broadcast lists at >1024 members (becomes a pub/sub problem, not chat)
> - Account creation, contact discovery, phone-number verification
> - Spam / abuse classification (acknowledge as a hook, don't design)
>
> **Candidate:** "I'll spend most time on the connection gateway, the fanout-to-devices model, and how E2EE constrains the design. Sound right?"
>
> **Interviewer:** "Yes."

---

## 2. Functional requirements

| Priority | Requirement |
|---|---|
| P0 | User can send a 1:1 message; recipient gets it in real-time if online, queued if offline |
| P0 | User can send to a group of up to 1024 members |
| P0 | Message states: sent → delivered (recipient device received) → read (recipient opened) |
| P0 | Multi-device: same user can have up to 4 active devices; all see the same conversation state |
| P0 | Media messages (image up to 16MB, video up to 100MB, voice notes) attached to a message |
| P0 | Offline message queue per device, 30-day retention |
| P0 | End-to-end encryption — server stores ciphertext only |
| P1 | Presence: online status + "last seen at" (privacy-controlled) |
| P1 | Read receipts (per-conversation opt-in) |
| P1 | Reactions on messages, replies/quoted messages |
| P1 | Push notifications via APNs / FCM when device is offline |
| P2 | Typing indicators |
| P2 | Message edit / delete-for-everyone (within a window) |
| Out | Voice/video calls, Status, Payments, channels >1024, contact discovery |

---

## 3. Non-functional requirements

| Dimension | Target | Justification |
|---|---|---|
| Availability | 99.99% for message delivery; 99.9% for presence | Chat is a daily-driver; outages are headline news. Presence tolerates more degradation. |
| Send → Deliver latency (both online) | p50 < 200ms, p99 < 500ms intra-region; p99 < 1s cross-region | Below the perceptual "instant" threshold |
| Send → ack to sender | p99 < 300ms | Sender sees the checkmark immediately or the UX feels broken |
| Offline → online catch-up | p99 < 5s for ≤500 queued msgs | Reasonable on app cold-start |
| Durability | Zero message loss after server ack | Losing messages is unacceptable; durability ≥ 11 nines |
| Ordering | Per-conversation monotonic; **not** global | Global ordering is impossible at this scale and unnecessary |
| Consistency | Per-conversation: causal + monotonic-reads | A reply must follow the message it replies to |
| Scale | 1B DAU, 100B msgs/day, 50M+ concurrent connections/region | Industry-typical assumption |
| Connections | 50M+ concurrent persistent TCP/WSS per region; 200M global | Hard cap: ephemeral port range × OS tuning |
| Media | Messages reference media by ID; CDN serves bytes | Don't blow up the message store with binary blobs |
| Geography | Multi-region active-active; EU + IN cells with residency | Latency + compliance |

---

## 4. Capacity estimation

> **Candidate:** "Let me estimate aggressively — round numbers. The two punchlines I'm headed for are concurrent-connection count and group fanout volume."

### Users

- 1B DAU, 2B MAU
- 4 devices/user → up to **4B total active devices**
- Concurrent online: empirically ~20% of DAU = **200M concurrent users globally**
- With multi-device, each online user has ~2 connections live → **~400M concurrent connections globally**
- Across 5 regions on average → **~80M concurrent connections per region** (peak region: ~150M)

### Messages

- 100B messages/day
- Per second avg: 100B / 86,400 ≈ **1.16M msg/s**
- Peak (3×): **~3.5M msg/s globally**
- Per region peak (assume top region = 30%): **~1M msg/s peak in the largest region**
- Avg message size (ciphertext + envelope): ~500 B for text; ~2 KB if reply/quoted/threaded

### Group fanout amplification

- Suppose 30% of messages are in groups, avg group size 50, max 1024
- 70% × 1B = 700M 1:1 msgs → 700M × 2 device-deliveries (sender's other devices + recipient's devices on avg 4) = ~5.6B device-deliveries
- 30% × 1B = 300M group msgs → 300M × 50 (avg group) × 4 (devices) = **60B device-deliveries**
- **Total: ~65B device-level deliveries/day** — this is the real number that drives fanout
- Per second: 65B / 86,400 ≈ **750K device-deliveries/sec avg, 2–3M peak**

### Storage

- Server stores ciphertext only — **30-day offline retention**, then dropped (per WA policy historically; iMessage/iCloud-backed differ)
- 100B msg/day × 500 B = **50 TB/day of message ciphertext**
- × 30-day retention = **1.5 PB hot**
- × 3 RF (Cassandra) = **4.5 PB replicated, hot tier**
- Per year if we extended retention: 100B × 365 × 500B × 3 = **~55 PB/year × 3 RF = 165 PB**
- **Decision driver:** we are NOT a long-term archive. Ciphertext drops at retention. Clients are the source of truth for chat history.

### Media

- ~10% of messages have media; avg media size 200 KB (mostly photos), 5% are videos at ~5 MB
- Media bytes/day: 10B × 200KB + 0.5B × 5MB = 2 PB/day + 2.5 PB/day = **~4.5 PB/day**
- Hot retention 30 days, then cold-tier or evict per client policy: **~135 PB hot**
- Stored once in object storage (S3-class), CDN-fronted

### Bandwidth

- Egress: 750K msg/s × 500B avg = 375 MB/s for chat metadata
- Plus media via CDN: ~50 GB/s peak in CDN egress (CDN, not us)
- WebSocket overhead per connection: ~5 KB/s steady-state for keepalives × 80M conns = **400 GB/s region heartbeat overhead** — dominated by TCP stack, not application bytes

### Connection gateway sizing (the punchline)

- 80M conns per region peak / 50K conns per host = **1,600 connection-gateway hosts/region**
- Top region: 150M / 50K = **3,000 hosts**
- Memory per conn: ~50 KB (TLS state + buffers + per-user routing) → 50K × 50 KB = 2.5 GB/host. Trivial.
- CPU per conn: dominated by TLS handshake on connect; steady-state is heartbeat work. ~10K conns/CPU. So 5 cores per host @ 50K conns = comfortably fits a c6i.4xlarge.

### Why this matters

- Connection gateway is the **single largest fleet** in the system. ~10K hosts globally just to hold sockets open.
- Group fanout writes (60B/day) dictate the chat-service / Kafka throughput.
- Media storage (~135 PB hot) is a CDN + S3 problem, not a database problem.

---

## 5. API design

> **Candidate:** "I'll split this into client-facing (mobile app) and inter-service. Client-facing is mostly WebSocket frames, not REST."

### Client ↔ Connection Gateway: WebSocket frame protocol

```
// Connect
CONNECT { user_id, device_id, auth_token, last_seen_message_id_per_conversation }

// Send (client → server)
SEND_MSG { client_msg_id, conversation_id, recipient_user_ids[], ciphertext, content_type, media_id?, reply_to_msg_id? }

// Server ack to sender
MSG_ACK { client_msg_id, server_msg_id, server_ts }

// Deliver (server → recipient client)
DELIVER_MSG { server_msg_id, conversation_id, sender_user_id, sender_device_id, ciphertext, content_type, media_id?, server_ts }

// Recipient ack (delivered/read)
MSG_DELIVERY_RECEIPT { server_msg_id, status: 'delivered' | 'read', recipient_user_id, recipient_device_id }

// Presence
PRESENCE_UPDATE { user_id, status: 'online' | 'offline', last_seen_ts }
PRESENCE_SUBSCRIBE { user_ids[] }

// Typing
TYPING_START / TYPING_STOP { conversation_id }
```

**Why frame-based, not REST:** REST adds 1 RTT + auth on every send. Persistent WSS amortizes auth + TLS over the session. We DO use REST for session-bootstrap and one-shot operations (uploads, key-fetch).

### REST surface (one-shot operations)

```
POST /v1/sessions { device_id, auth_creds }
  → { ws_endpoint, session_token, ttl }

POST /v1/media/uploads { content_type, size }
  → 200 { media_id, signed_url, expires_at }
  // Client PUTs bytes to the signed_url directly; never through our app servers

GET /v1/keys/{user_id}/prekey-bundle
  → { identity_key, signed_prekey, one_time_prekeys[1] }
  // For X3DH session establishment

POST /v1/keys/upload  // device uploads its own prekeys
  → 200

GET /v1/conversations/{conv_id}/history?cursor=&limit=
  → { messages: [...], next_cursor }
  // Pull from server-side queue if device was offline

GET /v1/devices  // list user's devices
POST /v1/devices/link  // add a new device (companion device flow)
DELETE /v1/devices/{id}
```

### Inter-service (gRPC, mTLS)

- `ChatService.SubmitMessage(envelope)` — Connection Gateway → Chat Service
- `FanoutService.Deliver(server_msg_id, recipient_devices[])` — Chat Service → Fanout
- `RoutingService.LookupConnection(user_id, device_id) → gateway_host_id`
- `KeyService.GetPrekeyBundle(user_id, device_id)`

### Anti-patterns to avoid

- Don't return media bytes through your server — always signed-URL straight to S3/CDN
- Don't expose `server_msg_id` as a sequential int (PII / enumeration) — Snowflake or UUIDv7
- Don't make the client poll for presence — push-only via subscribed channels
- Don't return decrypted ciphertext anywhere — server must never have plaintext

---

## 6. Data model

### Tables / collections

#### `users` (DynamoDB-style KV, sharded by `user_id`)

| Column | Type | |
|---|---|---|
| user_id | bigint (Snowflake) | PK |
| phone_number_hash | string | secondary lookup |
| created_at | timestamp | |
| identity_key_pub | bytes | long-term Ed25519 |
| account_meta | json | |

#### `devices` (sharded by `user_id`)

| Column | Type | |
|---|---|---|
| user_id | bigint | PK part 1 |
| device_id | uuid | PK part 2 |
| device_type | enum (ios, android, web, desktop) | |
| push_token | string | APNs/FCM token, opaque |
| signed_prekey | bytes | rotated weekly |
| identity_key_pub | bytes | per-device |
| last_active_at | timestamp | |
| linked_at | timestamp | |

#### `prekeys` (sharded by `user_id` + `device_id`)

| Column | Type | |
|---|---|---|
| user_id | bigint | PK1 |
| device_id | uuid | PK2 |
| prekey_id | int | PK3 |
| prekey_pub | bytes | |
| consumed | bool | one-time prekey, deleted on consume |

#### `conversations` (sharded by `conversation_id`)

| Column | Type | |
|---|---|---|
| conversation_id | bigint (Snowflake) | PK |
| type | enum (one_to_one, group) | |
| created_at | timestamp | |
| group_meta | json | name, avatar, admin list — for groups |
| member_count | int | denormalized; max 1024 |

#### `conversation_members` (sharded by `conversation_id`; **also** by `user_id`)

| Column | Type | |
|---|---|---|
| conversation_id | bigint | |
| user_id | bigint | |
| role | enum (member, admin) | |
| joined_at | timestamp | |
| read_watermark_msg_id | bigint | last message this user has read |
| muted_until | timestamp? | |

- Stored twice (denormalized): once partitioned by `conversation_id` (fanout: "who do I deliver to?") and once by `user_id` (UI: "what conversations am I in?").

#### `messages` (Cassandra/Scylla — wide row keyed by `conversation_id`)

| Column | Type | |
|---|---|---|
| conversation_id | bigint | partition key |
| message_id | bigint (Snowflake — time-sortable) | clustering key DESC |
| sender_user_id | bigint | |
| sender_device_id | uuid | |
| ciphertext | blob | per-device encrypted (or sender-keys for groups) |
| content_type | enum (text, image, video, voice, reaction, system) | |
| media_id | string? | reference into object storage |
| reply_to_msg_id | bigint? | |
| created_at | timestamp | |

- **Partition key = conversation_id** is the single most important schema decision. All reads ("show me conversation X") and writes (one append) hit a single partition. Hot partitions = hot conversations (e.g., a 1024-member work group during an event); we mitigate with caching and admission control.

#### `device_inbox` (per-device offline queue — Cassandra, partitioned by `device_id`)

| Column | Type | |
|---|---|---|
| device_id | uuid | partition key |
| message_id | bigint | clustering key (Snowflake; sorts by time) |
| envelope | blob | the full delivery payload |
| inserted_at | timestamp | |
| ttl | 30 days | TTL'd at storage layer |

- This is the **multi-device sync source-of-truth**. When a device reconnects, it scans `device_inbox[device_id]` since `last_seen_message_id` and drains.
- Per-device, not per-user — because each device has different drain state.

#### `presence` (Redis cluster, key = `user_id`)

| Field | Type | |
|---|---|---|
| user_id | int | key |
| status | enum (online, offline) | |
| last_seen_ts | int (epoch sec) | |
| connected_devices | set<device_id> | |
| TTL | 30 sec | refreshed on heartbeat |

#### `connection_directory` (Redis, key = `(user_id, device_id)` → `gateway_host_id`)

| Field | Type | |
|---|---|---|
| user_id | int | |
| device_id | uuid | |
| gateway_host_id | string | which Connection Gateway holds this socket |
| connected_at | timestamp | |
| TTL | 60 sec | refreshed by gateway on heartbeat |

- Authoritative routing table. Fanout asks: "where is this device's socket?" and pushes to that host.

### Why these choices

- **Cassandra for `messages`**: append-only, wide-row by conversation, time-sortable, write-heavy, eventual consistency tolerable per-conversation. Linear scale to PB. The wide-row pattern + clustering by Snowflake message_id matches `SELECT … WHERE conversation_id=X ORDER BY message_id DESC LIMIT 50` perfectly — one partition, one disk seek, no fanout in the read path.
- **DynamoDB-style KV for `users`/`devices`**: low-cardinality access pattern (always by user_id), no range queries needed, low latency.
- **Redis for `presence` and `connection_directory`**: ephemeral, sub-ms reads, set/get with TTL. Loss is acceptable (presence is best-effort).
- **Snowflake IDs for `message_id`**: sortable in time, no central allocator, gives us per-conversation monotonic ordering when issued by the same Chat Service shard.

---

## 7. High-level architecture

```
                       ┌─────────────────────────────────────────────┐
                       │            Mobile / Desktop Client          │
                       │      (holds 1 persistent WSS per device)    │
                       └─────────────────────┬───────────────────────┘
                                             │
                                  ┌──────────▼──────────┐
                                  │   GeoDNS / Anycast  │
                                  └──────────┬──────────┘
                                             │
                                  ┌──────────▼──────────┐
                                  │  L4 LB (TCP/WSS)    │  ← consistent-hash on
                                  │  + WS-aware router  │     user_id for stickiness
                                  └──────────┬──────────┘
                                             │
              ┌──────────────────────────────┼──────────────────────────────┐
              │                              │                              │
       ┌──────▼──────┐               ┌──────▼──────┐               ┌──────▼──────┐
       │ Conn GW #1  │  ........     │ Conn GW #N  │   ........    │ Conn GW #M  │
       │ (stateful)  │               │ (stateful)  │               │ (stateful)  │
       │ holds 50K   │               │ holds 50K   │               │ holds 50K   │
       │ WSS conns   │               │ WSS conns   │               │ WSS conns   │
       └──────┬──────┘               └──────┬──────┘               └──────┬──────┘
              │                              │                              │
              │ on connect: register         │                              │
              │ (user_id,device_id)→host     │                              │
              ▼                              ▼                              ▼
       ┌──────────────────────────────────────────────────────────────────────┐
       │                    Connection Directory (Redis)                       │
       │           (user_id, device_id) → gateway_host_id, TTL 60s             │
       └──────────────────────────────────────────────────────────────────────┘
              │                              │                              │
              │ SEND_MSG via gRPC            │                              │
              ▼                              ▼                              ▼
                         ┌──────────────────────────────┐
                         │       Chat Service           │
                         │     (stateless ingress)      │
                         │  - validates envelope        │
                         │  - assigns server_msg_id     │
                         │  - writes to messages table  │
                         │  - publishes to Kafka        │
                         └─────┬─────────────────┬──────┘
                               │                 │
                               ▼                 ▼
                       ┌─────────────┐    ┌──────────────┐
                       │  Cassandra  │    │  Kafka       │
                       │  messages   │    │  topic:      │
                       │  (RF=3)     │    │  msg.created │
                       └─────────────┘    └──────┬───────┘
                                                 │
                                                 ▼
                                  ┌──────────────────────────────┐
                                  │    Fanout / Delivery Service │
                                  │  - look up conv_members      │
                                  │  - look up devices/user      │
                                  │  - look up gateway/device    │
                                  │  - push DELIVER_MSG to GW    │
                                  │  - write to device_inbox     │
                                  │  - APNs/FCM if device offline│
                                  └──┬──────────┬─────────┬──────┘
                                     │          │         │
                                     ▼          ▼         ▼
                          ┌─────────────┐  ┌─────────┐  ┌──────────┐
                          │ Conn GW     │  │ device_ │  │ APNs/FCM │
                          │ (push to    │  │ inbox   │  │ Push svc │
                          │ live socket)│  │ Cassandra│  │          │
                          └─────────────┘  └─────────┘  └──────────┘

       ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐
       │ Presence Svc    │  │  Key Server     │  │ Media Service   │
       │ Redis-backed    │  │  prekey bundles │  │ + Object Store  │
       │ pub/sub channels│  │  identity keys  │  │ + CDN           │
       └─────────────────┘  └─────────────────┘  └─────────────────┘
```

### Request walk-through

**Send path (1:1 message, both devices online):**

1. Sender device has WSS open to `Conn GW #42`. Sends `SEND_MSG { client_msg_id, conversation_id=C, ciphertext_per_device[…], … }`.
2. Conn GW #42 forwards via gRPC to **Chat Service** (any instance — stateless).
3. Chat Service:
   - Validates auth, conversation membership.
   - Assigns `server_msg_id` (Snowflake).
   - Writes to `messages` (Cassandra, RF=3, CL=LOCAL_QUORUM). p99 ~10ms intra-region.
   - Publishes `msg.created { server_msg_id, conversation_id, ciphertext_envelopes[] }` to Kafka, partitioned by `conversation_id` (preserves per-conversation ordering).
   - Returns `MSG_ACK` to Chat Service caller → Conn GW → sender. Sender sees the gray checkmark. **Total ~80ms.**
4. **Fanout Service** consumer reads the Kafka event:
   - Looks up conversation members (`conversation_members[conversation_id]`).
   - For each recipient user_id, looks up that user's devices (`devices[user_id]`).
   - For each device, looks up `connection_directory[(user_id, device_id)]`:
     - If found → push `DELIVER_MSG` to that gateway via gRPC stream; gateway pushes down the WSS to the client.
     - If NOT found (offline) → write envelope to `device_inbox[device_id]`, optionally trigger APNs/FCM push notification.
   - Also fans out to **sender's other devices** (multi-device sync): same path, but with the sender's own device list minus the originating device.
5. Recipient device receives `DELIVER_MSG`, sends `MSG_DELIVERY_RECEIPT { status: delivered }` back. This is a small ack; Chat Service writes it as a pseudo-message of `content_type: receipt` (or to a separate receipts table — see deep dive 4 on tradeoffs).
6. When user opens the chat, recipient device sends `MSG_DELIVERY_RECEIPT { status: read }` per the privacy preference.

**Total send→deliver latency** (both online, same region): ~150–250ms p99.

**Send path (group of 200, half offline):**

Same as above, but Fanout iterates over 200 members × 4 devices = 800 deliveries. Half land in live sockets, half land in `device_inbox` + push notification. Total fanout work for this single message: 800 writes/pushes, completes in ~500ms across the worker pool.

**Reconnect / catch-up path:**

1. Device A wakes from background. Establishes WSS to nearest Conn GW.
2. Conn GW upserts `connection_directory[(user_id, device_id)] = gw_host_id`.
3. Client sends `last_seen_message_id_per_conversation` map in CONNECT frame (or queries `GET /v1/conversations/{id}/history?cursor=…` for each).
4. Server reads from `device_inbox[device_id]` since `last_seen_message_id`, streams envelopes back ordered by message_id.
5. Once drained, client is "caught up" and live messages flow normally.

### Why this hybrid (online push + offline queue + APNs)

- **Pure push (live socket only)**: fails the moment the recipient is offline — and on mobile, "offline" is the common case (background, screen off, network change).
- **Pure pull (client polls)**: wastes battery, wastes server CPU on pointless QPS, and you can't deliver real-time when both ends are online.
- **Hybrid (push when online, queue when offline, APNs to wake the device)** matches the workload: <20% of users online at any moment, but 100% need messages eventually.

This is the single most important insight at the architecture layer. State it explicitly.

---

## 8. Deep dives

### Deep dive 1: The connection gateway (50M+ persistent connections per region)

> **Interviewer:** "Walk me through the Connection Gateway. How does a single host hold 50K WebSocket connections, and what happens when one host dies?"

**Candidate:**

The Connection Gateway is the most operationally fragile component because it's **stateful** — it holds a TCP socket per device. Everything else (Chat Service, Fanout) is stateless and trivially replaceable. The gateway is not.

**Per-host capacity:**
- 50K WSS connections × ~50KB memory (TLS state + buffers + per-conn user struct) = ~2.5 GB RAM. Fits c6i.4xlarge with headroom.
- File descriptors: 50K. Need to bump `ulimit -n` to ~200K (room for upstreams).
- Ephemeral ports on the **client side** of the gateway: irrelevant (gateway is the server).
- Ephemeral ports for **gateway → Chat Service** gRPC: pooled connections, not 1:1 with WSS, so we use ~500 connections.
- CPU: idle WSS heartbeat at 30s intervals → 50K / 30 = ~1700 events/sec. Trivial. The CPU spike is at connect time (TLS handshake) and at deploy time (re-establishment storm).

**Sticky session sharding:**
- L4 LB hashes by `user_id` (or by IP if user_id isn't visible at L4) → routes to a deterministic gateway host. This means a user's two devices land on (likely) the same host or nearby hosts in the same shard. Useful for cache locality.
- We use **consistent hashing** with virtual nodes so that adding/removing hosts moves only ~1/N of users, not all of them.
- Stickiness is at the **session** level (TLS session) — once connected, the LB just pipes bytes; it doesn't re-route per-message.

**Failure of one gateway host (`gw-42` dies):**

1. Detection: health check fails within 5–10s. LB removes from rotation.
2. Blast radius: 50K users (their socket drops). Their other devices on other gateways are unaffected.
3. Client behavior: client detects dropped socket within 2–5s (TCP keepalive or app-level heartbeat), reconnects with **jittered exponential backoff** (jitter is critical — without it, 50K clients all reconnect in the same 100ms window and DDoS the LB).
4. New connection lands on a different gateway (consistent-hash redistributes). Updates `connection_directory`.
5. During the window of disconnection: messages routed to that user's device fall into `device_inbox` (the offline path). On reconnect, drain.
6. **Total user-visible impact:** 5–15s of "no live messages", recovered automatically. Messages are not lost — they're queued.

**Failure of one gateway region (much worse):**

1. ~30M users disconnected.
2. Re-connection storm to remaining regions. We pre-provision **30% headroom** specifically for this case.
3. DNS-level failover: GeoDNS swings traffic to nearest healthy region. Takes 30–60s for cache TTL to expire.
4. Cross-region message routing: a user in failed region's home cell now connects to a backup cell — but their offline queue is in the home cell's Cassandra. Cross-region reads to drain. Acceptable degradation; eventual recovery.

> **Interviewer:** "What about deploys? You said the gateway is stateful — how do you push code without dropping 50M connections?"

**Candidate:**

This is the hardest operational problem in the whole system. Three strategies, in order of sophistication:

1. **Naive: rolling restart.** Take 5% of fleet down, redeploy, bring up, repeat. Means 5% of connections drop per wave (~2.5M users). Reconnect backoff handles it but it's a thunderclap. **Used by us in disaster recovery only.**
2. **Drain + reconnect:** before SIGTERM, gateway sends `RECONNECT_HINT` frame to all clients telling them "reconnect to LB in 10–30s with jitter". Smooths the storm. Clients still drop, but spread over 30s. **Our default for routine deploys.**
3. **Hot reload of binary, retain sockets:** kernel passes file descriptors to the new process via `SO_REUSEPORT` + Unix domain socket fd-passing (Envoy / Nginx-style hot reload). Sockets survive the binary swap. Application-level state has to be serialized + restored. Complex; we'd reserve this for security patches that **must** ship without disconnecting users. **Used by WhatsApp historically (per public talks).**

> **Interviewer:** "Won't sticky sessions create hotspots if a celebrity user has 100M followers DM'ing them?"

**Candidate:**

Inbound DMs to a celebrity don't go through their gateway socket — they go through the **sender's** gateway, then through Chat Service / Fanout. The fanout pushes to the celebrity's gateway only when delivering. So the celebrity's single socket sees 100M deliveries/sec? No — it sees N/sec where N is how many messages are written to their inbox. Even Cristiano Ronaldo isn't reading his DMs at 100M/sec.

The real hotspot is the celebrity's **device_inbox** Cassandra partition — not the gateway connection. We mitigate that with admission control (cap inbound rate per recipient) and bucketing (`device_inbox_bucket = device_id || (timestamp / 10s)`), but it's a long tail problem, not a typical-user problem.

---

### Deep dive 2: Multi-device sync and the per-device fanout model

> **Interviewer:** "User has 4 devices: phone, tablet, laptop, web. They send a message from phone. How does it appear on the other 3? And how does an incoming message reach all 4?"

**Candidate:**

The naive answer is "we deliver to the user." That's wrong. We deliver to **devices**, not users. Multi-device is fundamentally a per-device fanout problem, and the architecture has to acknowledge that.

**Why per-device, not per-user:**

1. **E2E encryption demands it.** Each device has its own keypair. The same plaintext message is encrypted *separately* for each recipient device using that device's identity key. The server can't deliver "the message to Alice" because there is no single message — there are 4 ciphertexts, one per Alice's devices.
2. **Read state is per-device.** If you read on phone, your tablet should still show the message; what should sync is the *read receipt* (a separate event), not the message itself.
3. **Offline state is per-device.** Phone could be offline, laptop online. Same message must reach laptop now, phone on reconnect.

**Send-side fanout to sender's other devices:**

When Alice (`device_phone`) sends "hi" to Bob:
- Alice's app encrypts the message **per recipient device**. With 4 Bob devices and 3 Alice-other devices = 7 ciphertexts.
- Single `SEND_MSG` envelope contains: `[(recipient_device_id, ciphertext)]` for all 7.
- Chat Service writes one logical message row in `messages[conversation_id]` with the full envelope.
- Fanout iterates the 7 recipients, pushes to each. Alice's other 3 devices receive the message and decrypt with their own keys → "outgoing" message appears in their UI.

**Receive-side fanout to all of Bob's devices:**

Same mechanism. Each device has its own `device_inbox` partition. Each device's WSS gets the appropriate ciphertext.

**Why this is hard:**

- **Key distribution amplifies on group sends.** Alice sends to a group of 100. Each member has 4 devices. Naively: 400 ciphertexts per message. This is why Signal Protocol uses **Sender Keys** for groups (see deep dive 3) — sender encrypts once with a group key, distributes the group key per-device pairwise. Reduces 400 ciphertexts to (1 message + 400 key-message pairs done once at key-rotation). Big win.
- **Adding a new device** triggers a re-keying of every group conversation the user is in. This is non-trivial; we batch it and do it lazily ("reload chat to see history" UX).
- **Old devices that come back online** after a long offline period might have stale group keys. They request a fresh sender-key from a peer device.

**Read-receipt sync across own devices:**

When Bob reads on phone:
- Phone sends `MSG_DELIVERY_RECEIPT { status: read, conversation_id, up_to_msg_id }`.
- Chat Service writes this to the `messages` table (or a dedicated `read_watermarks` table — see tradeoffs in deep dive 4).
- Fanout pushes the read-event to: (a) sender Alice's devices (for the blue checkmark), and (b) Bob's other devices (so tablet doesn't show the message as unread).

**Drain order on reconnect (subtle):**

When phone reconnects after a day offline, it must drain `device_inbox` in `message_id` order (Snowflake = time-sorted). But received messages are interleaved with sent messages (from when phone sent something while laptop was online — that send came back to phone via fanout). The clustering key on `message_id` handles this; client sees a single ordered stream.

> **Interviewer:** "What if the same message is delivered twice — say a retry across a network blip?"

**Candidate:**

At-least-once delivery + client dedup on `server_msg_id`. The server may push a `DELIVER_MSG` twice if the gateway retries on no-ack. Client maintains a sliding window of recent `server_msg_id`s (say last 1000) and discards duplicates. Idempotent on the read side; the write was already `INSERT IF NOT EXISTS`-style at the messages table (Cassandra LWT, or just allow benign duplicate keys).

> **Interviewer:** "What if the user adds a 5th device?"

**Candidate:**

WhatsApp-style: cap at 4 companion devices linked to one primary (the phone). Adding a 5th requires unlinking one. Why a cap? Each device adds key-distribution overhead + an inbox partition + a connection. Cost grows linearly. Product decision; technically we could support 10, but UX gets noisy and security-review on key-leak surface area worsens.

iMessage-style: no primary, all peers. More elegant but requires harder key-graph coordination (key transparency). Out of scope here; flag as a tradeoff.

---

### Deep dive 3: End-to-end encryption and how it constrains everything

> **Interviewer:** "Explain how E2EE works in this design. What does the server know? What does it not know? How does a new device join?"

**Candidate:**

Three pieces: **session establishment (X3DH)**, **message encryption (Double Ratchet)**, **groups (Sender Keys)**. The Signal Protocol covers all three.

**(a) Session establishment — X3DH**

When Alice's device wants to start a conversation with Bob's device:
1. Alice fetches Bob's **prekey bundle** from our Key Server: `{ identity_key_pub, signed_prekey, one_time_prekey[1] }`.
2. Alice computes a shared secret using Diffie-Hellman over (identity, signed prekey, one-time prekey, ephemeral). This is X3DH.
3. Alice sends her first message; the envelope includes her ephemeral key so Bob can derive the same shared secret.
4. The one-time prekey is **consumed** (deleted from server) so it can't be replayed.

What the server stores:
- Identity public key per device
- Signed prekey (rotated weekly)
- ~100 one-time prekeys per device (replenished by the device)
- **Never** any private keys. Ever.

What the server cannot do:
- Read message contents (it has only ciphertext)
- Impersonate users (it doesn't have private keys)
- Replay attacks (one-time prekeys consumed)

What the server *can* do (and what users must trust us not to do):
- Lie about a user's identity key (substitute our own → MITM). Clients defend with **safety numbers / QR code verification** ("verify Alice's number out-of-band").
- Withhold prekeys (DoS). Acceptable failure mode.

**(b) Double Ratchet**

Per-conversation rolling key. Each message uses a fresh symmetric key derived from a chain. Two ratchets:
- **DH ratchet** (advances on every received message round-trip): provides forward secrecy across long timescales.
- **Symmetric ratchet** (advances on every sent message): provides forward secrecy within a session.

Net: if Alice's device key is compromised today, attacker cannot decrypt yesterday's messages.

Server-side implication: **none**. The server just stores ciphertexts. The protocol is entirely in the client.

**(c) Group encryption — Sender Keys**

Naive: sender encrypts the message N times for N members. 1024-member group → 1024 ciphertexts per message. Bad.

Sender Keys: sender generates a symmetric **group key** once. Encrypts the message ONCE with the group key. Distributes the group key to each member device pairwise (using their pairwise X3DH/Double Ratchet sessions). Members decrypt with the group key.

- One ciphertext, one fanout message, N small key-distribution messages (only when keys rotate).
- Forward secrecy: rotate the group key periodically (e.g., every 100 messages, or on member add/remove).
- **On member leave:** rotate immediately (so leaver can't read future messages).

This is why server-side **group operations** (add/remove member) are expensive: every active sender's device must regenerate and redistribute a new group key. We batch this and tolerate ~5s eventual consistency on group-state.

**What does the server know about messages?**
- Sender device ID, recipient device IDs, conversation ID, timestamp, content type (text/image/voice — not the bytes), media reference, ciphertext size.
- **Metadata is leaked.** Server knows you talked to Bob at 3pm. Can't read what you said. WhatsApp publishes this in their privacy whitepaper.

**What changes in our design because of E2EE:**

1. **No server-side search.** Search is client-side only, on locally-decrypted history. Or, mobile-only since web devices have less storage.
2. **No server-side moderation.** Spam classifiers run on metadata (rate, fanout shape) or on-device (with reporting hooks).
3. **No server-side analytics on content.** Metrics like "average message length" require client-side telemetry, opt-in.
4. **Backups are tricky.** Either: client encrypts with a user-derived key + cloud stores the ciphertext (iMessage iCloud), or: backups are local-only (WhatsApp Google Drive option requires a passphrase). We'd offer both; flag for product decision.
5. **Notifications are content-less.** Push notification payload is "you have a message from Alice" — server can't put the message preview because it doesn't have it. (Some clients fetch the message via APNs's data-channel + decrypt locally, then inject the preview.)
6. **Reactions, replies need ciphertext refs.** "React to message X" sends `{ reply_to_msg_id: X, ciphertext_of_emoji }`. Server links by ID, can't validate the emoji is a legitimate reaction.

> **Interviewer:** "What happens when Bob loses his phone and reinstalls? All his old messages are encrypted with keys he no longer has."

**Candidate:**

Right — server can't help. Three options, in order of common usage:

1. **Loss is loss.** Bob's history is gone. He starts fresh. Conversation partners' clients see "new device, security number changed, verify". UX is jarring but security-correct.
2. **Encrypted backup.** Client encrypts local DB with a passphrase / user-secret, uploads to user's iCloud / Google Drive. On reinstall, decrypts. Server is not involved in keys.
3. **Cross-device transfer.** New phone re-pairs with companion devices via QR code, pulls history off the still-trusted laptop. Doesn't help if all devices are lost.

The principle: **server is not a key escrow**. If we held keys, the entire E2EE story collapses.

---

### Deep dive 4: Read receipts, delivery receipts, and presence — the metadata firehose

> **Interviewer:** "How do you implement 'sent / delivered / read'? Is presence really that hard?"

**Candidate:**

These three together are 30%+ of message volume even though they're "just metadata". Let's count, then design.

**Volume estimation:**
- 100B messages/day → each generates: 1 sent ack to sender, 1 delivered receipt per recipient device, 1 read receipt per recipient device (when read).
- Avg recipients per message: 1.5 (skewed by 1:1 vs group); avg devices per recipient: 4.
- Per-message receipt events: 1 sent + 1.5 × 4 × 2 (delivered + read) = ~13 events.
- Total receipt events: **1.3T events/day**, ~15M/sec average.

**Receipts are the largest write volume in the system, larger than the messages themselves.**

**Design choice — receipts in `messages` table vs separate:**

| Approach | Pros | Cons |
|---|---|---|
| Receipts as rows in `messages` (content_type=receipt) | Single read path for "give me everything in this conversation"; consistent ordering | 13× write volume to a table that's already hot; bloats wide rows; complicates pagination |
| **Receipts in a separate `read_watermarks` table** (one row per (conversation, user, device) → high-watermark message_id) | 100× smaller; updates are key-overwrites not appends; simple semantic ("you're caught up to msg N") | Loses per-message granularity (don't know exactly when each was read) — but we don't need it |

**My pick:** separate `read_watermarks` table for read state. Per-message-delivered receipts go in a small ephemeral structure (TTL'd), since "delivered" is only useful for "show two checkmarks" UX.

```
read_watermarks (sharded by conversation_id)
  conversation_id | user_id | device_id | last_read_msg_id | updated_at
```

Sender computes "read by Bob" = `min(last_read_msg_id[bob][device]) >= my_msg_id` over Bob's devices. Wait — that's `min` in a dumb way; we want "any of Bob's devices has read up to here". Use the **max** across Bob's devices for the "Bob has read" flag. Per-device-precise UI is overkill.

**Privacy: read receipts can be turned off.**

User setting: "send read receipts" off → client sends DELIVERY_RECEIPT for `delivered` but never for `read`. Server doesn't enforce; it's a client behavior. Recipients of disabled-receipts users see "delivered" forever.

**Presence (online / last-seen):**

- Connection Gateway publishes to `presence` Redis on connect/disconnect: `SET presence:{user_id} {online, last_seen_ts}` with TTL 30s, refresh every 25s on heartbeat.
- Subscribers (people viewing your chat header) call `PRESENCE_SUBSCRIBE { user_ids: [you] }` → server adds them to a Redis pub/sub channel for those user_ids.
- Updates are pushed via the subscriber's WSS.

**The cost problem:**
- A user opens a chat with Alice → subscribes to `presence:alice`. If Alice has 1M people who could subscribe to her, and 50K of them open chat with her at any time, that's 50K subscribers to one Redis pub/sub channel.
- Worse: every time Alice flicks online/offline (commute, screen lock), 50K pushes go out. Phones on metered connections hate this.

**Mitigations:**
1. **Throttle:** debounce status changes on the server (don't propagate <30s flickers).
2. **Coarser granularity:** "last seen ~5 min ago" instead of "last seen 12:34:56". Round to nearest minute.
3. **Privacy-by-default:** only mutual contacts can see your presence (drop subscriber count by 100×).
4. **Sample at scale:** for 1M-following users (celebrities), don't broadcast presence to all subscribers — let the client poll on chat-open instead. Pull, not push.
5. **"Active in app"** instead of millisecond-accurate online: WhatsApp historically used a coarser "last seen", not a real-time bubble. Honest design choice.

> **Interviewer:** "Why not just have every message piggyback presence?"

**Candidate:**

Tried that on paper, doesn't work. Presence updates fire orders of magnitude more often than messages (every connect/disconnect/focus event). Coupling them creates dependency between two systems with very different SLAs. Keep them separate; presence is best-effort, messages are durable.

> **Interviewer:** "Typing indicators at scale?"

**Candidate:**

Treat them as ephemeral, never-stored events. Sent via the WSS on the sender side, fanned out only to actively-subscribed (i.e., the chat is open) recipients. Auto-expire client-side after 5s. Don't touch Cassandra. Don't even touch Redis if you can avoid it — gateway-to-gateway pubsub works.

For groups, throttle: at most one "X is typing" event per (sender, conversation) per 3s. Without throttle, a 1024-member chat with 20 active typers becomes a flame storm.

### Deep dive 5: Service-to-Gateway delivery mechanics (how Fanout pushes to a specific gateway host)

> **Interviewer:** "You said Routing tells Fanout *which* gateway holds the recipient's socket. Once you have `gateway_host_id`, how does the message actually get from Fanout to that gateway host?"

**Candidate:**

This is the hop most candidates wave over. It's worth being explicit because the choice has a 5–10× latency implication and dictates the failure model.

**Three real-world approaches:**

#### Option A — Long-lived gRPC bidirectional streams (my pick)

```
┌────────────────────┐      gRPC bidi stream      ┌──────────────────┐
│ Fanout host        │ ◄═════════════════════════►│ Gateway-1234     │
│ (one of ~200)      │   1 per (Fanout, Gateway)  │ (holds 50K WSS)  │
└────────────────────┘                             └──────────────────┘
                                                             │ WSS
                                                             ▼
                                                          Bob's phone
```

**How it works:**

1. At boot, each Gateway registers in service discovery (K8s endpoints / Consul) with `(host_id, ip, port)`.
2. Each Fanout host opens **one long-lived gRPC bidi stream to every Gateway host** lazily on first delivery — `200 Fanouts × 3,000 Gateways = 600K streams total`. Each stream costs ~30 KB of memory, so ~90 GB total across the Fanout fleet. Trivial.
3. On delivery:
   ```
   stream = pool.get("gw-eu-west-1234")
   stream.send(DeliverRequest{
     server_msg_id,
     device_id,
     envelope_bytes,
     deadline_ms: 2000,
   })
   ```
4. Gateway's stream handler receives, looks up `connMap[device_id]` (in-process Go/Erlang map — microsecond lookup), writes encrypted bytes to the WebSocket.
5. Gateway returns `DeliverResponse{server_msg_id, status: SENT | NO_CONN | ERROR}` on the same stream.
6. Fanout uses the response to ack to Chat Service, retry transiently, or fall through to offline queue + APNs/FCM if `NO_CONN`.

**Latency:** ~1–3 ms intra-DC (0.5 ms RTT × 2 + a couple ms processing). Total send→deliver budget of 200 ms is comfortable.

**Why bidi-stream not unary RPC:**
- One TLS handshake amortized over millions of messages
- HTTP/2 multiplexing — many in-flight messages on one stream
- Built-in flow control (Gateway can push back when overloaded)
- ~3× lower CPU than unary at high message rates

#### Option B — Per-gateway Redis pub/sub channel

```
Fanout ──PUBLISH──► Redis ──SUBSCRIBE──► Gateway-1234
            channel: gw.1234.deliver
```

Each Gateway subscribes to `gw.{host_id}.deliver` at boot. Fanout publishes; Gateway receives via subscription.

**Pros:** Decoupled, no service discovery, easy. 

**Cons:**
- Adds 5–15 ms broker hop (vs ~2 ms gRPC)
- Redis pub/sub is **fire-and-forget** — if the Gateway crashed *between* publish and consume, the message is lost. Forces a double-write to offline queue, doubling the write path.
- No per-message ack from broker (Kafka has acks but adds tens of ms)
- At 50M msg/sec, the broker fleet itself becomes critical-path infra

**When acceptable:** smaller scale; broadcast-style fanout (e.g., presence updates to *all* gateways — pub/sub naturally fans out). For 1:1 chat at WhatsApp scale, the latency tax + lossy semantics rule this out.

#### Option C — BEAM cluster message passing (what WhatsApp actually did)

Each WebSocket is an Erlang process. PIDs are cluster-global addresses. To deliver:

```erlang
Pid ! {deliver, MsgEnvelope}
```

The runtime routes to whichever node holds the PID. Functionally equivalent to Option A but baked into the language. WhatsApp ran 900M users with ~50 engineers on this model. Tradeoff: locks you into BEAM and clustering tops out around ~150 nodes per cluster (multi-cluster needed at hyperscale).

#### My pick and why

gRPC bidi-streams (Option A) for the 1:1 hot path:
- Lowest latency (~3 ms)
- Strong per-message ack semantics
- Industry-portable (no language lock-in)
- BEAM is the elegant alternative if you're already Erlang/Elixir

Reserve Redis pub/sub for **broadcast** events like presence changes that genuinely benefit from one-publish-many-receive semantics.

> **Interviewer:** "What happens when the routing lookup is stale — the user just reconnected to a different gateway?"

**Candidate:**

This is the race I'd surface unprompted. Four cases:

**(a) Routing is stale (most common).** User's WSS migrated from `gw-1234` to `gw-5678` in the last second; Routing's TTL hasn't expired.

- Fanout sends `DeliverRequest` to `gw-1234`
- `gw-1234` looks up `connMap[device_id]` — not there
- Returns `DeliverResponse{status: NO_CONN}`
- Fanout invalidates its routing cache entry for that device and re-queries `RoutingService.LookupConnection`
- Retries once. If still no conn (truly offline), writes to `device_inbox` + sends APNs/FCM.

Bound: one extra ~3 ms hop on the race. Acceptable.

**(b) Gateway dies between routing lookup and gRPC send.** Stream surface: `Unavailable`.

- Fanout drops the pool entry for that gateway
- Routing TTL is 60s — gateway's entries expire within a minute and Routing stops returning that host
- Within 60s, retries naturally re-route via Routing's new view
- For the in-flight message: retry via fresh Routing lookup, fall to offline path if still gone

**(c) Multi-device race.** User has 4 devices, one transitioned online→offline mid-fanout.

- Fanout iterates the 4 devices; for the transitioning one, gateway returns `NO_CONN`
- Fall to offline path **for just that device**. Other 3 got the message on the live socket.
- No coordination needed across devices — each is independent.

**(d) Idempotency.** Gateway might retry `DeliverRequest` on timeout. Client dedups on `server_msg_id` (sliding window of last 1000 seen ids). Same at-least-once semantic the doc commits to.

> **Interviewer:** "How big is the gRPC stream pool? Doesn't every Fanout connecting to every Gateway scale poorly?"

**Candidate:**

Concretely: 200 Fanout hosts × 3,000 Gateways = **600K streams**. Each stream consumes ~30 KB (HTTP/2 connection state + buffers). Total ~18 GB across the Fanout fleet, ~6 MB per Fanout host. Negligible.

If we grew Fanouts to 2,000 and Gateways to 30,000, we'd hit 60M streams — too many. At that scale I'd shard: each Fanout host owns a slice of Gateways (consistent hashed), and Routing tells Fanout "this device's gateway is in slice 7, hand off to a Fanout that owns slice 7." But we're nowhere near that breakpoint at WhatsApp scale.

**Single number to remember:** ~3 ms service-to-gateway hop, ~600K total streams, ~6 MB pool per Fanout host. Cheap compared to anything else in the system.

---

## 9. Tradeoffs & alternatives

### Decision: Persistent WebSocket + connection gateway vs HTTP long-poll vs MQTT

| Option | Pros | Cons | When to pick |
|---|---|---|---|
| **HTTP long-poll** | Trivially HTTP-friendly; works through every proxy; no special infra | High overhead per message (HTTP headers); battery drain on mobile; doesn't scale to bidirectional chat | Tiny chat product or behind hostile networks |
| **MQTT (over TCP/TLS)** | Tiny header overhead; designed for mobile; QoS levels built in | Requires MQTT broker fleet (HiveMQ/EMQX); harder to debug; less Web-friendly | High-scale mobile (Facebook Messenger uses MQTT historically) |
| **✅ WebSocket (WSS)** | Standard, web+mobile compatible, persistent, bidirectional | Custom gateway service to operate; sticky sessions; deploy complexity | Default at hyperscale, especially if web client matters |
| **gRPC streaming** | Typed schemas, multiplexing, HTTP/2 | Requires gRPC-Web for browser; less battle-tested at this scale for chat | Internal services, not client-facing chat |

I'd defend WSS by saying: "we have 1B clients including web, MQTT loses us the browser story, and the gateway operational complexity is what it is — we'd pay it in any persistent-connection model."

### Decision: Cassandra for `messages`

| Option | Pros | Cons | When to pick |
|---|---|---|---|
| **PostgreSQL** | ACID; familiar; great for small scale | Doesn't scale to 1.5 PB hot + 1M writes/sec; sharding is DIY | Small-scale chat or small per-tenant |
| **DynamoDB** | Managed; scales; predictable latency | Vendor lock-in; per-WCU/RCU pricing painful at this scale; eventual-consistency knobs are coarse | AWS-native team without ops capacity |
| **MongoDB** | Flexible schema | Wide-row semantics weaker than Cassandra for time-series-like reads | Schema flexibility > write-scale |
| **HBase** | Strong consistency; good for write-heavy | Operationally heavy (HDFS, Zookeeper); colder community | Hadoop shops |
| **✅ Cassandra / ScyllaDB** | Wide-row + clustering by message_id matches the access pattern; tunable consistency; linear scale; we have ops muscle | Eventual consistency on reads; secondary indexes weak; compaction tuning is its own discipline | Wide-row write-heavy chat at hyperscale |

**Why I'd pick Scylla over Cassandra in 2026:** same data model + wire protocol, dramatically lower per-node cost (10× throughput per host), better tail latency (shard-per-core, no JVM GC pauses). Tradeoff: smaller community, fewer 3rd-party integrations. At this scale, the cost win is 8-figure annually — worth the integration tax.

I'd defend by saying: "the wide-row + clustering-key pattern maps perfectly to a per-conversation timeline, the eventual consistency is fine because we use causal ordering at the conversation level, and we already have ops experience. If we were a 5-engineer team I'd flip to DynamoDB and pay the cost."

### Decision: Kafka for the fanout pipeline

| Option | Pros | Cons | When to pick |
|---|---|---|---|
| **SQS** | Managed | No ordering guarantees; per-msg cost; not great at millions/sec | Smaller systems; AWS-native |
| **RabbitMQ** | Flexible routing | Not at this scale | Complex routing, smaller scale |
| **Direct gRPC fanout (no queue)** | Lower latency end-to-end | No buffering; failures cascade; can't replay | Very small scale or extreme latency |
| **✅ Kafka** | Persistent log; partitionable by conversation_id; replayable; consumer-group scaling | Ops overhead; partition count is a forever decision | Hyperscale stream pipelines |
| **Kinesis** | Managed Kafka-equivalent | AWS lock-in; throttling at high QPS | AWS-native streaming |
| **NATS JetStream** | Lightweight, strong perf | Smaller community; less feature-rich | Smaller scale, simpler routing |

Partition by `conversation_id` so per-conversation order is preserved within a partition. Caveat: a 1024-member group sends a message → fanout work for that single message is large; if we use one consumer per partition, it serializes. We **fan out the fanout** like in the news-feed problem: the consumer reads `msg.created`, decomposes into per-recipient chunks, emits to a second topic `delivery.work` partitioned by `recipient_user_id`, processed by a wider worker pool.

### Decision: Redis for `connection_directory`

Alternatives: Cassandra (too slow for 100K reads/sec routing lookup), in-memory at gateway (each gateway only knows its own connections, fanout would have to broadcast).

Redis at sub-ms with TTL fits exactly. Loss is acceptable: if Redis loses a row, the device looks offline → message goes to inbox → on next heartbeat the row repopulates and live delivery resumes. Self-healing.

### Decision: Per-device offline queue (separate from messages table)

| Approach | Pros | Cons |
|---|---|---|
| **Single messages table; clients filter by `device_inbox` semantics on read** | One source of truth | Query "what does device D need" is a scatter-gather across all conversations the user is in |
| **✅ Per-device offline queue (`device_inbox` partitioned by device_id)** | Single read partition for catch-up; clean drain semantics | Write amplification (each message written to N device_inboxes) |

Write amp is acceptable because per-device-deliveries was already 65B/day in our estimation. We're just making the storage layout match the access pattern.

### Decision: Eventual consistency for message ordering, causal across conversation

| Scenario | Consistency model | Mechanism |
|---|---|---|
| Per-conversation order | Causal (and monotonic per device) | Snowflake message_id assigned by the conversation's Chat Service shard; Cassandra clustering key |
| Cross-conversation order | None (don't care) | Different conversations are independent timelines |
| Sender's own write read by sender's other devices | Read-your-writes | Fanout pushes to sender's other devices proactively |
| Read receipts | Eventual (≤5s) | Best-effort; if lost, recipient's "blue check" doesn't update — acceptable |
| Group membership change (add/remove member) | Eventual (≤30s) | Fanout reaches members async; key rotation amortized |

---

## 10. Defending architectural choices — interviewer attacks

> **Interviewer:** "Why not just store messages in DynamoDB? It's managed, scales linearly, no Cassandra ops team needed."

**Candidate:** Three reasons. (1) **Cost:** at 100B writes/day at 500B avg, DynamoDB on-demand is roughly $1.25 per million writes, so ~$125K/day = $45M/year for writes alone. Cassandra/Scylla self-hosted is closer to $5–8M/year for the same workload. (2) **Multi-cloud:** at 1B DAU, vendor lock-in is a strategic risk; we want the option to move to GCP or our own metal. (3) **Tunable consistency per query:** Dynamo's strong/eventual is binary; Cassandra lets us pick `LOCAL_QUORUM` for writes and `LOCAL_ONE` for hot reads on the same table. The price is operational — we need a Cassandra/Scylla team. If we had no such team and 5 SREs total, I'd flip to Dynamo despite the cost; engineering capacity is the bottleneck more often than dollars are.

> **Interviewer:** "Why a stateful Connection Gateway? Couldn't this be a stateless service that just brokers via Redis?"

**Candidate:** A WebSocket connection IS state — there's no way to make holding a TCP socket "stateless." You can move the state out of the gateway process (e.g., kernel holds the socket, multiple processes share it via fd-passing) but the **machine** is still stateful. Pretending otherwise leads to designs where every message goes through Redis pub/sub which doesn't scale to 50M concurrent fanout. The honest design is: connection state is on the gateway host; everything else is stateless. We minimize what state the gateway holds (just routing + auth context, no application state) and lose 50K users when one host dies — recoverable.

> **Interviewer:** "Why per-conversation ordering? Why not global ordering?"

**Candidate:** Global ordering is impossible at this scale (would require a single ordering oracle — the entire write throughput goes through one node) and unnecessary (no user cares whether their unrelated friend Bob's message to a random group came before or after their own message to a different group). Per-conversation order is what users perceive: "this reply came after the message it's replying to." That's preserved by partitioning Kafka by `conversation_id` (single consumer per partition → in-order processing) and Cassandra clustering by message_id (a Snowflake assigned by a single Chat Service shard, which is itself partitioned by conversation_id). The cost is that two messages in two different conversations have arbitrary global order — which I'd argue is correct, not a bug.

> **Interviewer:** "What's the celebrity threshold for messaging? You called it out for news feed but skipped it here."

**Candidate:** Different problem shape. In news feed, celebrity = millions of followers and a single post fanouts to them all. In messaging, "celebrity" = group size, and we already cap groups at 1024. The hard cap is the design lever. For broadcast lists / channels (which we explicitly scoped out), you'd switch to a pull model — broadcast posts go to a separate `channel_posts` table, subscribers pull on app-open. Conceptually identical to the news-feed celebrity story.

> **Interviewer:** "What if Kafka goes down?"

**Candidate:** Detection: producer error rate, consumer lag. Blast radius: messages keep being accepted by Chat Service (writes to Cassandra succeed) but **fanout stops** — recipients don't see new messages until Kafka recovers. Mitigation: (1) Chat Service has a local write-ahead log; if Kafka is down, Chat Service buffers `msg.created` events to local disk. On Kafka recovery, drains. (2) Cross-region failover: each region runs its own Kafka cluster; if one is dead, the regional Chat Service can't accept new messages until either Kafka recovers or we cut over to a standby cluster. Recovery time: aim for <5 min full recovery; data loss bound: zero (everything is on disk). The visible UX: messages sent during the outage show "pending" / single checkmark, deliver when Kafka is back.

> **Interviewer:** "What if Redis (`connection_directory`) goes down?"

**Candidate:** Fanout cannot route messages to live sockets. Falls back to writing all deliveries to `device_inbox` and triggering APNs/FCM. Recipients still get notifications and messages on next app-open / reconnect. Live-message UX is broken (no real-time push during the outage), but no message loss. Recovery: Redis cluster restart, gateways re-register their connections on next heartbeat (within 30s).

> **Interviewer:** "How do you handle abuse — someone scripting 1M messages/sec from one account?"

**Candidate:** Rate limit at multiple layers:
- **Per-device send rate** at the Connection Gateway: e.g., 30 msg/s burst, 5/s sustained.
- **Per-account send rate** at Chat Service: 1000/min.
- **Per-(sender, recipient) pair**: 100/min — prevents spamming a single victim.
- **Server-side spam classifier** on metadata (graph features: "this account just joined and is messaging 500 strangers/min"). Auto-shadowban.

Abuse signals also fed back into the rate-limit decision dynamically. The rate limiter is itself an interesting subsystem (token bucket per key, distributed via Redis with replicated counters), but I'd treat it as a known pattern.

> **Interviewer:** "How do you migrate this if WhatsApp existed and you were rebuilding?"

**Candidate:** Parallel build, dual-deliver, cut over. Concretely:
1. Build new system, point at production DBs (read-only) for read traffic.
2. Dark-launch: every write that goes to old WhatsApp also gets shadow-written to new system. Compare delivery latency, content fidelity. Run 4–6 weeks.
3. Per-region migration: smallest market first. Cut over write traffic; old system still reads old data for 90 days (rollback insurance).
4. Per-feature flag: 1:1 chat first (highest test coverage), groups second, multi-device third.
5. Decommission old after 6 months of clean operation.

The hardest part is **data migration** — old encrypted blobs in old format. Per-user re-key would require client cooperation; in practice you keep the old store readable, only new messages go to the new system.

---

## 11. Failure modes & mitigations

| Failure | Detection | Blast radius | Mitigation | Recovery |
|---|---|---|---|---|
| Connection Gateway host crash | Health check 5–10s | 50K users disconnected (~0.05% of region) | LB removes host; clients reconnect with jittered backoff; 30% headroom absorbs reconnect storm | Auto: clients reconnect to other hosts within 5–15s |
| Connection Gateway region failover | Region health probe | ~30M users; reconnect storm | DNS-level failover to nearest region; cross-region offline-queue fetch on reconnect | DNS TTL drives recovery (30–60s); full 5–10 min |
| Chat Service slowdown | p99 latency alarm | Messages take longer to ack; sender sees pending state | Auto-scale on QPS; circuit-breaker downstream | Add capacity; investigate (DB, GC, dependency) |
| Cassandra `messages` shard down | Per-shard latency / 5xx | 1/N of conversations have degraded reads/writes | RF=3 + LOCAL_QUORUM allows 1 down per shard; writes route to remaining replicas | Replace failed node, stream-rebuild (hours) |
| Kafka cluster lost | Producer error rate / consumer lag | New messages stop fanning out; recipients delayed | Chat Service local WAL buffers events; replay on recovery; multi-AZ Kafka prevents this | Multi-AZ Kafka makes this rare; if total loss, standby cluster <30 min |
| Redis `connection_directory` down | Cache hit rate / errors | Live-push routing fails; everything goes to offline path | All messages still delivered (via APNs + drain on reconnect); just no real-time during outage | Redis cluster restart; gateways re-register on heartbeat <30s |
| Redis `presence` down | Cache errors | Presence is stale or shows everyone offline | Acceptable degradation; don't block message flow | Restart; presence rebuilds on next heartbeat |
| APNs / FCM rate limits or outage | Push delivery error rate | Offline users don't get push notifications | Local-side device polls on app-open; no message loss, just no preemptive wakeup | APNs/FCM SLAs; multi-vendor strategy as backup |
| `device_inbox` partition hot (celebrity recipient) | Per-partition write rate | One Cassandra partition hammered; tail latency spike | Bucket key by time (`device_id || hour`); admission control on inbound rate per recipient | Auto-bucket + alert |
| Fanout consumer lag spike | Kafka lag metric | Messages visible to recipients delayed | Auto-scale workers; backpressure on Chat Service ingest if lag >5min | Drain backlog; investigate root cause |
| Multi-device key sync conflict | Client error reports | Some devices can't decrypt | "Reset session" flow: clients re-X3DH; show "security number changed" | User confirms; session re-established |
| New-device add storm | Group key rotation rate | Group conversations re-keying; spike in key-message volume | Batch rotations; lazy re-key on next-message-send | Self-heals; eventual within minutes |
| Schema change | Pre-deploy | Service crash on deploy | Forward+backward compatible (additive only); dual-version client support since 2B clients run different versions | Roll back; 2-version-deep wire compat |
| DDoS at edge | WAF + per-source rate | Connection establishment storm | Per-IP rate limit; Anycast spreads load; CAPTCHA on suspicious patterns | DDoS mitigation team |
| Catastrophic data loss in one region | Backup integrity check | Up to 30 days of message ciphertext lost (offline queues); active conversations recover via clients | Cross-region replication of `messages`; clients are source-of-truth for chat history beyond retention | Restore from cross-region replica or accept ciphertext loss (clients have plaintext) |

---

## 12. Productionization

### Pre-launch

- [ ] Dark launch: shadow real production traffic to new Chat / Fanout services for 4 weeks. Compare delivery latency p50/p99 vs current.
- [ ] Load test to 5× peak (5M msg/s globally). Verify Connection Gateway can hold 100M+ concurrent.
- [ ] Chaos test: kill random Cassandra nodes, gateway hosts, Kafka brokers under load. Verify recovery + zero message loss.
- [ ] Game day: simulate full Kafka outage; verify Chat Service WAL drains correctly on recovery.
- [ ] Game day: simulate region failover; verify reconnect storm absorbed without DNS thrash.
- [ ] Penetration test of E2EE implementation (independent firm); publish white paper for trust.

### Rollout

- [ ] Feature flag per region. Smallest market first (e.g., a single small country).
- [ ] 1% of users in market → 5% → 25% → 100%, 24h bake at each step.
- [ ] Auto-rollback if: send→deliver p99 > 1s for 5 min, message loss rate > 1 in 1M, or 5xx > 0.5%.
- [ ] **Client version gating:** because we have 2B clients running different versions, server must support N-2 wire format always. New features are additive; deprecate after ≥6 months of telemetry showing <0.5% client adoption.

### Capacity

- [ ] Reserve 50% headroom on Connection Gateway fleet. Reconnect storms need it.
- [ ] Auto-scale Chat Service / Fanout on QPS; target 50% CPU.
- [ ] Cassandra: provision for 3× current write volume; node-add takes hours.
- [ ] Kafka: 5× partition count vs current peak; repartitioning is expensive.
- [ ] Per-region capacity budget; rebalance quarterly with growth projections.

### Migration considerations (if rebuilding)

- [ ] Dual-write to old + new for 4–6 weeks.
- [ ] Per-conversation cutover with shadow comparison.
- [ ] Old system retains ciphertext for 90 days as rollback insurance.
- [ ] Decommission old after stable operation.

### Cost

- Estimated $/DAU: roughly $0.005–0.01/DAU/month for storage + compute (industry public numbers; WhatsApp historically very efficient at <$0.01/DAU). Bandwidth dominates if media isn't cached.
- Cost levers:
  - Aggressive 30-day TTL on offline queue (we don't archive)
  - Compress envelope ciphertext at storage layer (already random-looking; ~20% gain on metadata fields only)
  - Per-region Kafka with 24h retention on `msg.created` (don't need replay window beyond a day)
  - Tier-down media to glacial after 90 days; clients cache locally
  - Sample (not full) telemetry on inactive users

### Compliance

- GDPR right-to-erasure: cascade delete to `users`, `devices`, `conversation_members`, `read_watermarks`. `messages` partition by conversation_id — delete user's contributions selectively (write tombstones; readers filter). Cleanup sweep is async.
- Data residency: per cell. EU users' messages stored in EU cell. Cross-cell delivery (US user messaging EU user) — see section 19.
- Audit log: every admin action (account suspension, key rotation forced) logged immutably.
- Lawful intercept: we cannot decrypt content (E2EE). We can produce metadata under valid legal request — this is a known capability and product-positioned in our privacy whitepaper.

---

## 13. Monitoring & metrics

### Layer 1 — Business metrics

- DAU, MAU, retention
- Messages sent per DAU
- Group sends per DAU
- Devices per user (median, p99)
- "Activation": % of users who send a message within 24h of install
- Cross-cell message rate (geographic indicator)

### Layer 2 — SLIs

**Front-door SLOs:**
- **Send → ack to sender**: p50 < 100ms, p99 < 300ms (intra-region)
- **Send → deliver to all online recipients**: p50 < 200ms, p99 < 500ms
- **Send → deliver including offline path** (catch-up): p99 < 5s after reconnect
- **Message loss rate**: target <1 in 100M
- **Connection Gateway availability**: 99.99% per region

Per-endpoint, per-region, per-conversation-type (1:1 vs group) breakdowns.

### Layer 3 — Component metrics

#### Connection Gateway
- Concurrent connection count per host
- Connect rate, disconnect rate
- TLS handshake p99
- Heartbeat health (% of conns with <60s last-heartbeat)
- Bytes/sec ingress + egress
- File descriptor utilization
- Per-host memory (TLS state)

#### Chat Service
- Submit rate, error rate, p99 latency
- Cassandra write latency (p99)
- Kafka publish latency
- Per-conversation rate-limit rejections

#### Fanout Service
- Kafka consumer lag (alert if > 30s)
- Per-message fanout completion latency (p50, p99)
- Push success rate (live socket delivery)
- Inbox write rate
- APNs/FCM submit rate + error rate

#### Cassandra
- Per-keyspace read/write latency (p99)
- Replication lag (cross-DC)
- Compaction queue depth
- Hint queue
- Disk utilization
- Per-shard write skew (detect hot conversations)

#### Redis (presence + connection_directory)
- Hit rate
- Eviction rate
- Memory utilization
- Replication lag
- Slow log

#### Kafka
- Producer publish rate, error rate
- Consumer lag per group
- Broker disk/network utilization
- Partition skew (detect hot conversations)

#### Key Server
- Prekey-bundle fetch rate
- One-time prekey exhaustion (per user)
- Identity-key change events (potential MITM signal — should be rare)

### Layer 4 — Infrastructure

- CPU, memory, disk, network per host
- Kernel TCP stats: retransmit rate, time-wait socket count, ephemeral port exhaustion
- Per-AZ network packet loss
- Cross-region RTT

### Alert routing

- **Page (24/7):** SLO burn rate (1h fast-burn, 6h slow-burn), regional gateway outage, Kafka lag > 5 min, message loss > 1 in 1M, Cassandra cluster degraded
- **Ticket (next business day):** cache hit rate degraded, single-host Cassandra issue, prekey exhaustion for individual users
- **Dashboard-only:** cost trends, capacity headroom, growth rate

### Dashboards

1. **Front-door health** — RED metrics, p99 send→deliver, by region
2. **Connection Gateway** — concurrent count, connect/disconnect rate, hot hosts
3. **Fanout pipeline** — Kafka lag, per-message fanout latency, inbox write rate
4. **Cassandra** — per-cluster, per-shard, hot partitions
5. **Business** — DAU, msgs/DAU, retention, group sends share

---

## 14. Security & privacy

- **E2EE:** Signal Protocol; server stores ciphertext only; private keys never leave devices.
- **Identity verification:** safety numbers / QR-code verification for users who want assurance against MITM by us.
- **Forward secrecy** (Double Ratchet): compromise of current key doesn't expose past messages.
- **Post-compromise security** (DH ratchet advances): new key after every round-trip.
- **Authn:** mTLS service-to-service; client auth via long-lived session token + device public key.
- **Authz:** every send is auth'd against `conversation_members[conversation_id]` — drop messages from non-members.
- **Spam:** rate limits per device + per account + per (sender, recipient) pair; ML on graph features.
- **Lawful intercept:** we provide metadata (who messaged whom, when) under valid legal request. We cannot provide content.
- **PII:** phone numbers stored as salted hashes for contact discovery; never log plaintext.
- **Audit log:** admin actions immutable, retained 7 years.
- **Compliance:** GDPR (EU), DPDP (India), LGPD (Brazil), CCPA (California). Data residency per cell — see section 19.
- **DDoS:** WAF + per-IP rate limit at edge; Anycast spreads attack surface; CAPTCHA on connect for suspicious patterns.
- **Account takeover:** anomaly detection on device-link events, geographic logins, sudden message volume changes.

---

## 15. Cost analysis

| Component | Rough cost driver | Optimization lever |
|---|---|---|
| Connection Gateway (~10K hosts global) | Compute + bandwidth | Hot reload to avoid reconnect storms; right-size hosts (50K conns/host) |
| Cassandra (`messages`, `device_inbox`) | Storage + replication × 30-day retention | TTL aggressive; no long-term archive |
| Cassandra (`read_watermarks`, `users`, `devices`) | Storage (smaller) | Standard |
| Kafka (`msg.created`, `delivery.work`) | Disk + retention | 24h retention max; cheap |
| Redis (presence, connection_directory) | RAM | TTL 30–60s; size for peak |
| Object storage (media) | Bytes stored × duration | 30-day hot, 90-day warm, glacial after; client-side cache |
| CDN egress (media downloads) | Bytes served | Compression (webp, avif, h264 adaptive); pre-warm popular media |
| APNs / FCM | Free for us; Apple/Google charge themselves | Batching; coalesce within 5s window |
| Compute (Chat Service, Fanout, Key Server) | CPU-hours | Autoscale; reserved instances for baseline |

**Largest cost: Connection Gateway compute + CDN egress** for media. Engineering effort goes furthest there.

**$/DAU rough estimate:** $0.005–0.01/DAU/month. WhatsApp historically operated at <$0.01/DAU which is industry-leading; our design should not exceed 2× that.

---

## 16. Open questions / what I'd validate with PM

1. **Multi-device cap:** is 4 the right number? iMessage allows ~10. Each device adds key-distribution overhead and inbox cost. PM tradeoff.
2. **Group size cap:** 1024 vs Telegram's 200K vs Discord's millions. Architecturally different above ~1K (becomes pub/sub, not chat). Confirm scope.
3. **Server-side message history:** today we drop ciphertext at 30 days. iCloud-backup model would store encrypted backups (server holds ciphertext, user holds key). Privacy tradeoff: more recoverable but more attractive attack surface.
4. **Read-receipt default:** on or off? Affects volume (4× difference) and privacy norms in different markets.
5. **Channel / broadcast feature:** in scope eventually? Different architecture — pull-based.
6. **Voice/video calls:** when? Affects which signaling layer to invest in now (this design's WSS works for SDP exchange).
7. **Client-side ML moderation:** are we shipping on-device classifiers? Affects client app size, which affects emerging-market install rate.
8. **Cross-cell residency policy per market:** which countries strictly require local storage; which allow cross-cell with consent.

---

## 17. Staff-level scorecard for this problem

| Signal | Did the candidate... |
|---|---|
| ✅ Identified per-device fanout (not per-user) as the central insight | Within first 15 min, motivated by E2EE and multi-device |
| ✅ Quantified concurrent connections | "80M conns/region → 1,600 gateway hosts" calculation |
| ✅ Picked databases with defended reasons | Cassandra justified by access pattern, not "because it scales" |
| ✅ Treated E2EE as a first-class constraint | Explained which features become impossible (search, ML moderation, content-rich notifications) |
| ✅ Volunteered failure modes before being asked | Mentioned gateway crash, Kafka outage, Redis loss, key conflicts |
| ✅ Designed for graceful degradation | "Live push fails → all-offline path → messages still deliver via APNs" |
| ✅ Talked about cost in concrete terms | $/DAU number, identified gateway + CDN as dominant |
| ✅ Discussed productionization unprompted | Hot reload, deploy strategies, schema versioning across 2B clients |
| ✅ Asked the interviewer at decision points | "Calls in scope?" "E2EE required?" — drove the scope |
| ✅ Closed with a wrap-up speech | Restated dominant constraints, choices, what to do next |

### What separates Staff from Senior on *this* problem

A Senior candidate will:
- Correctly draw the connection gateway + chat service + fanout architecture
- Mention WebSocket and offline queues
- Pick reasonable databases
- Acknowledge multi-device exists

A Staff candidate will additionally:
- Lead with **"per-device fanout, not per-user"** as the load-bearing decision
- Discuss the **operational cost of stateful gateways** (deploys, hot reload, drain protocols) — most candidates skip this
- Connect E2EE to specific feature *impossibilities* (no server search, no content-aware push) and own the tradeoff
- Quantify the **receipt firehose** (1.3T events/day) and design `read_watermarks` separately
- Identify that fanout itself needs internal fanout for large groups (chunked work topic)
- Discuss **client version compatibility across 2B clients** as a real schema-evolution constraint
- Volunteer the **deploy / hot-reload** problem unprompted
- Frame **presence as best-effort** and propose coarser granularity rather than building millisecond-accurate at scale
- Discuss **multi-region active-active with cross-cell residency** rather than pretend it's one global cluster
- Explicitly cover what the system **cannot** do because of E2EE — and turn it into a positioning advantage

---

## 18. Wrap-up speech (template)

> "To wrap up: the dominant constraints here are (1) ~50M+ concurrent persistent connections per region — making the Connection Gateway the most operationally fragile component, (2) per-device fanout — because E2EE encrypts per-device-keypair and offline state is per-device, and (3) E2EE itself constraining what the server can do. The architecture answers all three: stateful Connection Gateway sharded by user_id with 30% headroom for reconnect storms; Cassandra-backed per-device offline queues drained on reconnect with Kafka-orchestrated fanout; server stores only ciphertext, prekey bundles, and routing metadata. The two pieces I'd worry about in production are (1) gateway deploys causing reconnect storms — mitigated by graceful drain protocols and ideally hot-reload with fd-passing, and (2) hot conversation partitions in Cassandra during viral group events — mitigated by per-recipient bucketing and admission control. To productionize: per-region feature-flagged rollout starting at 1% in the smallest market, dark-launched against existing system for 4 weeks, SLO-alerted on send→deliver p99 and Kafka fanout lag. I'd also commission an independent E2EE pen-test and publish a privacy whitepaper before 100% rollout. If we had another 30 minutes, I'd deep-dive (a) cross-cell delivery for residency-restricted markets — selective replication vs federation — or (b) the abuse / spam classifier given we cannot read content."

---

## 19. Market-based isolation (cellular architecture) — bonus / "if time permits"

> Bring this up in the last 5 minutes. For chat at WhatsApp scale, this is not optional — data residency and blast-radius reduction are existential. Frame it as: *"If we had more time, the next architectural layer I'd add is cellular isolation by market — but for chat, the cross-cell story is more interesting than for news feed because messages are interpersonal."*

### What & why

A **market** is a geographic + regulatory unit (US, EU, India, Brazil, Japan; China usually excluded — different product entirely). A **cell** is a self-contained per-market replica: its own Connection Gateway fleet, its own Chat Service, its own Cassandra cluster, its own Kafka, its own Key Server.

Drivers, in priority order:

1. **Compliance / data residency** — GDPR (EU), India's DPDP, Brazil's LGPD all require local data storage. Hard requirement.
2. **Latency** — Indian users connect to Mumbai cell (~30ms RTT), not Virginia (~250ms). For chat, every 100ms of additional latency is felt.
3. **Blast radius** — bad config push in EU cell does not blow up US users.
4. **Operational independence** — markets with different growth curves (India growing 30% YoY, US flat) don't fight for capacity.
5. **Cost attribution** — per-market $/DAU.
6. **Feature differentiation** — India cell may use lower-bitrate media defaults; EU cell stricter on data minimization defaults.

### Architecture

```
        ┌─────────────────────── Global Edge ───────────────────────┐
        │  GeoDNS + Anycast → user routes to nearest cell            │
        └────────┬────────────────┬─────────────────┬────────────────┘
                 │                │                 │
       ┌─────────▼────────┐ ┌─────▼────────┐ ┌─────▼────────┐
       │   US Cell        │ │   EU Cell    │ │  IN Cell     │
       │ ┌──────────────┐ │ │┌────────────┐│ │┌────────────┐│
       │ │ Conn GW fleet│ │ ││ Conn GW    ││ ││ Conn GW    ││
       │ │ Chat Svc     │ │ ││ Chat Svc   ││ ││ Chat Svc   ││
       │ │ Fanout Svc   │ │ ││ Fanout     ││ ││ Fanout     ││
       │ │ Cassandra    │ │ ││ Cassandra  ││ ││ Cassandra  ││
       │ │ Kafka        │ │ ││ Kafka      ││ ││ Kafka      ││
       │ │ Redis        │ │ ││ Redis      ││ ││ Redis      ││
       │ │ Key Server   │ │ ││ Key Server ││ ││ Key Server ││
       │ └──────────────┘ │ │└────────────┘│ │└────────────┘│
       └─────────┬────────┘ └──────┬───────┘ └──────┬───────┘
                 │                  │                 │
                 └──────────────────┴─────────────────┘
                              │
                  ┌───────────▼───────────┐
                  │  Cross-Cell Bridge    │
                  │  - User → home cell   │
                  │    directory          │
                  │  - Cross-cell delivery│
                  │  - Selective metadata │
                  │    replication        │
                  └───────────────────────┘
```

### Routing: how a request lands in the right cell

- Each user has a **home cell** assigned at signup (based on phone-number country, with opt-in migration on travel).
- A small global **User Directory** (replicated KV) maps `user_id → home_cell`. Cached aggressively at the edge (TTL 5 min).
- Connection Gateway in the user's home cell is where the WSS lives.
- A user roaming temporarily (US user in Tokyo for a week) still connects to their **home cell** — we don't route them to the closer cell, because their data is not there. Latency cost is accepted.

### The hard part: cross-cell messaging

This is the central question for chat (it wasn't for news feed).

**User A is in US cell. User B is in EU cell. A sends a message to B.**

Three approaches:

| Approach | How | Pros | Cons |
|---|---|---|---|
| **Full conversation replication** | Conversation lives in BOTH cells; every message replicated both ways | Reads always local; simple read-path | 2× storage; replication lag affects "delivered" semantics; violates strict residency interpretations |
| **Selective replication (conversation metadata only)** | Conversation metadata replicated; messages live in originating cell; recipient's cell pulls on demand or via push | Less storage duplication; respects residency by partial scope | Cross-cell read on every fetch; complex routing |
| **Federation (cell-to-cell push)** | Originating cell pushes message to recipient's home cell over a Cross-Cell Bridge; recipient's cell stores in their `device_inbox` | Each cell only stores its own users' inboxes; clean residency | New service (Bridge); cross-cell latency in delivery path; more failure surface |

**My pick for chat: federation.** Reasoning:
- For 1:1 messaging, "the conversation" is already two-sided — message lives in both endpoints' inboxes anyway. Federation just makes that explicit at the cell boundary.
- For groups with mixed-cell membership, the originating cell fans out to local members directly + crosses the Bridge once per remote-cell-member-set with a single payload. The remote cell expands to its own members.
- E2EE makes this clean: ciphertexts are opaque blobs; cells just route them; no plaintext crosses borders.
- Storage cost: each cell stores only its own users' inboxes. Sender's home cell stores the canonical `messages[conversation_id]` row (or both cells store, but encrypted with keys belonging to local users).

The tradeoff: cross-cell delivery latency adds 80–150ms (trans-Atlantic RTT) + Bridge processing (~50ms). Acceptable for cross-cell, since users intuitively expect international DMs to be slightly slower.

### Where conversations live

Two patterns:

- **1:1 conversation between US user A and EU user B:** `conversation_id` is anchored to A's cell (originator). Messages stored canonically there. Replicated to B's cell as B's inbox entries (encrypted for B's devices). If GDPR requires deletion of B's data: B's cell deletes B's inbox entries; A's canonical record remains (it's A's data). Subtle but workable.
- **Group with mixed-cell members:** group is anchored to the cell of the **creator** (or migrated explicitly on user request). Members in other cells receive cross-cell deliveries. Group metadata replicated as needed.

### Cell tradeoffs vs single global system

| Dimension | Global | Cellular | Net |
|---|---|---|---|
| Total infra cost | Lower (no duplication) | +20–40% | Cellular tax |
| Latency for in-cell messages | Higher | Lower | Cellular wins |
| Latency for cross-cell | N/A | Bridge adds 80–150ms | Cellular pays |
| Compliance (residency) | Hard | Easier | Cellular wins (often non-negotiable) |
| Blast radius | Global | Per-cell | Cellular wins |
| Engineering complexity | Lower | Higher (cell-aware everywhere) | Cellular tax |
| Cross-cell consistency | N/A | New problem | Cellular pays |

### What changes in the design with cells

- **User Directory** — new global KV, replicated, sub-ms reads at edge. Caches `user_id → home_cell`.
- **Cross-Cell Bridge** — new service; owns SLAs for cross-cell delivery latency. Runs at ~5% of total message volume.
- **Per-cell Kafka** — `msg.created` partitioned within cell. Cross-cell events flow through Bridge into target-cell Kafka.
- **Per-cell Key Server** — prekeys stored per-cell; cross-cell sender fetches recipient's prekey from recipient's cell via Bridge.
- **Per-cell ML / abuse models** — different markets have different abuse patterns.
- **Per-cell rollouts** — feature flags now have a `cell` dimension.
- **Failover policy:** if EU cell goes down, do we route EU users to US cell? **No, for residency reasons.** Cell IS the failure domain; intra-cell HA is the protection.

### Operational implications

- Deploy pipelines per cell. CI/CD must be cell-aware.
- On-call per cell, often per-team-spanning-cells with cell-aware paging.
- Capacity planning per cell — India may need to grow gateway fleet 3× faster than US.
- Cost dashboards per cell — every team sees $/DAU per cell.
- Schema migrations per cell — staggered.

### When NOT to do cellular for chat

- Pre-scale: under ~30M users globally, the operational tax exceeds the benefit.
- No regulatory pressure (rare for chat — most jurisdictions have at least metadata regulations).
- Single team, no capacity to operate N cells.

### What "Staff signal" sounds like here

> "I'd structure this as cellular from day one. Chat is interpersonal — every message is potentially cross-cell — so the Bridge is a load-bearing subsystem, not an afterthought. I'd default to **federation** over replication: each cell stores its own users' inboxes, messages cross via the Bridge as encrypted blobs, and residency is enforced by where the inbox lives, not by complex policy. The tax is real — 20–40% more infra, a permanent complexity surcharge, +80–150ms on cross-cell delivery — but it's the only design that survives a GDPR audit, an Indian DPDP audit, and a regional cloud outage simultaneously. The cell IS the unit of blast-radius isolation, deploy, capacity, and compliance — and for chat specifically, the federation protocol over the Cross-Cell Bridge is the most interesting subsystem to deep-dive on next."
