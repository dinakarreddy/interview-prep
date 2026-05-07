# Problem 12 — Notification System

The Adapter + Strategy + Decorator showcase. Tests how cleanly you can wrap third-party SDKs behind a uniform interface, route requests by channel, and stack cross-cutting wrappers (retry, rate limit, logging) without modifying the underlying senders.

## Interview opener

> **Interviewer:** Design a notification system that supports email, SMS, and push notifications.

---

## Clarification phase

> **You:** Multi-channel: email, SMS, push? Anything else — in-app, WhatsApp, voice?
>
> **Interviewer:** Email, SMS, push for the core. Mention how you'd add others.

> **You:** Single notification per request, or batched / templated?
>
> **Interviewer:** Both. A user-triggered "order confirmation" is one; a marketing campaign sends to millions.

> **You:** Templates with variable substitution?
>
> **Interviewer:** Yes — `Hello {{name}}, your order #{{id}} shipped.`

> **You:** User preferences — opt-out per channel, do-not-disturb hours?
>
> **Interviewer:** Yes; respect user preferences.

> **You:** Delivery guarantees — at-least-once, at-most-once, exactly-once?
>
> **Interviewer:** At-least-once (with idempotency at the receiver). Don't lose notifications.

> **You:** Retries on transient failures (vendor 503)?
>
> **Interviewer:** Yes — exponential backoff.

> **You:** Channel fallback — if push fails, fall back to SMS?
>
> **Interviewer:** Yes — make it configurable.

> **You:** Throughput? Marketing send to 10M users?
>
> **Interviewer:** Yes; design should scale to that.

> **You:** Rate limits per vendor (SendGrid 1000 req/s, Twilio 100/s)?
>
> **Interviewer:** Yes — respect those.

> **You:** Audit / observability — track delivery status?
>
> **Interviewer:** Yes — every notification's status should be queryable.

> **You:** Persistence?
>
> **Interviewer:** Discuss at end.

OK — design needs: pluggable channels (Adapter), per-user preferences, templating, retries with backoff, rate-limiting per vendor, async dispatch, fallback chains, observability.

---

## Refined requirements

### Functional
1. `send(notification)` — accept a notification request.
2. Resolve user preferences and template; expand into per-channel sends.
3. Dispatch to the right channel (email, SMS, push) via vendor SDK.
4. Retry transient failures with exponential backoff.
5. Fall back to next channel if primary fails permanently.
6. Track delivery status (queued, sending, sent, failed, bounced).
7. Bulk send (campaigns).
8. Honor rate limits per vendor.

### Non-functional
- **At-least-once delivery**.
- **High throughput** (10M sends per campaign).
- **Latency**: per-notification dispatch latency < 100ms (excluding vendor processing time).

### Out of scope (mentioned)
- WhatsApp / voice / in-app (extensions).
- Inbox UI / read receipts.
- Detailed analytics dashboards.

### Core operations
1. `NotificationService.send(NotificationRequest)` → returns `notificationId`.
2. `NotificationService.sendBatch(List<NotificationRequest>)`.
3. `NotificationService.status(notificationId)` → `DeliveryStatus`.

---

## Domain model

### Entities (have ID)
- **`Notification`** — one logical message. References user + template.
- **`Delivery`** — one attempt on one channel. A notification has many deliveries.
- **`Template`** — versioned template content.
- **`User`** — preferences and contact info.

### Value objects (immutable)
- **`Recipient(userId, email, phone, deviceTokens)`** — resolved contact info.
- **`RenderedMessage(subject, body, channelMetadata)`** — template rendered with vars.

### Enums
- `Channel` — `EMAIL`, `SMS`, `PUSH`, `WHATSAPP`, `VOICE`.
- `DeliveryStatus` — `QUEUED`, `SENDING`, `SENT`, `FAILED`, `BOUNCED`.
- `Priority` — `TRANSACTIONAL`, `MARKETING`. Different rate-limit lanes.

### Patterns
- **Adapter** — each vendor SDK behind a uniform `NotificationSender` interface.
- **Strategy** — channel selection per user / per notification.
- **Decorator** — wrap senders with retry, rate-limit, logging, metrics, dedup.
- **Chain of Responsibility** — fallback chain (push → SMS → email).
- **Observer** — events feed delivery status updates.
- **Producer-Consumer** — `BlockingQueue` between API and dispatcher workers.
- **Template Method** (extension) — render workflow.

---

## Class diagram

```
┌──────────────────────────────────────┐
│       <<interface>>                  │
│       NotificationSender             │
├──────────────────────────────────────┤
│ +send(Recipient, RenderedMessage)    │
│   : DeliveryResult                   │
│ +channel() : Channel                 │
└──────────────────────────────────────┘
            △
   ┌────────┼────────────┬──────────────┐
   │                     │              │
┌──────────────┐  ┌──────────────┐ ┌──────────────┐
│ EmailSender  │  │  SMSSender   │ │ PushSender   │  (Adapters)
│ (SendGrid)   │  │  (Twilio)    │ │ (FCM/APNs)   │
└──────────────┘  └──────────────┘ └──────────────┘

┌──────────────────────────────────────┐
│   <<abstract>> SenderDecorator       │
├──────────────────────────────────────┤
│ -wrapped : NotificationSender        │
└──────────────────────────────────────┘
            △
   ┌────────┼────────┬─────────┬──────────┐
   │                 │         │          │
┌──────────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐
│ RetryDecor   │ │RateLimit │ │ Logging  │ │ Metrics  │
└──────────────┘ └──────────┘ └──────────┘ └──────────┘

┌────────────────────────────────────────┐
│       NotificationDispatcher           │
├────────────────────────────────────────┤
│ -sendersByChannel : Map<Channel, ...>  │
│ -fallbackChain : ChannelFallback       │
│ -queue : BlockingQueue<NotifJob>       │
│ -workers : ExecutorService             │
├────────────────────────────────────────┤
│ +dispatch(Notification)                │
└────────────────────────────────────────┘

┌──────────────────────────┐    ┌──────────────────────┐
│       Template           │    │ UserPreferences      │
├──────────────────────────┤    ├──────────────────────┤
│ -id; -version            │    │ -userId              │
│ -subjectTemplate         │    │ -optOut: Set<Chan>   │
│ -bodyTemplate            │    │ -dndStart, -dndEnd   │
│ +render(vars)            │    │ -primaryChannel      │
└──────────────────────────┘    └──────────────────────┘

   record NotificationRequest(userId, templateId, vars, channel?, priority)
   record DeliveryResult(success, vendorRef, errorCode, retriable)
```

---

## Key abstractions

```java
public interface NotificationSender {
    DeliveryResult send(Recipient r, RenderedMessage m);
    Channel channel();
}

public interface ChannelFallback {
    /** Returns ordered list of channels to try, given user preferences. */
    List<Channel> chainFor(NotificationRequest req, UserPreferences prefs);
}

public interface TemplateEngine {
    RenderedMessage render(Template t, Map<String, Object> vars, Channel c);
}

public interface DeliveryStatusStore {
    void recordAttempt(String notificationId, Channel c, DeliveryStatus status, String vendorRef);
    DeliveryStatus current(String notificationId);
}
```

---

## Core code

### Value objects + enums

```java
public enum Channel        { EMAIL, SMS, PUSH, WHATSAPP, VOICE }
public enum DeliveryStatus { QUEUED, SENDING, SENT, FAILED, BOUNCED }
public enum Priority       { TRANSACTIONAL, MARKETING }

public record Recipient(String userId, String email, String phone, List<String> deviceTokens) { }
public record RenderedMessage(String subject, String body, Map<String, String> metadata) { }
public record DeliveryResult(boolean success, String vendorRef, String errorCode, boolean retriable) { }
public record NotificationRequest(
    String idempotencyKey,
    String userId,
    String templateId,
    Map<String, Object> vars,
    Optional<Channel> preferredChannel,   // null = use user's preference
    Priority priority
) { }
```

### Adapters: per-vendor `NotificationSender`

```java
public final class SendGridEmailSender implements NotificationSender {
    private final SendGridClient client;     // third-party SDK
    private final String fromAddress;

    public SendGridEmailSender(SendGridClient client, String fromAddress) {
        this.client = client; this.fromAddress = fromAddress;
    }

    @Override
    public DeliveryResult send(Recipient r, RenderedMessage m) {
        try {
            var msg = new SendGridMessage();
            msg.setTo(r.email());
            msg.setFrom(fromAddress);
            msg.setSubject(m.subject());
            msg.setHtml(m.body());
            SendGridResponse resp = client.mail(msg);
            boolean success = resp.statusCode() < 300;
            return new DeliveryResult(success, resp.messageId(),
                                      success ? null : "vendor_" + resp.statusCode(),
                                      resp.statusCode() == 503 || resp.statusCode() == 429);
        } catch (IOException e) {
            return new DeliveryResult(false, null, "io_error", true);
        }
    }

    @Override public Channel channel() { return Channel.EMAIL; }
}

public final class TwilioSMSSender implements NotificationSender {
    private final TwilioClient client;
    private final String fromNumber;

    public TwilioSMSSender(TwilioClient client, String fromNumber) {
        this.client = client; this.fromNumber = fromNumber;
    }

    @Override
    public DeliveryResult send(Recipient r, RenderedMessage m) {
        // SMS doesn't have a separate subject — concatenate
        String text = m.subject().isEmpty() ? m.body() : m.subject() + "\n" + m.body();
        try {
            TwilioMessage msg = client.messageCreate(fromNumber, r.phone(), text);
            return new DeliveryResult("queued".equals(msg.getStatus()) || "sent".equals(msg.getStatus()),
                                      msg.getSid(), null, false);
        } catch (TwilioException e) {
            return new DeliveryResult(false, null, e.getCode(),
                                      e.getCode().equals("rate_limit") || e.getCode().equals("server_error"));
        }
    }

    @Override public Channel channel() { return Channel.SMS; }
}

public final class FCMPushSender implements NotificationSender {
    private final FcmClient client;

    public FCMPushSender(FcmClient client) { this.client = client; }

    @Override
    public DeliveryResult send(Recipient r, RenderedMessage m) {
        for (String token : r.deviceTokens()) {
            FcmPayload payload = FcmPayload.builder()
                .token(token).title(m.subject()).body(m.body())
                .build();
            FcmResult result = client.dispatch(payload);
            // Simplified — real impl tracks per-token results
            if (result.success()) return new DeliveryResult(true, result.messageId(), null, false);
        }
        return new DeliveryResult(false, null, "no_active_device", false);
    }

    @Override public Channel channel() { return Channel.PUSH; }
}
```

Each adapter:
- Wraps the vendor SDK behind the uniform `NotificationSender` interface.
- Translates the domain model (`Recipient`, `RenderedMessage`) to the vendor's request shape.
- Translates the vendor's response back to `DeliveryResult`, including a `retriable` flag for the retry decorator.
- Vendor-specific quirks (SMS lacks subject, push fans out across device tokens) handled inside.

### Decorators (cross-cutting concerns)

```java
public abstract class SenderDecorator implements NotificationSender {
    protected final NotificationSender wrapped;
    protected SenderDecorator(NotificationSender wrapped) { this.wrapped = wrapped; }
    @Override public Channel channel() { return wrapped.channel(); }
}

public final class RetryDecorator extends SenderDecorator {
    private final int maxAttempts;
    private final Duration baseBackoff;

    public RetryDecorator(NotificationSender wrapped, int maxAttempts, Duration baseBackoff) {
        super(wrapped);
        this.maxAttempts = maxAttempts;
        this.baseBackoff = baseBackoff;
    }

    @Override
    public DeliveryResult send(Recipient r, RenderedMessage m) {
        DeliveryResult result = null;
        for (int attempt = 1; attempt <= maxAttempts; attempt++) {
            result = wrapped.send(r, m);
            if (result.success()) return result;
            if (!result.retriable()) return result;     // permanent failure → give up
            if (attempt < maxAttempts) sleepWithJitter(attempt);
        }
        return result;
    }

    private void sleepWithJitter(int attempt) {
        long ms = (long) (baseBackoff.toMillis() * Math.pow(2, attempt - 1));
        ms += ThreadLocalRandom.current().nextLong(ms / 2);   // 50% jitter
        try { Thread.sleep(ms); } catch (InterruptedException ignored) { Thread.currentThread().interrupt(); }
    }
}

public final class RateLimitDecorator extends SenderDecorator {
    private final RateLimiter limiter;   // token bucket

    public RateLimitDecorator(NotificationSender wrapped, RateLimiter limiter) {
        super(wrapped);
        this.limiter = limiter;
    }

    @Override
    public DeliveryResult send(Recipient r, RenderedMessage m) {
        try {
            limiter.acquire();   // blocks until a token is available
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            return new DeliveryResult(false, null, "interrupted", false);
        }
        return wrapped.send(r, m);
    }
}

public final class LoggingDecorator extends SenderDecorator {
    private static final Logger log = LoggerFactory.getLogger(LoggingDecorator.class);

    public LoggingDecorator(NotificationSender wrapped) { super(wrapped); }

    @Override
    public DeliveryResult send(Recipient r, RenderedMessage m) {
        long start = System.nanoTime();
        DeliveryResult result = wrapped.send(r, m);
        long elapsed = (System.nanoTime() - start) / 1_000_000;
        log.info("send channel={} userId={} success={} ref={} latency={}ms",
                 channel(), r.userId(), result.success(), result.vendorRef(), elapsed);
        return result;
    }
}

public final class MetricsDecorator extends SenderDecorator {
    private final MetricsRegistry metrics;

    public MetricsDecorator(NotificationSender wrapped, MetricsRegistry m) { super(wrapped); this.metrics = m; }

    @Override
    public DeliveryResult send(Recipient r, RenderedMessage m) {
        long start = System.nanoTime();
        DeliveryResult result = wrapped.send(r, m);
        metrics.timer("notification.send", "channel", channel().name())
               .record(System.nanoTime() - start, TimeUnit.NANOSECONDS);
        metrics.counter("notification.send.outcome",
                        "channel", channel().name(),
                        "success", String.valueOf(result.success())).increment();
        return result;
    }
}
```

Composition example:
```java
NotificationSender emailSender = new MetricsDecorator(
    new LoggingDecorator(
        new RateLimitDecorator(
            new RetryDecorator(
                new SendGridEmailSender(sendGridClient, "noreply@x.com"),
                3, Duration.ofMillis(500)
            ),
            new TokenBucketLimiter(1000, 1000)   // 1000/sec
        )
    ),
    metricsRegistry
);
```

The wrapped order is the order of execution: outer decorators wrap inner decorators. Metrics outermost = capture total time including retry. Rate limit before retry = retries don't bypass the limiter.

### Channel fallback (Chain of Responsibility)

```java
public final class PreferenceBasedFallback implements ChannelFallback {
    @Override
    public List<Channel> chainFor(NotificationRequest req, UserPreferences prefs) {
        // Start with primary channel preference, then fall back
        List<Channel> chain = new ArrayList<>();
        if (req.preferredChannel().isPresent()) chain.add(req.preferredChannel().get());
        else chain.add(prefs.primaryChannel());
        // Append remaining channels in user's order, excluding opt-outs
        for (Channel c : prefs.fallbackOrder()) {
            if (!chain.contains(c) && !prefs.optOut().contains(c)) chain.add(c);
        }
        return chain;
    }
}
```

### Templates

```java
public record Template(String id, int version, String subjectTemplate, String bodyTemplate, Channel channel) { }

public final class MustacheTemplateEngine implements TemplateEngine {
    private final TemplateRepository templates;

    public MustacheTemplateEngine(TemplateRepository repo) { this.templates = repo; }

    @Override
    public RenderedMessage render(Template t, Map<String, Object> vars, Channel c) {
        String subject = renderString(t.subjectTemplate(), vars);
        String body = renderString(t.bodyTemplate(), vars);
        return new RenderedMessage(subject, body, Map.of());
    }

    private String renderString(String template, Map<String, Object> vars) {
        // Trivial mustache-like substitution; production uses Mustache.java or Handlebars.
        String result = template;
        for (var e : vars.entrySet()) result = result.replace("{{" + e.getKey() + "}}", String.valueOf(e.getValue()));
        return result;
    }
}
```

Templates are channel-aware: an email template differs from an SMS template (length, formatting). Either: separate template per channel, or a single template with channel-specific overrides.

### Dispatcher (Producer-Consumer)

```java
public final class NotificationDispatcher {
    private final Map<Channel, NotificationSender> sendersByChannel;
    private final ChannelFallback fallback;
    private final TemplateEngine templates;
    private final TemplateRepository templateRepo;
    private final UserPreferenceService prefs;
    private final RecipientResolver recipients;
    private final DeliveryStatusStore statusStore;
    private final BlockingQueue<NotificationRequest> queue;
    private final ExecutorService workers;

    public NotificationDispatcher(Map<Channel, NotificationSender> senders,
                                  ChannelFallback fallback,
                                  TemplateEngine templates, TemplateRepository tr,
                                  UserPreferenceService prefs, RecipientResolver rr,
                                  DeliveryStatusStore status,
                                  int queueCapacity, int workerThreads) {
        this.sendersByChannel = Map.copyOf(senders);
        this.fallback = fallback;
        this.templates = templates;
        this.templateRepo = tr;
        this.prefs = prefs;
        this.recipients = rr;
        this.statusStore = status;
        this.queue = new ArrayBlockingQueue<>(queueCapacity);
        this.workers = Executors.newFixedThreadPool(workerThreads, namedThreadFactory("notif-worker"));
        for (int i = 0; i < workerThreads; i++) workers.submit(this::workerLoop);
    }

    public String submit(NotificationRequest req) {
        statusStore.recordAttempt(req.idempotencyKey(), null, DeliveryStatus.QUEUED, null);
        if (!queue.offer(req)) throw new BackpressureException("Queue full");
        return req.idempotencyKey();
    }

    private void workerLoop() {
        while (!Thread.currentThread().isInterrupted()) {
            try {
                NotificationRequest req = queue.take();
                process(req);
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
                return;
            } catch (Exception e) {
                // Don't let worker die
                log.error("worker error", e);
            }
        }
    }

    private void process(NotificationRequest req) {
        UserPreferences userPrefs = prefs.forUser(req.userId());
        if (isInDND(userPrefs) && req.priority() == Priority.MARKETING) {
            statusStore.recordAttempt(req.idempotencyKey(), null, DeliveryStatus.FAILED, "dnd");
            return;
        }

        Recipient recipient = recipients.resolve(req.userId());
        List<Channel> chain = fallback.chainFor(req, userPrefs);

        for (Channel c : chain) {
            NotificationSender sender = sendersByChannel.get(c);
            if (sender == null) continue;
            Template t = templateRepo.findByIdAndChannel(req.templateId(), c).orElse(null);
            if (t == null) continue;

            statusStore.recordAttempt(req.idempotencyKey(), c, DeliveryStatus.SENDING, null);
            RenderedMessage msg = templates.render(t, req.vars(), c);
            DeliveryResult result = sender.send(recipient, msg);

            if (result.success()) {
                statusStore.recordAttempt(req.idempotencyKey(), c, DeliveryStatus.SENT, result.vendorRef());
                return;
            }

            statusStore.recordAttempt(req.idempotencyKey(), c, DeliveryStatus.FAILED, result.errorCode());
            // Permanent failure: don't try other channels with the same recipient field if the error is recipient-related
            // Otherwise: continue to next channel in fallback chain.
        }
    }

    private boolean isInDND(UserPreferences prefs) {
        if (prefs.dndStart() == null) return false;
        LocalTime now = LocalTime.now(prefs.timezone());
        return now.isAfter(prefs.dndStart()) || now.isBefore(prefs.dndEnd());
    }
}
```

### `NotificationService` — entry point

```java
public final class NotificationService {
    private final NotificationDispatcher dispatcher;
    private final IdempotencyStore idempotencyStore;

    public NotificationService(NotificationDispatcher d, IdempotencyStore is) {
        this.dispatcher = d; this.idempotencyStore = is;
    }

    public String send(NotificationRequest req) {
        // Idempotency check
        if (req.idempotencyKey() != null) {
            Optional<String> existing = idempotencyStore.lookup(req.idempotencyKey());
            if (existing.isPresent()) return existing.get();
        }
        String id = dispatcher.submit(req);
        if (req.idempotencyKey() != null) idempotencyStore.record(req.idempotencyKey(), id);
        return id;
    }

    public void sendBatch(List<NotificationRequest> requests) {
        for (NotificationRequest req : requests) send(req);
    }
}
```

---

## Concurrency strategy

### What's contested
- The dispatch queue (multi-producer, multi-consumer).
- Per-vendor rate limiters (atomic token buckets).
- The delivery status store (concurrent updates per notification).

### Strategy
- **`BlockingQueue` (multi-producer / multi-consumer)** — `ArrayBlockingQueue` for bounded throughput.
- **Token-bucket per vendor** — atomic CAS updates to the token count and last-refill timestamp.
- **`ConcurrentHashMap`-based status store** — atomic per-key writes; readers see consistent state.

### Backpressure
- Queue is bounded. When full, `submit` throws `BackpressureException`.
- Producers handle: log, persist to DB for later replay, or reject.
- For marketing campaigns: rate-limit at the entry point (don't enqueue 10M at once).

### Async failure isolation
- Worker exceptions caught — workers don't die.
- Vendor SDK exceptions wrapped in `DeliveryResult(success=false)`.
- Catastrophic failures (worker thread crash) detected by health checks; restarted.

### Ordering
- Notifications are independent — no cross-notification ordering required.
- Within one notification: fallback chain is sequential.

---

## Edge cases / gotchas

| Case | Handling |
|---|---|
| **User opts out of all channels** | All `chainFor` returns empty; status recorded as no-channel-available. |
| **Template missing for channel** | Skip that channel in fallback; try next. |
| **All channels fail** | Final status FAILED; alert ops if pattern emerges. |
| **DND hours for transactional** | Override DND for transactional priority. |
| **Idempotency key collision (same key, different content)** | Either treat as duplicate (return original) or reject. Specify the contract. |
| **Vendor returns "queued" but never delivers** | Need vendor webhooks for delivery confirmation; update status async. |
| **Push token expired** | Vendor returns "invalid token"; mark token dead, try next token. |
| **Long-running send blocks worker** | Worker count ≥ peak concurrent sends; consider per-channel worker pools to isolate. |
| **Retry infinite loop on a always-retriable error** | Bounded `maxAttempts`. |
| **Thundering herd after vendor outage** | Token bucket smooths recovery; retry queues drain gradually. |
| **Same notification ID processed twice** | Status store keyed by `(notificationId, channel, attemptNo)` — or a single transition; must dedupe. |

---

## Extensions / interview follow-ups

### "Add WhatsApp / voice."
- New adapter: `WhatsAppSender implements NotificationSender`.
- New channel enum entry.
- Dispatcher map gains the new entry.
- No changes to existing senders or dispatcher logic.

### "Schedule a notification for later (at 9am tomorrow)."
- Add a `scheduledAt` field; persistent queue (DB table) with `scheduled_at` index.
- Scheduler thread polls due notifications and enqueues to the dispatch queue.

### "Handle vendor webhooks (delivery confirmation, bounces)."
- Webhook endpoint receives vendor events.
- Match by `vendorRef` (saved on send); update status store.
- Trigger retries on bounces (maybe try next channel).

### "Quiet hours per timezone."
- `UserPreferences.timezone`; DND check uses local time.
- Already in the design.

### "Multi-tenant — different orgs use different vendors."
- `NotificationSender` resolved per tenant: `senders.get(tenantId).get(channel)`.
- Tenant-scoped templates, preferences, vendors.

### "Why Decorator instead of inheritance?"
- Each cross-cutting concern (retry, log, rate limit) is independent.
- Inheritance forces a single hierarchy: `RetryingLoggingRateLimitedSender extends Sender`. Combinatorial explosion.
- Decorator stacks freely; order is composition-time choice.

### "Why Adapter for vendor SDKs?"
- Each SDK has its own request/response shape.
- Adapter normalizes to `NotificationSender`.
- Swapping SendGrid → Mailgun is one new adapter; everything else unchanged.

### "What's the difference between Decorator and Chain of Responsibility here?"
- **Decorator** wraps the same sender with extra behavior (retry, log).
- **Chain of Responsibility** tries multiple senders in fallback order until one succeeds.
- Different patterns at different scopes.

### "How do you avoid sending duplicates after a crash?"
- Idempotency keys + persistent status store.
- On restart, re-process only notifications in `QUEUED` or `SENDING` status; senders' idempotency keys (passed to vendors) prevent vendor-side duplicates.

---

## Scaling

A single-process system handles ~1k sends/sec. For 10M-user campaigns:

### Sharded dispatchers
- Multiple dispatcher instances; partition by `userId` hash.
- Each instance has its own queue + workers.

### Persistent queue (Kafka)
- Replace in-memory `BlockingQueue` with Kafka topic.
- Consumers (dispatchers) scale independently of producers.
- At-least-once delivery guaranteed by Kafka.

### Per-channel queues
- Email's vendor rate is 1k/s; SMS is 100/s. Mixed queue means SMS bottlenecks email.
- Per-channel queues + workers isolate.

### Bulk send optimization
- Some vendors support batch APIs (SendGrid: 1000 messages per request).
- `BatchEmailSender` accumulates outgoing messages and flushes when batch full or timer fires.

### Multi-region
- Active-active with sticky routing per user.
- Cross-region replication of templates and preferences.

### Observability at scale
- Per-channel, per-vendor latency histograms.
- Delivery success rate as SLO.
- Alert on success-rate dips, queue depth spikes.

---

## Persistence tradeoffs

Notification systems are write-heavy with audit requirements; the data has different shapes that benefit from different stores.

### What persistent data exists
- **Templates**: small, versioned, occasionally edited.
- **User preferences**: per-user, occasional updates.
- **Notification requests** (queued / scheduled): transient until processed.
- **Delivery records**: append-only audit log; what was sent to whom, when, via which channel, with what outcome.
- **Idempotency table**: short-TTL key-result pairs.

### Option 1 — Relational (PostgreSQL) for everything
- `templates`, `user_preferences`, `notifications`, `deliveries`, `idempotency`.
- **Pros**: simple. Consistency by default. Good for moderate volume.
- **Cons**: insert pressure on `deliveries` table at scale (millions/day). Partition by date or shard by user.
- **When to pick**: starting point. Scales further than most expect.

### Option 2 — Polyglot (recommended for scale)
- **Templates + preferences**: PostgreSQL (small, queryable).
- **Notification queue**: Kafka (durable, scalable).
- **Delivery records**: time-series store (Cassandra, ScyllaDB, BigTable) — append-optimized, queryable by user + time.
- **Idempotency**: Redis with TTL (24h is typical).
- **Status lookup cache**: Redis for hot reads of recent notifications.

### Option 3 — Event-sourced
- Every notification is a sequence of events: `RequestReceived`, `RenderingStarted`, `DispatchingTo(channel)`, `VendorAccepted`, `Delivered` / `Bounced`.
- Status is the latest event.
- **Pros**: complete history. Time-travel debugging ("why did this not deliver?").
- **Cons**: storage volume; query latency for status (mitigated by snapshots).
- **When to pick**: complex multi-step workflows; audit-heavy compliance environments.

### Option 4 — DynamoDB for delivery records
- Partition key = `userId`; sort key = `timestamp`. Naturally queryable per user.
- TTL on records older than retention period (90 days, etc.).
- **When to pick**: cloud-native; predictable access pattern.

### Schema choices for delivery records
- **Wide row** (one row per delivery attempt) — simple, audit-friendly, large.
- **Aggregated** (one row per notification with attempts in JSON) — compact, harder to query.
- Wide row + retention policy is the production default.

### Persistent queue
For at-least-once delivery, the queue must be durable:
- **Kafka** — strong durability, replay-able, supports many consumers.
- **RabbitMQ** — easier to set up, weaker on ordered replay.
- **DB table as queue** — simple but doesn't scale; OK for low volume.
- **Redis Streams** — middle ground.

### Concurrency under persistence
- **Idempotency lookup**: must be done before the send. SQL `INSERT ... ON CONFLICT` or Redis `SETNX` ensures atomic "first one wins."
- **Status updates**: append-only delivery log; no in-place updates → no concurrency issues.
- **Template fetch**: cached, periodically refreshed.

### Recommendation
- **Default**: PostgreSQL for templates and preferences; Kafka for the queue; PostgreSQL or Cassandra for delivery records; Redis for idempotency.
- **At scale**: switch delivery records to Cassandra; status cache in Redis; multi-region Kafka.

---

## Talking points for the interview

- "Each vendor SDK is an Adapter. The domain depends on `NotificationSender`, not on Twilio's `MessageCreator`. Swapping vendors is a one-class change."
- "Cross-cutting concerns are Decorators — retry, rate limit, logging, metrics. Order matters: rate-limit before retry so retries respect the limit."
- "Channel fallback is a Chain of Responsibility — try push, fall to SMS, fall to email. Each step decides if it can serve."
- "Idempotency keys at the API gateway prevent duplicate processing on retries. Vendor-side keys prevent duplicate delivery."
- "Bounded queue with backpressure rejects rather than buffers — logging shouldn't bring down the application."
- "Async dispatch decouples the API thread from vendor latency. Workers process from the queue; vendor calls happen there, not in the API thread."
- "DND is timezone-aware and overridden by transactional priority. Marketing in the middle of someone's night is the textbook reason this fails in production."

---

## Summary

Patterns load-bearing: **Adapter** (vendor SDKs), **Strategy** (channel selection), **Decorator** (cross-cutting wrappers), **Chain of Responsibility** (fallback), **Producer-Consumer** (async dispatch).

The mental model:
1. Request comes in → idempotency check → enqueue.
2. Worker picks up → render template → resolve channel → dispatch to adapter chain.
3. Adapter chain: rate-limit → retry → vendor SDK.
4. Failure on one channel → fall back to the next.
5. Final status persisted.

The senior signal is the **layered concerns**: each adapter does one thing; each decorator does one thing; the dispatcher orchestrates without knowing the vendors. Add WhatsApp = one new adapter; tweak retry policy = swap the decorator parameters.
