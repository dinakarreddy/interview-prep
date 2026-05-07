# 10 — Design a Multi-Channel Notification System

> The Staff signal here isn't building a queue and a worker. It's identifying that **(1) channels have radically different economics, latencies, and failure modes**, and **(2) user preferences + dedup + quiet hours are the actual hard problems** — not the send pipeline. Spend your time there.

---

## 0. Problem statement (as the interviewer would drop it)

> **Interviewer:** "Design a notification system for a large consumer app. Anything in the product that needs to alert a user — order updates, marketing pushes, password resets, friend requests — goes through this system. It should support push notifications, email, SMS, and in-app. Walk me through how you'd build it."

That's the prompt. Two traps to avoid:
1. Don't draw a queue and three workers and call it done — that's Senior, not Staff.
2. Don't pick one channel and design for it. The whole point is the abstraction.

---

## 1. Clarifying questions

> **Candidate:** "Before I design, let me scope. The space is broad — I want to make sure we're solving the right slice."

> 1. **"Who's the caller — internal services, marketing teams, end users?"** — *Expect: internal services (transactional like 2FA / order updates) AND marketing systems (campaigns). Different SLOs.*
> 2. **"Is the caller picking the channel, or are we deciding?"** — *Critical. Expect: we decide based on user preferences + urgency. Caller specifies intent, not channel.*
> 3. **"What's the rough scale — DAU and notifications per user per day?"** — *Expect: 200M DAU, ~5 notifications/user/day average → 1B/day. Marketing campaigns spike 10×.*
> 4. **"What's the channel mix?"** — *Expect: 70% push, 15% email, 5% SMS, 10% in-app. Drives where we focus.*
> 5. **"Is content/templating in scope, or do callers send pre-rendered messages?"** — *Expect: templating IS in scope (with localization). Callers send template_id + variables, not strings.*
> 6. **"Do we need to handle quiet hours and timezones?"** — *Yes — 3am push notifications are a top user complaint and a top source of opt-outs.*
> 7. **"What's the compliance posture — GDPR, CASL, TCPA?"** — *Hard requirement. Affects opt-in/out semantics, audit logging, regional data routing.*

> **Out of scope (state explicitly):**
> - Marketing campaign UI (audience builder, copy editor) — upstream system. We accept "send to this user list" as input.
> - Realtime chat (different problem — see `02-whatsapp.md`).
> - Content/copywriting tools.
> - Voice calls (out — separate IVR system).
> - The notification *display* on mobile (OS-controlled).

> **Candidate:** "I'll design the **send pipeline** end-to-end: API surface for callers, channel-routing logic with preferences, the per-channel send workers and their provider integrations, dedup/throttling/quiet hours, and the click/open tracking pipeline. Sound right?"

> **Interviewer:** "Yes."

---

## 2. Functional requirements

| Priority | Requirement |
|---|---|
| P0 | Caller submits a notification request with `user_id` + `template_id` + `variables` + `category` (transactional vs marketing) |
| P0 | System picks the right channel(s) based on user preferences and category |
| P0 | Push delivery via APNs (iOS) and FCM (Android) |
| P0 | Email delivery via SES / SendGrid |
| P0 | SMS delivery via Twilio |
| P0 | In-app notifications (delivered next time the user opens the app) |
| P0 | User preferences: opt-in/out per category, per channel |
| P0 | Quiet hours: respect user timezone; never push 9pm–8am local unless transactional override |
| P0 | Deduplication: same logical notification within window must not fire twice |
| P0 | Throttling: per-user cap (don't spam); per-provider rate limit |
| P0 | Templating with variable substitution and localization (50+ languages) |
| P1 | Click and open tracking |
| P1 | Bounce / complaint handling (auto-suppress bad email/phone) |
| P1 | Scheduled / future-dated sends (campaigns) |
| P1 | Cross-channel fallback (push fails → SMS for transactional) |
| P2 | A/B testing of templates |
| Out | Marketing campaign UI, content authoring, voice calls, web-push (relegated to push channel) |

---

## 3. Non-functional requirements

| Dimension | Target | Justification |
|---|---|---|
| Availability | 99.99% for transactional path; 99.9% for marketing path | Transactional includes 2FA/security — losing it is a P0 incident |
| Send latency (transactional) | p99 < 5s from API ingress to provider acknowledged | 2FA codes are useless after 30s; users perceive >10s as broken |
| Send latency (marketing) | minutes are fine; SLO p99 < 5 min | No user expectation of immediacy |
| Dedup window | 24h default, configurable per category | Long enough to absorb typical retry patterns |
| Throughput | 1B/day average; 100M/hour peak (10× campaign spike) | Industry-realistic for 200M DAU |
| Durability | Transactional: zero loss after 200 OK. Marketing: <0.01% loss tolerable | Marketing batches are huge; perfect durability would 10× cost |
| Consistency | Eventual for preferences cache (≤1 min staleness); strong for opt-out writes | Users tolerate brief preference cache lag; opt-out must apply immediately or compliance breach |
| Compliance | GDPR, CASL, TCPA, CAN-SPAM. Per-region data residency | Hard regulatory requirements |
| Cost target | <$0.001 per notification blended | Achievable if push-dominant channel mix holds |

---

## 4. Capacity estimation

> **Candidate:** "Round numbers — let me derive the workload."

### Volume
- 200M DAU × 5 notifications/user/day = **1B notifications/day**
- Per second avg: 1B / 86,400 ≈ **11.5K notifications/sec**
- Marketing spike (10× during campaigns): **100M/hour ≈ 28K/sec sustained for an hour**
- Peak burst (campaign send-all-at-once, 10 min window): ~150K notifications/sec

### Channel mix (at the daily blended ratio)
- Push: 700M/day (8K/sec avg) — dominant by volume
- Email: 150M/day (1.7K/sec avg)
- SMS: 50M/day (580/sec avg)
- In-app: 100M/day (1.2K/sec avg)

### Provider rate-limit reality check
- **APNs HTTP/2:** ~10K notifications/sec **per connection**, recommend 100s of connections per provider account → effectively unlimited if we manage connection pools right. Constraint is connection management, not raw throughput.
- **FCM HTTP v1:** rate-limited per-project; multi-project sharding is standard. ~10K/sec/project comfortable.
- **SES:** account-level send rate — 14/sec default, ramps to 1000s/sec after warm-up. We negotiate dedicated IPs.
- **Twilio SMS:** per long-code 1 msg/sec, per short-code ~100 msg/sec; toll-free can do 3 msg/sec. We use **shortcode + 10DLC** mix; need many sender numbers to absorb volume.

### Storage
- **Notification log (Cassandra):** every notification recorded for audit + dedup + analytics. ~500 bytes/record → 500 GB/day. With RF=3 and 90-day retention: ~135 TB. Tier older data to S3.
- **User preferences (Cassandra ground truth):** ~1B users × ~2 KB/user = 2 TB. Hot subset (200M DAU) cached in Redis: ~400 GB cache.
- **Templates (Git → Redis):** ~10K templates × 50 locales × 5 KB = 2.5 GB. Trivially fits in Redis.
- **Dedup index (Redis):** keyed by `idempotency_key`. ~1B keys × 60 bytes × 24h TTL = 60 GB working set.
- **Click/open events (Druid):** ~10% of notifications produce events. 100M/day × 1 KB → 100 GB/day; aggregated, retained 13 months → ~3 TB.

### Cost (back-of-envelope)
| Channel | Volume/day | Unit cost | Daily cost |
|---|---|---|---|
| Push (APNs/FCM) | 700M | ~$0 (just bandwidth) | ~$200 |
| Email (SES) | 150M | $0.0001 | $15K |
| SMS (Twilio) | 50M | $0.005 | **$250K** |
| In-app | 100M | ~$0 | ~$50 |
| **Total** | 1B | — | **~$265K/day = ~$100M/yr** |

**Punchline:** SMS is 1000× more expensive per message than push. **Cost dominator is SMS.** Routing the same logical notification via push when possible saves money 1000:1. This is a load-bearing fact in this design.

### Bandwidth
- APNs payload ≤ 4 KB; avg ~1 KB. 8K/sec × 1KB = 8 MB/s outbound. Trivial.
- Email avg 50 KB rendered. 1.7K/sec × 50KB = 85 MB/s outbound. Significant but not the bottleneck (provider eats it).
- SMS 160 chars. Negligible.

---

## 5. API design

> **Candidate:** "Public surface — caller-facing only. No channel in the request; we decide."

### Submit a notification
```
POST /v1/notifications
  body: {
    user_id:           "u_123",
    template_id:       "order.delivered",
    variables:         { order_id: "ord_456", eta: "2026-05-07T19:00Z" },
    category:          "transactional",        // or "marketing", "social", "security"
    idempotency_key:   "order_delivered_ord_456_attempt_1",
    priority:          "high",                 // high | normal | low
    channel_hint:      null,                   // optional; we may override
    expires_at:        "2026-05-07T20:00Z",    // don't send after this; useful for time-sensitive
    locale:            "en-US",                // optional; falls back to user pref
  }
→ 202 Accepted { notification_id: "nt_789", status: "queued" }
```

Notes:
- **`category` drives policy.** Transactional bypasses quiet hours and most preference checks (compliance-permitted exceptions: 2FA, security, fraud).
- **`idempotency_key` is the dedup primary key.** Caller-generated and stable across retries. We dedup on `(user_id, idempotency_key)` for 24h.
- **`channel_hint`** is advisory only. The system reserves the right to override based on preferences/availability.
- **`expires_at`** lets a stale 2FA code be dropped instead of delivered late.
- **202, not 201:** the call is async-accepted, not done. We return a `notification_id` for status lookups.

### Bulk submit (campaigns)
```
POST /v1/notifications:bulk
  body: {
    template_id, variables_per_user[], category, send_at, ...
  }
→ 202 { batch_id, accepted_count }
```
Batch endpoint enables one campaign call to enqueue 50M notifications without 50M HTTP round-trips.

### Query status
```
GET /v1/notifications/{notification_id}
→ 200 { status: "delivered" | "queued" | "throttled" | "failed" | "dropped",
        channel: "push", attempts: 1, delivered_at: "...", failure_reason: null }
```

### User preferences
```
GET /v1/users/{user_id}/preferences
→ {
    channels:   { push: true, email: true, sms: false, in_app: true },
    categories: { transactional: { all: true },
                  marketing:     { email: true, push: false, sms: false },
                  social:        { all: true } },
    quiet_hours: { enabled: true, start: "21:00", end: "08:00", timezone: "America/Los_Angeles" },
    locale: "en-US",
  }

PUT /v1/users/{user_id}/preferences { ... }
```

### Click / open tracking (server-to-server from client SDK)
```
POST /v1/events { notification_id, event_type: "open" | "click" | "dismiss", timestamp }
```

### Webhooks for inbound provider events
```
POST /webhooks/ses/bounce       (SES SNS callback)
POST /webhooks/twilio/status    (Twilio status callback)
POST /webhooks/fcm/feedback     (rare — invalid token notifications)
```

### Anti-patterns to avoid
- ❌ Don't expose a `channel` field as a hard send target — caller picks the channel and you've defeated the abstraction.
- ❌ Don't make this synchronous (return 200 only after delivery). Provider latency is unbounded.
- ❌ Don't accept rendered strings — accept `template_id + variables`. Rendering at the caller is a security/i18n nightmare.
- ❌ Don't return user PII (phone, email) in any response. Address is internal.

---

## 6. Data model

### `notifications` (Cassandra; partition by user_id, clustering by created_at desc)
| Column | Type | Notes |
|---|---|---|
| user_id | bigint | partition key |
| notification_id | snowflake | clustering key, time-sortable |
| idempotency_key | string | for dedup; secondary lookup table by `(user_id, idempotency_key)` |
| template_id | string | |
| category | enum | transactional / marketing / social / security |
| channel | enum | actual channel used (decided by router) |
| status | enum | queued / sent / delivered / failed / dropped / throttled |
| attempts | int | retry count |
| created_at | timestamp | |
| sent_at | timestamp | |
| delivered_at | timestamp | |
| failure_code | string | provider-specific |
| variables_blob | json | rendered at send time, kept for audit |

Why Cassandra:
- Write-heavy (1B inserts/day, status updates).
- Partition by user_id gives "all notifications for user X" as a single-partition read — useful for in-app inbox and user support.
- Wide-row pattern matches the workload. Don't need joins on this hot path.

### `user_preferences` (Cassandra ground truth + Redis hot cache)
| Column | Type | Notes |
|---|---|---|
| user_id | bigint | PK |
| channels_enabled | json | per-channel boolean |
| category_prefs | json | nested per-category, per-channel |
| quiet_hours | json | start/end/timezone |
| locale | string | BCP-47 |
| device_tokens | list<json> | APNs/FCM tokens with platform + last_seen |
| email | string (encrypted) | |
| phone_e164 | string (encrypted) | |
| suppression_list | set<channel> | auto-populated from bounces/complaints |
| updated_at | timestamp | for cache invalidation |

Redis cache: `prefs:{user_id}` → JSON blob, TTL 5 min. Invalidated on write via pub/sub.

### `dedup_index` (Redis)
- Key: `dedup:{user_id}:{idempotency_key}`
- Value: `notification_id` (so we can return the original)
- TTL: 24h (configurable per category)

### `templates` (Git → Redis)
- Source of truth: Git repo (`templates/`). Reviewed via PR.
- Pushed to Redis on deploy: `tmpl:{template_id}:{locale}` → `{ subject, body, push_alert, push_data, sms, in_app, ... }` JSON.
- TTL: forever (manually invalidated on template version bump).

### `device_registry` (Cassandra; partition by user_id)
- Each user has 1+ devices (iOS, Android, web). Each has:
  - `token` (APNs token or FCM token)
  - `platform`
  - `app_version`
  - `last_seen`
  - `bad_token` flag (set on `unregistered` callbacks)

### `events` (Kafka → Druid)
- Stream: every send + every open/click/bounce/complaint/dismiss
- Druid for analytics queries (delivery rate by template, opt-out funnel, etc.)

### `throttle_counters` (Redis with sliding window)
- `throttle:{user_id}:daily` → counter, expires at midnight user-local
- `throttle:{user_id}:{category}:hourly` → fine-grained for noisy categories
- Cap: e.g., 10 marketing/day, 50 total/day (configurable).

### Why these choices
- **Cassandra for notification log + preferences:** write-heavy, partitioned access patterns, eventual consistency tolerable.
- **Redis for dedup, prefs cache, throttle counters:** sub-ms reads on the hot path; we read 3–4 Redis keys per send and we need that to be fast.
- **Git-backed templates:** templates are code-reviewed, version-controlled, deployable in stages. Enabling marketers to edit them in a UI without engineer review is a leading cause of incidents (template injection, rendering errors).
- **Druid for events:** OLAP queries (delivery rate by template by region by day) at scale, sub-second.

---

## 7. High-level architecture

```
                  ┌─────────────────────────────────────────────────────────┐
                  │                Callers (microservices, batch)           │
                  └────────────────────────┬────────────────────────────────┘
                                           │ POST /v1/notifications
                                           ▼
                                ┌──────────────────────┐
                                │  Notification API    │  authn, rate limit, schema validate
                                │  (stateless, HA)     │
                                └─────────┬────────────┘
                                          │
                                          ▼
                                ┌──────────────────────────┐
                                │  Validation & Dedup Svc  │
                                │  - template lookup       │
                                │  - dedup check (Redis)   │
                                │  - prefs check (Redis)   │
                                │  - quiet-hours check     │
                                │  - throttle check        │
                                └─────────┬────────────────┘
                                          │
                                          ▼
                                ┌──────────────────────────┐
                                │  Channel Router          │
                                │  - decide channel(s)     │
                                │  - decide schedule       │
                                └─────────┬────────────────┘
                                          │
                ┌─────────────────────────┼────────────────────────────┐
                │                         │                             │
       send now │            schedule ──► │ Scheduler ──► (delays, batches)
                ▼                                                       ▼
       Per-channel Kafka topics                              re-enters Channel Router
       ─────────────────────────────────────────────────
       │ push.send │ email.send │ sms.send │ in_app.send │
       └─────┬─────┴──────┬─────┴────┬─────┴──────┬─────┘
             │            │          │            │
             ▼            ▼          ▼            ▼
       ┌──────────┐ ┌──────────┐ ┌────────┐ ┌──────────┐
       │  Push    │ │  Email   │ │  SMS   │ │  In-App  │
       │  Worker  │ │  Worker  │ │ Worker │ │  Worker  │
       └────┬─────┘ └────┬─────┘ └───┬────┘ └────┬─────┘
            │            │           │           │
            ▼            ▼           ▼           ▼
       ┌──────────┐ ┌──────────┐ ┌────────┐ ┌──────────┐
       │ APNs/FCM │ │ SES /    │ │ Twilio │ │ In-app   │
       │ pool     │ │ SendGrid │ │        │ │ Inbox DB │
       └──────────┘ └──────────┘ └────────┘ └──────────┘

   Side-channels:
       Preference Service ◄── Redis cache + Cassandra
       Template Service   ◄── Git → Redis
       Device Registry    ◄── Cassandra
       Notification Log   ◄── Cassandra (every state transition appended)

   Inbound:
       Provider webhooks (bounces, complaints, delivery receipts)
            ▼
       Feedback Processor ──► updates suppression lists
                          ──► emits events to Druid

   Tracking:
       Client SDK ──► /v1/events ──► Kafka events ──► Druid
```

### Request walk-through (transactional push, happy path)

1. Service S calls `POST /v1/notifications { user_id: u, template_id: order.delivered, idempotency_key: k, category: transactional }`.
2. **API:** authn (mTLS service identity), per-caller rate limit, schema validate. Generates `notification_id`. Returns 202 immediately.
3. The request goes (sync) into **Validation & Dedup Service** before workers see it:
   - `dedup`: `SET dedup:u:k notification_id NX EX 86400`. If `NX` fails, look up the original `notification_id`, return it on the response.
   - `prefs`: read `prefs:u` from Redis. Caller's category=transactional → bypass quiet hours, bypass marketing opt-out, but still respect channel-level opt-out (if user disabled push, fall to email).
   - `throttle`: increment counters. Transactional ignores per-user marketing cap; respects provider-level throttle.
   - `template lookup`: read `tmpl:order.delivered:en-US`. If missing locale, fall to default.
4. **Channel Router** picks: user has push enabled, has a valid APNs token → channel = push. Publishes to `push.send` Kafka topic, partitioned by `user_id`.
5. **Push Worker** consumes, renders the template (variables → final payload), calls APNs HTTP/2 connection pool. APNs returns 200. Worker writes status=`sent` to notification log + emits `event.sent` to Kafka.
6. (Async) Device receives notification. Client SDK fires `/v1/events { event_type: "delivered" }` → status=`delivered`.

End-to-end: ~500ms at p50, ~2s at p99.

### Walk-through (campaign push, 10M users, scheduled)

1. Marketing service calls `POST /v1/notifications:bulk { template, send_at: "9am local" }`.
2. **API** writes the campaign batch to a `campaigns` table; returns batch_id.
3. **Scheduler** at the `send_at` minute reads the batch and emits per-user notifications into the pipeline. **Crucially**: split into per-timezone groups so 9am-local fires at 17 different real times across the globe, not one giant 9am-UTC stampede.
4. Per-user, the same flow as above: dedup → prefs → throttle → router → channel topic → worker → provider.
5. Per-channel topics are sized to absorb the burst. Push topic has e.g. 200 partitions, allowing 200K msgs/sec aggregate.

### Why per-channel queues, not one global queue

- **Failure isolation.** APNs has an outage → push.send Kafka backs up; email/SMS/in-app keep flowing.
- **Throughput differentials.** Push handles 10× the volume of email; we don't want a slow email worker to slow push consumption.
- **Independent scaling.** Workers per channel scale based on their own provider's behavior.
- **Different consumer semantics.** Push workers are stateless and short-lived; SMS workers must manage Twilio long-code throttles via shared state.

---

## 8. Deep dives

### Deep dive 1 — Channel routing & user preferences

> **Interviewer:** "Walk me through how the system decides which channel to use."

**Candidate:**

This is the core abstraction. Caller doesn't pick — we do, given:

1. **Category** — transactional vs marketing vs social vs security. Drives whether opt-out applies.
2. **User preferences** — global channel toggles + per-category overrides.
3. **Channel availability** — does the user have a valid APNs token? Verified email? E.164 phone?
4. **Channel cost & latency suitability** — push is cheap and fast; SMS is expensive and reliable.
5. **Override hint** from caller (advisory only).

#### Decision logic

```python
def pick_channel(user, notification):
    prefs = user.preferences

    # Transactional + security categories: minimum bar is "any working channel"
    if notification.category in {"transactional", "security"}:
        candidates = [c for c in available_channels(user) if c not in prefs.suppressed]
        # Prefer cheap+fast: push > in_app > email > sms
        for c in ["push", "in_app", "email", "sms"]:
            if c in candidates:
                return c
        return None  # no channel — write to in-app inbox as fallback

    # Marketing / social: respect opt-outs
    enabled = [c for c in CHANNELS
               if prefs.channels[c]
               and prefs.category_prefs[notification.category][c]]
    if not enabled:
        return None  # user opted out — drop, log opt_out_drop event

    # Among enabled, pick by template hint or default
    return notification.channel_hint if notification.channel_hint in enabled else enabled[0]
```

#### Fallback chain

For transactional, if push fails (e.g., bad token, APNs rejected), we **don't immediately fall through to SMS** — that doubles cost and confuses the user. Instead:

- Retry push 2× with backoff.
- After 30s of no `delivered` callback (silent push failure), check policy:
  - 2FA / security: fall to SMS, log channel-fallback event.
  - Other transactional: fall to email if user has one.
- Cap fallback chain length at 2 (no infinite loops).

Fallback decisions are encoded **per template**, not per category — different templates have different urgency.

#### Preference cache architecture

Reading user prefs on every send is a 1B reads/day query — has to hit cache.

- **Hot cache:** Redis. `prefs:{user_id}` → JSON blob. TTL 5 min.
- **Cold path:** Cassandra. Read on cache miss, populate Redis.
- **Invalidation:** when user updates prefs via `PUT /v1/users/{id}/preferences`, write to Cassandra (strong consistency required for compliance), then publish `pref.changed` event to Redis pub/sub. Cache nodes invalidate.
- **Staleness budget:** ~1 minute is acceptable for marketing prefs. **Opt-out is the exception — must apply immediately.** For opt-out specifically, we additionally write a tombstone to a global Redis set with longer TTL, and the send pipeline checks both. A user who clicks "unsubscribe" should never receive another marketing message even if cache hasn't refreshed.

> **Interviewer:** "What if Redis is empty after a restart?"

**Candidate:** Cold-start scenario. We don't block — we serve from Cassandra at degraded latency (5–10ms instead of <1ms). Cassandra is sized to absorb a full cache miss for ~10 minutes. We pre-warm the cache on deploy by streaming the active-user set from Cassandra on startup.

### Deep dive 2 — Deduplication

> **Interviewer:** "Walk me through dedup. What's the failure mode and how do you avoid duplicates?"

**Candidate:**

Dedup is the single most user-visible reliability feature. Two notifications saying "your order arrived" feels broken. Two 2FA codes is worse — users use the wrong one.

#### Sources of duplicates

1. **Caller retries.** Service S submits, gets a 202, then network blips and retries. Two notifications enqueued.
2. **Multiple producers.** A monolithic service split into microservices may have two paths emitting the same business event (e.g., `order_completed` from order service AND from payments service).
3. **Reprocessing.** Kafka consumer fails after publishing to a downstream topic but before committing offset. Reprocessing re-publishes.
4. **Pipeline replay.** We replay events for backfill / disaster recovery and accidentally re-send.

#### Mechanism

**Two-layer dedup**:

**Layer 1 — Caller idempotency key (Redis):**
```
SET dedup:{user_id}:{idempotency_key} {notification_id} NX EX 86400
```
- `NX` makes it atomic: first writer wins.
- If we collide, we return the original `notification_id` to the caller. From their POV the call succeeded with the same ID — true idempotency.
- TTL 24h covers the realistic retry window. Configurable per-category (security tokens: 5 min; weekly digest: 7 days).

**Layer 2 — Send-time dedup (Cassandra unique key + Kafka log compaction):**
- Even if Layer 1 is bypassed (e.g., Redis miss), we write `idempotency_key` as a unique secondary in Cassandra. Insert-if-not-exists at the worker.
- Per-channel send topics use `user_id + idempotency_key` as the Kafka message key. Workers do an additional in-memory dedup over a 5-minute window (best-effort, catches reprocessing-within-window).

#### What gets dedup'd vs not

- **Same idempotency_key, same user, within window → dedup.** Returns original notification_id.
- **Same template, same user, no idempotency_key → NOT auto-dedup'd.** It's the caller's job to provide a key. This is intentional — silent dedup of distinct sends is worse than visible duplicates.
- **Same idempotency_key, different user → not dedup'd.** Keys are scoped to user.

#### Compliance overlay

For **transactional** (esp. 2FA): dedup must NEVER drop a message that belongs to a new request. If user requests a fresh 2FA code, the caller MUST use a new idempotency_key. If they reuse the old one (bug), the system returns the old notification_id and **does not send** — this is the correct behavior; the bug is in the caller.

#### What about marketing campaigns?

Campaigns: dedup at the **batch level**. A campaign with `campaign_id=C` sends each user at most once. Implementation: idempotency_key = `campaign:{C}:user:{u}`. Re-running the same campaign won't double-send.

> **Interviewer:** "What if Redis loses the dedup data?"

**Candidate:**
- Redis is replicated (RF=3, persistence on). A single-node loss doesn't lose the dedup index.
- A whole-cluster loss is the catastrophe scenario. Mitigation: Layer 2 (Cassandra unique key) catches it on the worker side. We accept ~5 min where dedup is "best-effort eventually consistent" while we restore cluster.
- For transactional categories, Cassandra layer is **synchronous** on the send path — adds 5ms but is the right call.

### Deep dive 3 — Quiet hours, timezones, and the campaign scheduler

> **Interviewer:** "How does the scheduler handle 'send at 9am local time' for 10M users across the world?"

**Candidate:**

Naïve approach: at 9am UTC, send everyone. Problem: 9am UTC is 1am Pacific. We just spammed California at 1am — opt-out spike, app-store complaints, possible regulatory issue.

**Correct approach: timezone-aware scheduling.**

#### Per-user timezone resolution

User's timezone is in their preferences (`America/Los_Angeles`, `Europe/London`, etc.). Default to inferred from IP at signup.

#### Scheduler architecture

- Campaign API `POST /v1/notifications:bulk` writes to `campaigns` table with `send_at_local: "09:00"` and a target user list.
- Scheduler service runs a per-minute cron.
- Each minute, scheduler scans for campaigns where `now in user.timezone == send_at_local`. For 10M users this is split:
  - **Pre-bucket users by timezone** at campaign creation. Now we have ~17 buckets (one per common UTC offset range, finer for half-hour zones like India).
  - At each minute, scheduler fires the bucket whose local time matches.
  - Within a bucket, the 10M/17 ≈ 600K users are streamed into the per-channel queue at the bucket's fire time.

#### Quiet hours enforcement

- **Per-user quiet hours** stored in prefs (default 21:00–08:00 local).
- At validation time: if a marketing notification arrives during the user's quiet hours, the **scheduler reschedules** it to the next non-quiet minute. We don't drop it — we delay.
- For transactional: we **bypass** quiet hours by default (2FA at 3am is correct). Except low-priority transactional (e.g., "your weekly summary") — config flag per template.

#### Spike absorption

A "9am local" campaign across all timezones spreads naturally over 24 hours, not 1 second. Within a single timezone bucket, 600K users still hit at the same minute. We further smooth:
- **Send-rate jitter:** spread the bucket's deliveries over a 10-minute window. Acceptable because users don't know when 9am "exactly" was.
- **Provider rate limits force smoothing anyway.** APNs at 10K/sec/connection means 600K takes 60 seconds with one connection, ~6s with 10 connections.

> **Interviewer:** "What if a user changes timezone right before send time?"

**Candidate:** We snapshot timezone at scheduling time. A user who flies to Tokyo and updates their phone won't get the 9am-Pacific notification re-routed — they'll get it at 9am Pacific (which is 1am Tokyo). This is wrong, but the alternative (re-resolve on send) is hard to do reliably and creates race conditions. We accept it; users rarely change timezones in the campaign window.

### Deep dive 4 — APNs / FCM connection management

> **Interviewer:** "Push is your highest-volume channel. Walk me through how the push worker scales."

**Candidate:**

APNs is the constraint. It has subtle requirements:

#### APNs HTTP/2 connection pool

- APNs is HTTP/2; one TCP connection multiplexes many streams.
- Throughput: ~10K notifications/sec/connection. Apple recommends maintaining "many" long-lived connections for high throughput.
- Connections must use mTLS with a provider certificate or token-based auth (JWT signed with team key). Token auth is preferred (one cert, all topics).
- Connections are LONG-LIVED — opening/closing thrashes Apple's edge.

**Push worker design:**
- Worker process maintains a pool of N connections (start N=10, scale up based on backlog).
- Round-robin sends across connections.
- **Connection-level retries:** if a connection's HTTP/2 stream fails, retry on another connection in the pool. If 5 connections fail in a row → mark APNs as degraded, alert.
- **Per-connection credit tracking:** APNs uses HTTP/2 flow control; a slow connection backs up. Skip it if it doesn't drain in 100ms.

**Sharding:**
- 200 push workers globally, each with a 10-connection pool = 2000 connections to APNs.
- That's 20M notifications/sec capacity — way more than we need.
- Real bottleneck: Apple may rate-limit per-team-id at very high volumes; we negotiate a higher quota in advance.

#### FCM differences

- FCM is HTTP/REST per message OR HTTP/2 batch. We use the v1 HTTP/2 endpoint.
- Multi-project sharding: distribute device tokens across N FCM projects to escape per-project rate limits.
- FCM has a "topic" feature (broadcast to subscribers) — we don't use it; we want per-user observability.

#### Bad token handling

- APNs returns `Unregistered` (410) when a device uninstalled the app.
- Worker marks the device token as `bad_token=true` in `device_registry`.
- Future sends skip that token. If user has no remaining valid tokens, channel router falls to next channel.
- **Cleanup:** background job purges `bad_token=true` rows after 90 days.

#### Silent push concerns

- iOS / Android limit silent (data-only) push frequency. Spam is detected at OS level; our app may be throttled or banned.
- We separate "user-visible" (alert/badge/sound) from "silent" (data refresh) at the API. Silent has its own rate limit (per-user, per-day).
- Templates explicitly declare type — caller can't sneak silent-pushes through a user-visible template.

### Deep dive 5 — Click & open tracking

> **Interviewer:** "How do you know if the notification got delivered, opened, clicked?"

**Candidate:**

Three event types, three mechanisms:

| Event | Mechanism | Reliability |
|---|---|---|
| **Sent** | Provider 200 OK | High — provider acked |
| **Delivered** | Provider callback (if subscribed) OR client SDK ping | Medium — depends on device online state |
| **Opened** | Client SDK fires when app opens with notification context, OR pixel fetch (email) | Lossy — privacy features (Apple Mail Privacy Protection) inflate open rates |
| **Clicked** | Redirect link via tracking domain | High — we control the redirect |

#### Pipeline

1. Client SDK posts to `POST /v1/events { notification_id, event_type, timestamp }`.
2. Events ingest service validates (signed event from SDK, prevents spoofing) and publishes to `events` Kafka topic.
3. Two consumers:
   - **Status updater** consumes and updates `notifications.status` in Cassandra.
   - **Druid ingest** consumes and lands events in Druid for analytics.

#### Click tracking via redirect

- Original URL: `https://app.example.com/orders/123`
- Rendered in notification: `https://nt.example.com/c/{notification_id}/{signed_url_hash}` → 302 redirect after logging.
- Tracking domain `nt.example.com` is intentionally separate (so users can block it; spam filters don't penalize main domain).

#### Open tracking via pixel (email)

- Embed `<img src="https://nt.example.com/o/{notification_id}.gif" />` in HTML email.
- Fetched on email open; logged.
- **Caveat:** Apple Mail Privacy Protection pre-fetches images, inflating opens. We disclose this metric is "directional" not "true."

#### Privacy

- Click/open events DO NOT carry user PII to Druid — only `notification_id` (which is internally joinable to user_id, but the analytics pipeline operates on aggregated metrics).
- GDPR delete: when user requests deletion, we tombstone all their `notification_id`s; aggregated metrics are not personally identifiable so survive.

### Deep dive 6 — Throttling & rate limits

> **Interviewer:** "Tell me about throttling — both per-user and per-provider."

**Candidate:**

Two distinct problems:

#### Per-user (don't spam)

- **Daily cap:** 50 notifications/user/day (transactional NOT counted; marketing IS).
- **Hourly cap:** 5 marketing/hour.
- **Per-template cap:** "weekly digest" template can fire at most 1×/week per user.

Implementation: Redis sliding window counters. Increment at validation time. If exceeded → drop with reason `throttled`, emit event for analytics.

**Why this matters:** opt-out rates are a leading indicator of trouble. Users who feel spammed unsubscribe. Caps are conservative defaults; PMs can raise them per-category but must justify.

#### Per-provider (don't get rate-limited)

- **APNs/FCM:** managed at connection-pool level (see Deep Dive 4).
- **SES:** account-level send rate. We monitor `SendQuota` API; if approaching 80%, send pipeline backs off.
- **Twilio:** per-number rate limits. We have N short-codes / 10DLC numbers; distribute load across them with consistent hashing on user_id (so the same user uses the same sender — better deliverability and 2-way SMS works).

#### Backpressure

If providers throttle:
- Worker emits 429 → consumer pauses Kafka offset commit → backlog grows.
- After 30s of sustained backpressure: alert.
- After 5 min: dynamically lower throughput targets — the producer side throttles as well, so we don't burn through retries.
- For transactional, we have a separate higher-priority Kafka topic (`push.send.priority`) consumed by a reserved worker pool. Marketing throttles before transactional does.

### Deep dive 7 — Failure modes & cross-channel fallback

> **Interviewer:** "What happens if APNs is down for 30 minutes?"

**Candidate:**

Detect → mitigate → recover, in that order.

**Detection (within 30s):**
- APNs error rate alarm at >5% over 1 minute.
- Connection-pool health check.
- Synthetic monitoring: every minute, send a test notification to a known device, expect callback.

**Immediate mitigation:**
- Workers stop committing Kafka offsets — backlog accumulates in Kafka (retains 24h, plenty of headroom).
- Status of in-flight notifications: marked `pending_provider`. Not lost.
- For **transactional/security category specifically**: trigger fallback to SMS or email after 30s no-delivery.
- For **marketing category**: hold in queue. Better to send late than send wrong-channel.

**Recovery:**
- When APNs recovers (error rate drops), workers resume committing.
- Kafka backlog drains in seconds-to-minutes (depending on duration of outage).

**Data loss bound:** zero notifications lost (Kafka retains; status tracked end-to-end). Some notifications late (up to outage duration).

#### What about a regional APNs outage?

- APNs is multi-region operated by Apple. Their issues are usually <30 min.
- For sustained outages, we'd halt new marketing pushes (announce to PM) and route transactional to SMS as the emergency path.

#### Other failures

| Failure | Detection | Blast radius | Mitigation | Recovery |
|---|---|---|---|---|
| Redis cache cluster lost | Cache miss rate spike | Send latency p99 ↑ from 50ms to 5s | Cassandra absorbs; degraded but functional | Re-warm from Cassandra; restore Redis |
| Cassandra node down | Per-shard latency alert | 1/N of users have higher write latency | RF=3, write at QUORUM allows 1 down | Replace, stream-rebuild |
| Kafka cluster lost (catastrophic) | Producer error rate | Send pipeline halts | Producers buffer in local WAL (10-min capacity); switch to standby cluster | Multi-AZ Kafka prevents most cases |
| Template render error | Per-template error rate | Sends with bad template fail (others unaffected) | Per-template circuit breaker — disable that template after N failures | On-call investigates; deploy fix |
| Bad email/phone (bounce/complaint) | Provider webhook | That user can't receive on that channel | Auto-suppression list update; reroute to alternate channel | Background re-validation if user updates address |
| Provider webhook lost | Reconciliation job | Status stuck at `sent` | Periodic reconciliation queries provider for status | Catch up status |
| Dedup miss (Redis lost a key) | Duplicate sent | One user gets 2 of the same notification | Layer 2 (Cassandra) catches most; some leak through | Accept rare leak for marketing; for transactional, Layer 2 is synchronous |
| Schema migration | Pre-deploy | Service crash | Forward+backward compatible; phased deploy | Roll back if needed |
| Massive opt-out spike (canary) | Hourly opt-out rate | Indicates spam — user trust at risk | Auto-pause campaigns; page on-call for content review | PM/legal review of recent sends |
| Silent push spam (app-store penalty) | Apple/Google notification | App may be downgraded in store | Per-category silent push caps; require justification for silent | Audit caller patterns |

### Deep dive 8 — Marketing vs transactional separation

> **Interviewer:** "You mentioned different SLOs for transactional vs marketing. How does that manifest in the architecture?"

**Candidate:**

**Separate queues, separate workers, separate connection pools.**

```
                                  ┌──────────────────────┐
                                  │   Channel Router     │
                                  └──────────┬───────────┘
                                             │
                  ┌──────────────────────────┼───────────────────────┐
                  │                          │                        │
         transactional                  marketing                 social/etc
                  │                          │                        │
                  ▼                          ▼                        ▼
        ┌──────────────────┐    ┌──────────────────┐    ┌──────────────────┐
        │ push.send.txn    │    │ push.send.mkt    │    │ push.send.social │
        │ (high-pri Kafka) │    │ (normal Kafka)   │    │                  │
        └────────┬─────────┘    └─────────┬────────┘    └─────────┬────────┘
                 │                        │                       │
                 ▼                        ▼                       ▼
        ┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
        │ TXN workers     │    │ MKT workers     │    │ SOC workers     │
        │ (reserved 30%   │    │ (autoscale on   │    │                 │
        │  of capacity)   │    │  backlog)       │    │                 │
        └─────────────────┘    └─────────────────┘    └─────────────────┘
```

Why:
- **No starvation.** A 100M-marketing-blast cannot delay a 2FA code.
- **Different SLOs auditable.** TXN p99 < 5s; MKT p99 < 5min. Separate dashboards.
- **Different scaling policies.** TXN target utilization 30% (always headroom); MKT target 70% (cost-efficient).
- **Different deploy cadences.** TXN deploys are conservative (canary 5% for 24h); MKT deploys can be more aggressive.

**Within transactional**, security (2FA, password reset) gets the highest priority — its own sub-queue with even tighter SLO.

> **Interviewer:** "Doesn't this duplicate infrastructure?"

**Candidate:** Yes. The cost is proportional to having reserved capacity for transactional. We don't double the workers — we partition them. Transactional has 30% reserved + can borrow from marketing pool when idle. Real cost increase: ~10–15%, justified by the SLO isolation.

---

## 9. Tradeoffs & alternatives

### Decision: Per-channel queues

| Option | Pros | Cons | When to pick |
|---|---|---|---|
| **One global queue with channel field** | Simpler | Failure of one channel backs up others; mixed SLO impossible | Toy systems |
| **✅ Per-channel topics** | Isolation, independent scaling, channel-specific consumer logic | Topic management overhead | Production at our scale |
| **Per-channel + per-priority** | Even better isolation | More topics to manage | Once volume crosses 100K/sec |

We picked per-channel + per-priority (txn/mkt/social) within each — 12 topics total. Manageable.

### Decision: Cassandra for notification log

| Option | Pros | Cons | When to pick |
|---|---|---|---|
| **PostgreSQL** | ACID, joins | Doesn't scale to 1B inserts/day | Small-volume systems |
| **MongoDB** | Schema flexibility | Worse write throughput at scale | Schema-heavy use cases |
| **DynamoDB** | Managed, scales | Vendor lock-in, expensive at this volume | AWS-native, smaller ops team |
| **✅ Cassandra** | Wide-row write-optimal, partitions match access pattern, tunable consistency | Operational complexity | Hyperscale write-heavy |

We have Cassandra muscle. The wide-row pattern (`partition by user_id, clustering by created_at desc`) maps perfectly to "show me this user's notifications" and "TTL old ones."

### Decision: Redis for prefs cache, dedup, throttle

| Option | Pros | Cons | When to pick |
|---|---|---|---|
| **Memcached** | Simple, fast | No persistence, no atomic SET NX EX | Pure-cache use cases |
| **In-process LRU cache** | No network hop | Inconsistent across worker fleet | Single-host services |
| **✅ Redis** | Atomic ops (SET NX), data structures (sliding windows), persistence option | RAM cost; cluster ops | Multi-host systems needing atomicity |

`SET dedup:k v NX EX 86400` is the killer — atomic check-and-set with TTL in one round trip.

### Decision: Kafka for pipeline

Same reasoning as in news feed. Persistent log, replayable, high throughput, partition ordering. The alternative (SQS) doesn't replay easily, and we need replay for the post-incident "re-deliver lost notifications" path.

### Decision: SaaS providers vs in-house for non-push

| Channel | Choice | Why |
|---|---|---|
| Push (APNs/FCM) | In-house workers, direct provider API | Volume justifies the connection management complexity; SaaS push (OneSignal) marks up too much at our scale |
| Email | SES (with SendGrid as backup) | Building email infra (DKIM, SPF, IP warming, deliverability) is multi-year work. SaaS wins for email at any scale. |
| SMS | Twilio | Same reasoning. Plus we'd need carrier relationships globally. |
| In-app | In-house (DB-backed inbox) | It's just a row in a table. No reason to outsource. |

The pattern: **in-house when volume is high enough that SaaS markup costs more than the engineering complexity. SaaS otherwise.** Push is in-house; everything else is SaaS-fronted with our adapters.

### Decision: Eventual consistency for prefs cache (1-min staleness)

Acceptable for marketing prefs. NOT acceptable for opt-out — when user clicks "unsubscribe," compliance requires we honor it immediately. So opt-out has its own faster invalidation path (cluster-wide pub/sub fanout), and we consult both the cache AND a separate "opt-out tombstone set" that has shorter staleness.

### Decision: Dedup with Redis Layer 1 + Cassandra Layer 2

Layer 1 alone is fast but loses data on Redis failure. Layer 2 alone is slow on the hot path. Combined: fast common case, durable rare case. Cost: ~5ms extra latency on writes when Layer 2 is synchronous (only for transactional). For marketing we accept Layer 1 only.

---

## 10. Defending architectural choices — interviewer attacks

> **Interviewer:** "Why not let callers pick the channel? It's their notification — they know best."

**Candidate:** No. Three reasons:
1. **User preferences** — the user owns the channel choice, not the caller. A caller can't know the user has SMS disabled.
2. **Cost and policy enforcement** — if every caller picked SMS, we'd burn $250K/day. The platform owns cost discipline.
3. **Channel availability** — the caller doesn't know if the user has a valid APNs token. The platform does.
The right abstraction is: caller declares **intent and category**; platform decides **channel**. The caller can hint, and we'll respect it when it doesn't violate policy.

> **Interviewer:** "Why not have one queue with priority, instead of per-channel and per-priority queues?"

**Candidate:** Priority queues at scale have a starvation problem — a flood of high-priority work freezes everything else. Separate queues with reserved capacity is simpler to reason about and gives you per-queue SLOs. We can independently set retention, partition count, and consumer count per queue. Mixing them in one queue couples their failures.

> **Interviewer:** "Why Cassandra over DynamoDB? You said we already operate Cassandra — what if we didn't?"

**Candidate:** If we were greenfield, DynamoDB is a strong default — managed, no ops, scales. The reasons we'd still pick Cassandra:
- Multi-cloud/multi-region with control over consistency tunables — Dynamo's strong/eventual is binary.
- At 1B writes/day × 90-day retention, Dynamo's per-WCU pricing is expensive. Cassandra is cheaper at hyperscale.
- We want secondary indexes on `idempotency_key`. Dynamo's GSI works but adds RCU/WCU costs.
At a smaller team / smaller scale I'd flip to Dynamo and pocket the ops savings.

> **Interviewer:** "Why a 24h dedup window, not forever?"

**Candidate:** Forever is wrong. Idempotency keys are operational artifacts (caller-generated, often `request_id`-based). Two months later the same key shouldn't dedup against an unrelated send. 24h covers realistic retry windows. Configurable per category (security keys: 5 min — fresh code on each request; weekly digest: 7 days). The window is a knob, not a constant.

> **Interviewer:** "How do you handle a user who has push enabled but a stale (uninstalled) APNs token?"

**Candidate:** APNs returns `Unregistered` (410). Push worker:
1. Marks the token `bad_token=true` in `device_registry`.
2. Re-queries the user's other valid tokens — if any, retry on those.
3. If no valid push tokens, the channel router re-evaluates: fall to next channel per policy.
4. The notification status becomes `delivered` (via fallback channel) or `failed` (no channel worked).
On the user side: when they reinstall, the SDK registers a new token and we resume push.

> **Interviewer:** "What's the worst customer-facing bug this system can have?"

**Candidate:** Three contenders:
1. **Sending after opt-out** — compliance violation. CASL fines. Lawsuits possible.
2. **Failing to deliver 2FA** — locks users out of their accounts. Direct revenue impact.
3. **Spamming during a campaign bug** — opt-out spike, app-store penalty.
We design for (1) with tombstone-fast-invalidation; for (2) with TXN priority queue and SMS fallback; for (3) with per-template kill switches and gradual rollout.

> **Interviewer:** "Why are templates Git-backed? Marketers want to edit them in a UI."

**Candidate:** Git-backed because:
- **Code review** — bad templates have crashed sends before. PR review catches injection, locale errors, missing variable handling.
- **Deploy gates** — staged rollout (shadow → 1% → 100%).
- **Rollback** — `git revert` is faster and safer than ad-hoc UI rollback.
Marketers DO get a UI — but the UI commits Git via a PR-style flow (with auto-review for low-risk template changes). The Git source of truth is non-negotiable; the UI can be friendlier on top.

> **Interviewer:** "What happens if Twilio's SMS endpoint is degraded — high latency but not down?"

**Candidate:**
- Worker observes p99 latency degradation → exposed as a metric.
- We have a circuit breaker: if Twilio p99 > 30s for 1 minute, open the breaker — pause SMS sends except critical (2FA), redirect to backup provider (Sinch / MessageBird).
- For pure marketing SMS, we'd hold rather than fail over (different provider may have different sender numbers, may break user expectations).
- Critical: don't burn through retry budget hammering a slow provider. Circuit breaker before retries, not after.

---

## 11. Failure modes & mitigations

| Failure | Detection | Blast radius | Mitigation | Recovery |
|---|---|---|---|---|
| API service down | 5xx alert | Caller-visible enqueue failures | LB removes unhealthy; multi-region failover | Auto-rollback if recent deploy |
| Validation/dedup service down | Per-service health | Send pipeline halts at validation | Multi-AZ deployment; failover within seconds | n/a |
| Redis prefs cache cluster lost | Cache hit rate drop, latency spike | Send latency p99 ↑ from 50ms → 5s | Cassandra direct reads; pipeline functional but slower | Re-warm cache from Cassandra |
| Redis dedup cluster lost | Dedup miss rate ↑ (rare since synchronous) | Some duplicates leak through | Cassandra Layer 2 catches most | Restore Redis from snapshot |
| Cassandra node down | Per-shard error rate | 1/N of users have higher latency | RF=3, QUORUM tolerates 1 down | Replace, stream-rebuild |
| Cassandra cluster split-brain | Replication lag, write errors | Severe — investigate immediately | Multi-region replication for DR | Manual intervention |
| Kafka topic backed up | Consumer lag metric | Channel-specific delay; transactional unaffected if separated | Auto-scale consumers; lower throughput target dynamically | Drain backlog |
| Kafka cluster lost | Producer errors | Send pipeline halts | Producers buffer locally (10 min capacity); switch to standby cluster | Multi-AZ prevents most cases |
| APNs / FCM degraded | Provider error rate spike | Push delayed | Connection pool retries; fall to SMS for transactional | Wait it out (usually <30 min) |
| SES throttled / blocked | SendQuota API | Email sends paused | Pre-warm IPs; backup provider (SendGrid) | IP reputation recovery |
| Twilio degraded | Provider latency / error | SMS delayed or failed | Circuit breaker; fail to backup provider for critical SMS | Wait out / fail over |
| Template render error | Per-template error rate alert | That template's sends fail | Per-template circuit breaker | Deploy fix; replay from Kafka |
| Bad email address (bounce) | SES bounce webhook | Future sends to that user fail on email | Auto-add to suppression list | User updates email |
| Spam complaint (email) | SES complaint webhook | If sustained, IP reputation damaged | Auto-add user to suppression; throttle sender pattern | IP warming if needed |
| Opt-out spike (canary signal) | Hourly opt-out rate alert | Indicates we're spamming; user trust at risk | Auto-pause campaigns; PM/legal review | Identify offending template; halt |
| Silent push spam (app-store penalty) | App-store notification | App downgraded; ranking impact | Per-category silent push caps; manual review for new silent templates | Audit caller; remediate |
| Schema migration regression | Pre-deploy testing | Service crash on deploy | Forward+backward compatible; phased deploy | Roll back |
| Compliance breach (sent after opt-out) | User report / audit | Legal exposure | Opt-out tombstone with fast invalidation; audit log | Immediate remediation, regulator notification |
| Cross-region data leak (e.g., EU notification routed via US) | Audit log | Compliance breach | Region-tagged routing; per-region cells | Manual incident, regulatory notification |
| DDoS on tracking endpoints (open/click) | Edge metrics | Tracking degraded; sends unaffected | WAF rate limit; CAPTCHA on click endpoint | n/a |
| Rogue caller spamming users | Per-caller rate limit alert | One caller can flood pipeline | Per-caller quotas at API gateway; auto-throttle | Identify and remediate |

---

## 12. Productionization

### Pre-launch
- [ ] **Dry-run mode:** every notification can be submitted with `dry_run: true` — full pipeline runs, validations applied, but final provider call is skipped. Returns simulated status. Critical for testing campaigns before they go live.
- [ ] **Test send:** "send to specific user-IDs only" — campaigns can specify a `test_audience` list. Used internally to preview a campaign before broad release.
- [ ] **Load test to 5× peak.** 750K notifications/sec sustained for 1 hour. Verify backlog stays empty, latency holds.
- [ ] **Chaos test:** kill APNs connections mid-campaign. Verify Kafka backlog absorbs and drains. Kill Cassandra nodes. Kill Redis nodes.
- [ ] **End-to-end test in production:** every minute, send a synthetic notification to a known-test user; verify open callback within 60s. SLO this.

### Rollout
- [ ] **Per-template gradual rollout:** new template starts in shadow mode (rendered but not sent), then 1%, 5%, 25%, 100% with bake at each.
- [ ] **Per-region rollout:** smallest market first (e.g., a non-EU non-US region) — verify per-region cell. Then EU. Then US.
- [ ] **Auto-rollback:** if delivery rate drops >2% from baseline, or opt-out rate doubles, auto-pause campaign and alert.

### Capacity
- [ ] Headroom: 50% above peak. Marketing campaigns are bursty; we MUST absorb 10× spike without latency degradation.
- [ ] Auto-scale workers on Kafka backlog depth (target: <30s of work).
- [ ] Reserved capacity for TXN: 30% of total worker fleet, never preempted.
- [ ] Cassandra: provision for 3× current write volume.
- [ ] Kafka: 5× current partition count, repartitioning is expensive.

### Migration (if replacing existing system)
- [ ] **Dual-write phase (1 month):** every send goes through both old and new system. Compare outputs. Investigate divergence.
- [ ] **Read traffic cutover:** new system serves status queries gradually.
- [ ] **Send traffic cutover:** per-category, per-channel. Transactional last (lowest risk-tolerance).
- [ ] **Decommission old:** keep for 2 weeks of rollback insurance.

### Cost
- $/notification target: <$0.001 blended. Cost dominator is SMS at $0.005.
- **Levers:**
  - Route via cheapest channel meeting SLA (push > in-app > email > SMS).
  - Batch APNs/FCM connection use (already done at protocol level).
  - Compress notification payloads.
  - Suppression lists shrink wasted sends.
  - For marketing: A/B test channel — does SMS lift vs push justify 1000× cost?

### Compliance & security
- **Authn:** mTLS service-to-service. Caller identity audited per request.
- **Authz:** per-category, per-template. Marketing service can't send transactional. Security service can't send marketing.
- **PII:** email/phone encrypted at rest. Access logged. Auto-deleted on user delete.
- **GDPR right-to-erasure:** propagates to notification log (TTL accelerates), prefs (deleted), suppression list (anonymized).
- **CASL/TCPA:** opt-out is honored within seconds; audit log retained 5 years.
- **Audit:** every send logged with caller, user, category, channel, timestamp. Queryable for compliance review.

### Per-template kill switches
- Each template has an `enabled` boolean in config (Redis-backed, sub-second propagation).
- On-call can flip a template to disabled instantly. Sends in-flight complete; new sends drop.
- Critical for incident response — "marketing campaign sending to wrong audience" is a 1-button fix.

### Bounce / complaint handling
- SES SNS subscriptions for bounce + complaint events.
- Twilio status callbacks.
- Async pipeline updates user's `suppression_list`. After 1 hard bounce, no more email to that address. After complaint, auto-unsubscribe from category.
- Soft bounces: tolerated up to 5×; then suppress.

---

## 13. Monitoring & metrics

### Business KPIs
- **Notifications sent/day** — total volume.
- **Delivery rate** per channel (sent → delivered fraction).
- **Open rate, click rate** per template (engagement).
- **Opt-out rate** per category, per template (CANARY signal — this is the leading indicator of trouble).
- **Conversion rate** (clicks → desired action).

### Service-level (SLIs)
- **API ingest** — RED metrics, p50/p99 latency, error rate. Per caller.
- **End-to-end send latency** — API ingress to provider acked. p50, p99. Per channel. Per category. SLO: TXN p99 < 5s; MKT p99 < 5min.
- **Send success rate** — sent / submitted. SLO: TXN ≥99.9%; MKT ≥99%.
- **Dedup hit rate** — what % of submissions are dedup'd. Should be small but non-zero (~1–5%); a sudden jump means a caller is broken.

### Component metrics

#### API & Validation
- Request rate, error rate, p99 latency
- Schema validation rejection rate (catches caller bugs)
- Per-caller rate-limit hit rate

#### Channel Router
- Per-channel routing distribution (% to push, email, etc.)
- Fallback rate (notifications that fell to alternate channel)
- "No valid channel" drop rate (compliance signal — should be near zero)

#### Push Worker
- APNs/FCM error rate per error type (BadDeviceToken, Unregistered, throttled)
- Connection pool size, utilization
- Per-connection p99 latency
- Bad token cleanup rate

#### Email Worker
- SES throttle rate (SendQuota approach)
- Bounce / complaint rate (CANARY — high → IP reputation damage imminent)
- Render failure rate per template

#### SMS Worker
- Twilio error rate (per error type)
- Per-sender-number throttle hit rate
- Long-code vs short-code mix

#### Provider integrations
- Provider p99 latency
- Provider error rate per error class
- Webhook receipt rate (for delivery callbacks)

#### Preference Service
- Cache hit rate (target ≥99%)
- Cache invalidation lag (pub/sub propagation time)
- Cassandra read latency on miss

#### Templates
- Per-template render success rate
- Per-template send volume (anomaly detection)
- Template version distribution

#### Dedup
- Dedup hit rate
- Layer 1 (Redis) miss rate that hit Layer 2
- TTL distribution

#### Kafka
- Per-topic publish rate, error rate
- Per-consumer-group lag (alert >30s for TXN, >5min for MKT)
- Partition skew

#### Cassandra
- Read/write latency per node
- Replication lag
- Disk utilization
- Hint queue
- Hot partition detection

#### Redis
- Hit rate per cache (prefs, dedup, throttle)
- Memory utilization
- Eviction rate

### Alert routing
- **Page (24/7):** TXN SLO breach, opt-out rate spike, complaint rate above 0.1%, regional cell down, provider sustained outage, send-pipeline-halt.
- **Ticket (next business day):** cache hit rate degraded, single Cassandra node, single template circuit breaker open, schema mismatches.
- **Dashboard-only:** cost trends, channel mix shifts, capacity headroom.

### Dashboards
1. **Send health** — RED metrics per channel, per category, end-to-end latency.
2. **Pipeline** — Kafka lag per topic, worker fleet utilization.
3. **Providers** — APNs, FCM, SES, Twilio dashboards (separate, owned by integration team).
4. **Preferences** — cache hit, opt-out events, suppression list size.
5. **Business** — sends/day, opens/clicks, opt-out rate (this is the one PMs watch).
6. **Compliance** — sends-after-opt-out (must be 0), region-leakage (must be 0), audit log integrity.

---

## 14. Security & privacy

- **Authn (caller-side):** mTLS service-to-service. JWT for human-driven flows. API keys are short-lived and rotated.
- **Authz (per category):** transactional callers cannot send marketing; marketing callers cannot send security messages. Enforced at API.
- **PII handling:** email/phone encrypted at rest with envelope encryption. Access logged. Decrypted only inside send workers, never returned via API.
- **Audit logging:** every send logged with `caller_service`, `user_id`, `template`, `channel`, `category`, `timestamp`. Retained 5 years.
- **Tracking domain isolation:** click/open URLs use a separate domain to limit primary-domain tracking and to enable user-side blocking.
- **Webhook signatures:** all inbound webhooks (SES, Twilio, FCM) signature-verified.
- **Rate limit at edge:** per-caller, per-user. Catches rogue callers.
- **Compliance:**
  - GDPR: consent recorded, opt-out within seconds, right-to-erasure honored within 30 days.
  - CASL (Canada): explicit consent required for marketing; unsubscribe link mandatory in every commercial message.
  - TCPA (US): time-of-day restrictions for SMS (8am–9pm local), STOP-keyword handling.
  - CAN-SPAM: physical address in email footer, unsubscribe link, accurate from/subject.
- **Suppression lists global:** opt-out propagates across categories within seconds.

---

## 15. Cost analysis

### Per-channel economics
| Channel | Volume/day | Unit cost | Daily cost | Annual |
|---|---|---|---|---|
| Push | 700M | $0 | ~$200 | $73K |
| Email | 150M | $0.0001 | $15K | $5.5M |
| SMS | 50M | $0.005 | $250K | **$91M** |
| In-app | 100M | $0 | $50 | $18K |

**SMS dominates by 10× over email and 1000× over push.**

### Cost levers
1. **Channel-routing optimization** — every notification we route via push instead of SMS saves $0.005. At scale, shifting 5% of sends from SMS to push is $25/day → millions/year.
2. **Suppression lists** — every bounce/complaint we honor saves a future send. Critical at email scale.
3. **Tiered storage for notification log** — hot tier (7 days) on Cassandra; warm tier (30 days) on cheaper storage; cold tier (audit) on S3 / object storage.
4. **Compression** — payloads are small (1KB), but at 1B/day every byte is 1GB. Compression in Kafka and Cassandra cuts 30%.
5. **Provider negotiation** — Twilio rate is volume-discounted; renegotiate annually.
6. **Marketing send caps** — caps per-user prevent waste and improve user experience simultaneously.

### Cost per active user
- ~$265K/day / 200M DAU = $0.0013/DAU/day → ~$0.40/DAU/year
- Of which SMS is ~$0.34/DAU/year (the cost question is "do we need SMS at all for non-2FA?")

---

## 16. Open questions / what I'd validate with PM

1. **What's our real SMS dependency?** If 99% of SMS is 2FA, we may push remaining marketing SMS to email and save 70% of cost.
2. **What's the per-region compliance posture?** EU and Brazil data residency drives multi-region cell architecture (next section). India's DPDP affects what we can send and store.
3. **Is real-time campaign editing required?** If marketing wants to edit templates and re-send mid-campaign, we need a hot-reload path (currently: Git deploy → Redis push, ~5 min).
4. **What's the abuse model?** Are there caller services we DON'T trust (e.g., third-party integrations)? Affects authz model.
5. **How much engagement data do we expose?** Click/open events are a privacy boundary. Aggregated per-template is safe; per-user is sensitive.
6. **Is there a 2-way SMS requirement?** If users reply STOP to opt out of campaigns, we need inbound SMS routing (Twilio supports it).
7. **Cross-region failover policy?** If EU cell is down, do we route EU users via US for transactional? Probably not (residency), but we should have an explicit answer.

---

## 17. Staff-level scorecard for this problem

| Signal | Did the candidate... |
|---|---|
| ✅ Identified channel routing as the core abstraction | Caller declares intent; platform picks channel, not the other way around |
| ✅ Quantified the cost asymmetry across channels | "SMS is 1000× push" stated explicitly; cost as a design lever |
| ✅ Designed dedup with two layers | Redis fast path + Cassandra durable layer |
| ✅ Treated quiet hours/timezone as first-class | Per-user timezone, per-bucket scheduling |
| ✅ Separated transactional from marketing | Different queues, workers, SLOs — not "one priority queue" |
| ✅ Connection management for APNs/FCM made explicit | Long-lived HTTP/2, pooling, 410-handling |
| ✅ Compliance discussed unprompted | GDPR, CASL, TCPA, opt-out semantics |
| ✅ Failure modes with cross-channel fallback | Push down → SMS for security; not "retry forever" |
| ✅ Bounce/complaint suppression as a closed loop | Webhook → suppression list → routing decision |
| ✅ Productionization volunteered | Dry-run, test-send, kill switches, gradual rollout |

### What separates Staff from Senior on *this* problem

A Senior candidate will:
- Draw API → queue → workers per channel
- Pick reasonable databases
- Mention preferences and dedup

A Staff candidate will additionally:
- Identify **cost asymmetry across channels** as a load-bearing fact
- Build the **two-layer dedup** rather than just Redis
- **Separate transactional and marketing pipelines** with reserved capacity
- Treat quiet hours as **scheduling problem with timezone bucketing**, not a `if hour > 21` check
- Volunteer **per-template kill switches and gradual rollout**
- Discuss **opt-out spike as a canary signal** for spam detection
- Treat **templates as code** (Git-backed, reviewed, deployed) not config
- Mention **regional cells for compliance** as the natural multi-region story
- Quantify **opt-out rate as the key SLO that PMs care about**

---

## 18. Wrap-up speech (template)

> "To wrap up: the dominant constraints are (1) channel cost asymmetry (1000× between push and SMS), (2) compliance + opt-out semantics across GDPR/CASL/TCPA, and (3) the bursty 10× campaign-spike profile against a 1B/day baseline. The architecture answers all three: per-channel queues with reserved transactional capacity, two-layer dedup (Redis + Cassandra), preference cache with fast opt-out invalidation, timezone-bucketed scheduling for quiet hours, and Git-backed templates with per-template kill switches.
>
> The two pieces I'd worry about in production: (1) preference cache staleness causing post-opt-out sends — mitigated with the tombstone fast-path but I'd watch this metric closely, and (2) a buggy campaign sending to the wrong audience — mitigated with dry-run mode, test-send, and gradual rollout, but the blast radius is large. I'd SLO-alert on opt-out rate as the canary.
>
> To productionize: dry-run mode for callers, per-template gradual rollout (shadow → 1% → 100%), feature flags per category and per channel, dark-launch new providers in parallel before cutover, multi-region cells for compliance, and per-template kill switches.
>
> If we had another 30 minutes, I'd deep-dive (a) the cellular architecture for compliance — how cross-region notifications route and how the opt-out tombstones replicate globally, or (b) the analytics pipeline — Druid schema design and how we close the loop from delivery → engagement → ranking model retraining."

---

## 19. Market-based isolation (cellular architecture) — bonus / "if time permits"

> Bring this up in the last 5 minutes. For notifications it's not optional polish — for any global app, it's a compliance hard requirement.

### Why notifications need cells more than most systems

- **GDPR/DPDP/PIPL/LGPD all touch notifications directly.** A notification IS personal communication; sending an EU user's notification through a US datacenter without contractual safeguards is a violation.
- **SMS providers have regional regulations.** Twilio's compliance posture differs by country. Indian SMS regulations (DLT) require sender registration; US 10DLC has different rules; UK and EU have country-specific quirks.
- **Push providers are global** but data residency for the notification *content* + the user's prefs is per-region.
- **Marketing campaigns are inherently regional** (timezone-driven, language-driven, sometimes regulation-driven — e.g., a campaign content allowed in US may violate India's DPDP).

### Architecture

```
        ┌─────────────────── Global Edge ───────────────────┐
        │ Anycast IP + GeoDNS routes caller to nearest cell │
        └────────┬───────────────┬────────────────┬─────────┘
                 │               │                │
       ┌─────────▼────────┐ ┌────▼─────┐ ┌──────▼──────┐
       │   US Cell        │ │ EU Cell  │ │ IN Cell     │
       │ ┌──────────────┐ │ │┌────────┐│ │ ┌──────────┐│
       │ │ Notif API    │ │ ││Notif   ││ │ │Notif API ││
       │ │ Validation   │ │ ││Valid.  ││ │ │Validation││
       │ │ Channel Rtr  │ │ ││Router  ││ │ │Router    ││
       │ │ Workers      │ │ ││Workers ││ │ │Workers   ││
       │ │ Cassandra    │ │ ││Cass.   ││ │ │Cassandra ││
       │ │ Redis        │ │ ││Redis   ││ │ │Redis     ││
       │ │ Kafka        │ │ ││Kafka   ││ │ │Kafka     ││
       │ │ SES (US)     │ │ ││SES(EU) ││ │ │SES(IN)   ││
       │ │ Twilio (US)  │ │ ││Twilio  ││ │ │Twilio    ││
       │ │ APNs / FCM   │ │ ││APNs/FCM││ │ │APNs/FCM  ││
       │ └──────────────┘ │ │└────────┘│ │ └──────────┘│
       └──────────────────┘ └──────────┘ └─────────────┘
                 │                │              │
                 └────────────────┼──────────────┘
                                  │
                  ┌───────────────▼──────────────────┐
                  │  Global Suppression List         │
                  │  (opt-out tombstones replicated  │
                  │   to all cells; eventually       │
                  │   consistent ≤30s)               │
                  └──────────────────────────────────┘
```

### Routing

- Each user has a **home cell** assigned at signup based on country.
- A user-directory lookup at API ingress: `user_id → home_cell`. Caller submits to ANY cell; cell routes to home if mismatched.
- Cross-cell submission is rare (a US service emitting to a EU user — happens when business spans regions).

### What replicates cross-cell

- **Opt-out tombstones** — replicate globally. A user in EU who opts out via the EU cell must not receive sends initiated from the US cell. Asynchronous replication, eventual consistency ≤30s. Compliance accepts this bounded staleness because the originating system saw the opt-out at write time.
- **Suppression lists (bounces/complaints)** — same. Replicate globally; eventually consistent.
- **Templates** — Git is the single source of truth, deployed to every cell. Same templates everywhere (with per-cell locale variants).

### What does NOT replicate cross-cell

- **Notification log** — stays in home cell. Each user's history lives where they live.
- **User preferences (full)** — stays in home cell. Cross-cell sends do a cross-cell read (rare path).
- **Device tokens** — stays in home cell. APNs/FCM tokens are global IDs, but their association with the user is cell-local.

### Cross-cell sends (the hard case)

US-based service wants to notify an EU-based user. Three options:

| Option | How | Pros | Cons |
|---|---|---|---|
| **Cross-cell submission** | US service hits EU cell's API directly | Simple; no cross-cell traffic mid-pipeline | Caller needs to know user's home cell |
| **Cell forwarding** | US service hits US cell; US cell looks up user, forwards to EU cell | Caller-agnostic | Adds hop, US cell carries EU data briefly (compliance-permissible if encrypted in transit and not stored) |
| **Cross-cell pull (federation)** | EU cell pulls pending sends from a global queue | Decouples completely | Latency; global queue is a single point |

**My pick:** Cell forwarding. It's caller-agnostic and the US cell never persists EU data — just routes the encrypted payload to the EU cell which handles it locally.

### Provider per region

- **APNs** is global (Apple-operated). One config works everywhere.
- **FCM** is global (Google-operated). Same.
- **SES** has per-region endpoints; we have an SES tenant per region with locally-warmed IPs.
- **Twilio** has per-country compliance — we register sender numbers per country, with country-specific 10DLC / DLT registration.
- **Email IPs are region-specific** — IP reputation is built per-region. EU IPs send EU mail; US IPs send US mail.

### Failover

- A cell going down does NOT failover to other cells (residency violation). Instead:
  - Intra-cell HA (multi-AZ within cell) is what protects users.
  - For sustained cell outage: communication paused for that cell's users. PM and legal informed.
  - Critical exception: 2FA. We may temporarily route via a non-home cell with explicit policy allowing emergency cross-cell, audited and time-bounded.

### Schema migrations

- Per-cell, staggered. Cell-N is stable while cell-N+1 is migrating.
- Templates: one global Git repo; cells pick up new versions independently. We can hold EU on template version v9 while US runs v10.

### What "Staff signal" sounds like

> "I'd structure this as cells from day one, because notifications hit GDPR / DPDP / TCPA directly — there's no path to global compliance without per-region data residency. The cell is the unit of blast radius, deploy, and compliance. The hard part is opt-out tombstones — they MUST replicate globally, and within seconds, even though the rest of user state stays in-cell. That's a separate global subsystem with its own SLO. The cost is real — 30–40% more infra, more deploy complexity — but it's the only design that survives an EU regulator audit and an APNs outage at the same time."

---
