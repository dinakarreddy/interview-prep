# Problem 4 — Logger

The cross-cutting-concerns problem. Tests whether you can compose multiple patterns (Strategy, Chain, Observer, async pipeline) into a coherent design.

## Interview opener

> **Interviewer:** Design a logging library.

---

## Clarification phase

> **You:** Quick scoping. Multi-level — DEBUG, INFO, WARN, ERROR — with a configurable threshold?
>
> **Interviewer:** Yes.

> **You:** Multiple destinations — console, file, network — and configurable per logger?
>
> **Interviewer:** Yes. Should be pluggable.

> **You:** Should the log format itself be pluggable? Plain text vs JSON?
>
> **Interviewer:** Yes — formatter as a separate concern from destination.

> **You:** Single logger or hierarchy? Like Java util logging — `com.foo` inheriting config from `com`?
>
> **Interviewer:** Hierarchy is nice but optional. A simple per-name registry is fine.

> **You:** Concurrency — many threads logging simultaneously?
>
> **Interviewer:** Yes — high throughput. Don't bottleneck on logging.

> **You:** Should logging be synchronous or asynchronous?
>
> **Interviewer:** Both modes; the user picks. Asynchronous shouldn't lose messages on shutdown.

> **You:** What about MDC / contextual fields — e.g., trace ID per request?
>
> **Interviewer:** Yes — should support thread-local context that gets attached to every log line on that thread.

> **You:** Filtering beyond level? E.g., "drop noisy messages from class X"?
>
> **Interviewer:** Pluggable filters, yes.

> **You:** Out of scope — file rotation, log shipping?
>
> **Interviewer:** Mention but don't implement.

OK — the design needs: levels, multiple appenders, formatters, filters, thread-local context, and async writing.

---

## Refined requirements

### Functional
1. Get a logger by name: `Logger log = LoggerFactory.getLogger("com.foo.Bar")`.
2. Log at levels: `log.debug(...)`, `log.info(...)`, `log.warn(...)`, `log.error(...)`.
3. Each logger has a level threshold; messages below are dropped.
4. Each logger has a list of **appenders** (file, console, network).
5. Each appender has a **formatter** (text, JSON) and optional **filters**.
6. Thread-local context (MDC): `MDC.put("traceId", "abc"); MDC.clear()`.
7. Both synchronous and asynchronous appenders.
8. Graceful shutdown — flush pending messages.

### Non-functional
- Hot path (level-disabled message) must be near-zero cost.
- High throughput.
- Lossless on graceful shutdown.

### Out of scope (mentioned)
- File rotation / size limits.
- Log shipping (sending to ELK / Splunk).
- Hierarchical configuration inheritance.

---

## Domain model

### Entities (loosely)
- **`LoggerFactory`** — singleton-ish registry of loggers.
- **`Logger`** — named, with level + appenders.

### Value objects
- **`LogEvent(level, time, thread, loggerName, message, throwable, mdcSnapshot)`** — immutable per emission.
- **`Level`** — enum.

### Strategy interfaces
- **`Appender`** — destination (console, file, network).
- **`Formatter`** — turns a `LogEvent` into a string (or JSON).
- **`LogFilter`** — boolean predicate; can drop events.

### Patterns at play
- **Singleton** — `LoggerFactory` (typically a single registry).
- **Strategy** — `Appender`, `Formatter`, `LogFilter`.
- **Chain of Responsibility** — filters chain on each appender.
- **Decorator** — async wrapper around any appender; metric-emitting wrapper; etc.
- **Observer** — appenders subscribed to a logger's events.
- **Composite** — log routing through a tree of loggers (extension).

---

## Class diagram

```
┌──────────────────────────┐
│      LoggerFactory       │  (singleton)
├──────────────────────────┤
│ -loggers : Map<String,..>│
│ -rootLevel : Level       │
├──────────────────────────┤
│ +getLogger(name) : Logger│
│ +shutdown()              │
└──────────────────────────┘
            │ creates
            ▼
┌──────────────────────────────────┐
│            Logger                │
├──────────────────────────────────┤
│ -name : String                   │
│ -level : Level (volatile)        │
│ -appenders : List<Appender>      │
├──────────────────────────────────┤
│ +debug/info/warn/error(...)      │
│ +log(level, msg, throwable)      │
│ +setLevel(Level)                 │
│ +addAppender(Appender)           │
└──────────────────────────────────┘
            │ ◇ holds
            ▼
┌──────────────────────────────────┐
│       <<interface>> Appender     │
├──────────────────────────────────┤
│ +append(LogEvent)                │
│ +close()                         │
└──────────────────────────────────┘
            △
   ┌────────┼─────────────────────┬────────────────┐
   │        │                     │                │
┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│ConsoleAppendr│  │ FileAppender │  │ AsyncAppender│  │HttpAppender  │
│              │  │              │  │ (Decorator)  │  │              │
└──────────────┘  └──────────────┘  └──────────────┘  └──────────────┘
                                            │ wraps
                                            ▼
                                    Appender (any)


┌────────────────────────┐    ┌──────────────────────┐
│ <<interface>> Formatter │    │<<interface>> LogFilter│
├────────────────────────┤    ├──────────────────────┤
│ +format(LogEvent):String│   │ +accept(LogEvent):bool│
└────────────────────────┘    └──────────────────────┘
        △                            △
  ┌─────┴─────┐                ┌─────┴──────────┐
┌──────────────┐ ┌───────────┐ ┌───────────────┐ ┌────────────────┐
│ TextFormatter│ │JsonFormatter│  │RegexFilter   │ │RateLimitFilter │
└──────────────┘ └───────────┘ └───────────────┘ └────────────────┘

┌──────────────────────┐
│         MDC          │  (thread-local context)
├──────────────────────┤
│ +put(K, V)           │
│ +get(K) : V          │
│ +clear()             │
│ +snapshot() : Map    │
└──────────────────────┘
```

---

## Key abstractions

```java
public enum Level { DEBUG(0), INFO(1), WARN(2), ERROR(3);
    public final int rank;
    Level(int r) { this.rank = r; }
}

public record LogEvent(
    Level level,
    Instant time,
    String thread,
    String loggerName,
    String message,
    Throwable throwable,         // nullable
    Map<String, String> mdc      // immutable snapshot
) { }

public interface Appender extends AutoCloseable {
    void append(LogEvent event);
    @Override void close();
}

public interface Formatter {
    String format(LogEvent event);
}

public interface LogFilter {
    boolean accept(LogEvent event);
}
```

---

## Core code

### `MDC` (thread-local context)

```java
public final class MDC {
    private static final ThreadLocal<Map<String, String>> CONTEXT =
        ThreadLocal.withInitial(HashMap::new);

    public static void put(String key, String value) { CONTEXT.get().put(key, value); }
    public static String get(String key)             { return CONTEXT.get().get(key); }
    public static void remove(String key)            { CONTEXT.get().remove(key); }
    public static void clear()                       { CONTEXT.get().clear(); }
    public static Map<String, String> snapshot()     { return Map.copyOf(CONTEXT.get()); }
}
```

`ThreadLocal` is the right tool — each thread has independent context. Always pair `put` with `clear` (or `remove`) at the end of the request lifecycle to prevent leaks across pooled threads.

### `Logger`

```java
public final class Logger {
    private final String name;
    private volatile Level level;
    private final List<Appender> appenders = new CopyOnWriteArrayList<>();

    Logger(String name, Level level) { this.name = name; this.level = level; }

    public void debug(String msg)              { log(Level.DEBUG, msg, null); }
    public void info(String msg)               { log(Level.INFO, msg, null); }
    public void warn(String msg)               { log(Level.WARN, msg, null); }
    public void error(String msg, Throwable t) { log(Level.ERROR, msg, t); }

    // Lazy-eval form for expensive messages
    public void debug(Supplier<String> msg) {
        if (isEnabled(Level.DEBUG)) log(Level.DEBUG, msg.get(), null);
    }

    public boolean isEnabled(Level l) { return l.rank >= level.rank; }

    public void log(Level lvl, String msg, Throwable t) {
        if (!isEnabled(lvl)) return;   // hot-path bail — must be cheap
        LogEvent event = new LogEvent(
            lvl, Instant.now(), Thread.currentThread().getName(),
            name, msg, t, MDC.snapshot()
        );
        for (Appender a : appenders) {
            try { a.append(event); }
            catch (Exception e) { /* never let an appender throw kill the caller */ }
        }
    }

    public void setLevel(Level l)              { this.level = l; }
    public void addAppender(Appender a)        { appenders.add(a); }
    public void removeAppender(Appender a)     { appenders.remove(a); }

    void closeAppenders() { for (Appender a : appenders) try { a.close(); } catch (Exception ignored) {} }
}
```

The `level` field is `volatile` so changes propagate. The appender list is `CopyOnWriteArrayList` because it's read-mostly (every log iterates it) and rarely mutated (at config time).

### `LoggerFactory`

```java
public final class LoggerFactory {
    private static final LoggerFactory INSTANCE = new LoggerFactory();
    private final ConcurrentMap<String, Logger> loggers = new ConcurrentHashMap<>();
    private volatile Level defaultLevel = Level.INFO;

    private LoggerFactory() { }

    public static Logger getLogger(String name) {
        return INSTANCE.loggers.computeIfAbsent(name, n -> new Logger(n, INSTANCE.defaultLevel));
    }

    public static Logger getLogger(Class<?> clazz)   { return getLogger(clazz.getName()); }
    public static void setDefaultLevel(Level l)      { INSTANCE.defaultLevel = l; }

    public static void shutdown() {
        for (Logger l : INSTANCE.loggers.values()) l.closeAppenders();
    }
}
```

### Formatters (Strategy)

```java
public final class TextFormatter implements Formatter {
    private final DateTimeFormatter timeFormat = DateTimeFormatter.ISO_INSTANT;
    @Override
    public String format(LogEvent e) {
        var sb = new StringBuilder();
        sb.append(timeFormat.format(e.time())).append(' ');
        sb.append('[').append(e.thread()).append("] ");
        sb.append(e.level()).append(' ');
        sb.append(e.loggerName()).append(" - ");
        sb.append(e.message());
        if (!e.mdc().isEmpty()) sb.append(' ').append(e.mdc());
        if (e.throwable() != null) {
            sb.append('\n');
            var sw = new StringWriter();
            e.throwable().printStackTrace(new PrintWriter(sw));
            sb.append(sw);
        }
        return sb.toString();
    }
}

public final class JsonFormatter implements Formatter {
    @Override
    public String format(LogEvent e) {
        // Use a real JSON library in production. Pseudocode shown.
        return new JsonObject()
            .add("ts", e.time().toString())
            .add("level", e.level().name())
            .add("thread", e.thread())
            .add("logger", e.loggerName())
            .add("msg", e.message())
            .add("mdc", e.mdc())
            .add("error", e.throwable() != null ? e.throwable().toString() : null)
            .toJson();
    }
}
```

### Appenders (Strategy + Chain of Responsibility for filters)

```java
public abstract class AbstractAppender implements Appender {
    private final Formatter formatter;
    private final List<LogFilter> filters;

    protected AbstractAppender(Formatter formatter, List<LogFilter> filters) {
        this.formatter = formatter;
        this.filters = List.copyOf(filters);
    }

    @Override
    public final void append(LogEvent event) {
        for (LogFilter f : filters) if (!f.accept(event)) return;
        write(formatter.format(event));
    }

    protected abstract void write(String formatted);
}

public final class ConsoleAppender extends AbstractAppender {
    public ConsoleAppender(Formatter f, List<LogFilter> filters) { super(f, filters); }
    @Override protected void write(String s) { System.out.println(s); }
    @Override public void close() { /* nothing to release */ }
}

public final class FileAppender extends AbstractAppender {
    private final BufferedWriter writer;
    public FileAppender(Path path, Formatter f, List<LogFilter> filters) throws IOException {
        super(f, filters);
        this.writer = Files.newBufferedWriter(path, StandardOpenOption.CREATE, StandardOpenOption.APPEND);
    }
    @Override
    protected synchronized void write(String s) {   // synchronous file write
        try { writer.write(s); writer.newLine(); writer.flush(); }
        catch (IOException e) { /* report internally */ }
    }
    @Override public synchronized void close() {
        try { writer.close(); } catch (IOException ignored) { }
    }
}
```

### Async appender (Decorator)

```java
public final class AsyncAppender implements Appender {
    private final Appender delegate;
    private final BlockingQueue<LogEvent> queue;
    private final Thread worker;
    private volatile boolean running = true;

    public AsyncAppender(Appender delegate, int queueCapacity) {
        this.delegate = delegate;
        this.queue = new ArrayBlockingQueue<>(queueCapacity);
        this.worker = new Thread(this::drain, "log-async-" + System.nanoTime());
        this.worker.setDaemon(true);
        this.worker.start();
    }

    @Override
    public void append(LogEvent e) {
        if (!queue.offer(e)) {
            // Queue full — drop or block? Drop is the safe default for logging.
            // Log internally that we dropped.
        }
    }

    private void drain() {
        while (running || !queue.isEmpty()) {
            try {
                LogEvent e = queue.poll(500, TimeUnit.MILLISECONDS);
                if (e != null) delegate.append(e);
            } catch (InterruptedException ie) {
                Thread.currentThread().interrupt();
                return;
            }
        }
    }

    @Override
    public void close() {
        running = false;
        worker.interrupt();
        try { worker.join(5_000); } catch (InterruptedException ignored) {}
        delegate.close();
    }
}
```

This is **Decorator** — `AsyncAppender` wraps any `Appender`. The wrapped appender knows nothing about asynchrony. You can stack: `new AsyncAppender(new MetricsAppender(new FileAppender(...)))`.

### Filters (Chain of Responsibility)

```java
public final class LevelFilter implements LogFilter {
    private final Level minLevel;
    public LevelFilter(Level l) { this.minLevel = l; }
    @Override public boolean accept(LogEvent e) { return e.level().rank >= minLevel.rank; }
}

public final class RegexFilter implements LogFilter {
    private final Pattern pattern;
    private final boolean exclude;   // true = drop matches, false = keep matches
    public RegexFilter(String regex, boolean exclude) {
        this.pattern = Pattern.compile(regex);
        this.exclude = exclude;
    }
    @Override
    public boolean accept(LogEvent e) {
        boolean match = pattern.matcher(e.message()).find();
        return exclude ? !match : match;
    }
}

public final class RateLimitFilter implements LogFilter {
    private final long perSecond;
    private final AtomicLong tokens;
    private volatile long lastRefillNs;
    public RateLimitFilter(long perSecond) { this.perSecond = perSecond; this.tokens = new AtomicLong(perSecond); }
    @Override
    public boolean accept(LogEvent e) {
        // Token bucket — refill on each call
        long now = System.nanoTime();
        long elapsed = now - lastRefillNs;
        if (elapsed > 1_000_000_000) {
            tokens.set(perSecond);
            lastRefillNs = now;
        }
        return tokens.decrementAndGet() >= 0;
    }
}
```

### Wiring it up

```java
Logger root = LoggerFactory.getLogger("com.example");
root.setLevel(Level.INFO);

Appender file = new FileAppender(Path.of("app.log"), new JsonFormatter(),
    List.of(new RegexFilter("password=", true)));    // redact
Appender console = new ConsoleAppender(new TextFormatter(),
    List.of(new LevelFilter(Level.WARN)));            // console gets warn+

root.addAppender(new AsyncAppender(file, 10_000));
root.addAppender(console);

// Per-request:
MDC.put("traceId", "abc-123");
try {
    root.info("processing order " + orderId);
    // ...
} finally {
    MDC.clear();
}

// At app shutdown:
LoggerFactory.shutdown();
```

---

## Concurrency strategy

### What's contested
- The logger registry (creation of new loggers).
- Each appender's destination (file write, network send).

### Strategy
1. **`ConcurrentHashMap` for the registry** — `computeIfAbsent` ensures one logger per name.
2. **`CopyOnWriteArrayList` for appenders** — read-mostly; copying on add/remove is fine since config changes are rare.
3. **Per-appender synchronization** — `FileAppender` synchronizes its `write` to keep file lines intact. `AsyncAppender` uses a `BlockingQueue` (already thread-safe) and a single worker — no lock contention on the hot path.
4. **`volatile` on `level`** — readers see updates immediately; no lock needed for the level check.
5. **Hot-path is lock-free** — `isEnabled` is a `volatile` read + comparison; appenders iteration is over an immutable snapshot.

### Async semantics
- `AsyncAppender` enqueues; the worker thread dequeues and calls the wrapped appender.
- On full queue: drop with internal warning. Better than blocking (logging shouldn't backpressure your application).
- On shutdown: drain the queue before exiting.

### Caller-thread cost
Even synchronously, the cost should be: a `volatile` read + (if enabled) one allocation (the `LogEvent`) + serialization on the appender. For an async appender, the allocation + queue offer is ~200ns. Disabled-level cost is ~5ns.

---

## Edge cases / gotchas

| Case | Handling |
|---|---|
| **Expensive message argument when level is disabled** | Use `Supplier<String>` / lambda overload: `log.debug(() -> "expensive: " + compute())`. |
| **Recursion: appender logs while logging** | Detect via thread-local flag; suppress nested calls. |
| **Appender throws** | Catch in `Logger.log` — never propagate to caller. Internally report. |
| **Async queue full** | Drop with internal warning; track drop count in metrics. |
| **MDC leaks across thread-pool boundaries** | Pair every `put` with `clear` in a finally. Wrap thread pools to copy MDC. |
| **File handle limit** | Reuse appenders; close cleanly on shutdown. |
| **Process crash mid-async-write** | Trade-off: async logging may lose the in-flight queue. For audit-critical logs, use sync. |
| **Time backwards (NTP correction)** | `Instant.now()` may go backward; sort/index logs by ingestion time, not log time, if strict ordering matters. |
| **High cardinality MDC** | Don't put per-request unique values into log indices that aren't expecting them. |
| **Sensitive data leaking** | `RegexFilter` to redact patterns (password=, ssn=, token=). |

---

## Extensions / interview follow-ups

### "Logger hierarchy with inherited config (com.foo inherits from com)."
- Compute parent name on `getLogger`: `com.foo.Bar` → parent `com.foo` → `com` → root.
- A logger's effective level is its own if set, else inherited.
- Appenders can be additive (sum from all ancestors) or overriding.
- Look at SLF4J / Logback architecture for reference.

### "File rotation."
- `RollingFileAppender`: on each write, check if size > threshold or date != today.
- On rotation: close old, rename to `app.2026-05-07.log` or `app.1.log`, open new.
- Beware of races during rollover — synchronize the rollover decision.

### "Log shipping to ELK / Datadog."
- Add `HttpAppender` that buffers events and POSTs in batches.
- Use `AsyncAppender` to decouple write latency from caller.
- Retry on failure with exponential backoff; persist to disk if cloud is down.

### "Structured logging (key-value pairs)."
- Replace `String message` with `String message + Map<String, Object> fields`.
- `JsonFormatter` outputs the fields directly.
- Saves cost on parsing logs downstream.

### "Sampling — log only 1% of debug events."
- Add `SamplingFilter` with a configurable rate.
- Useful for high-volume systems.

### "Why `CopyOnWriteArrayList` for appenders?"
- Reads (every log call) are wait-free.
- Writes (config changes) are rare; the array copy cost amortizes over many reads.
- Compare to `ArrayList` + `synchronized`: the lock blocks every read.

### "Why singleton `LoggerFactory`?"
- Loggers should be process-global; multiple factories would create duplicate loggers.
- The singleton is a registry (no business logic), so the usual singleton concerns are minimal.
- I'd prefer DI in a real codebase — pass a `LoggerFactory` instance — but the static API is the SLF4J convention and what teams expect.

### "How would you test it?"
- `Appender` is an interface; `TestAppender` collects events into a list.
- `Clock` injection so timestamps are deterministic.
- MDC tested by setting context, logging, asserting the captured event.

---

## Scaling

For a single service, the design above is fine. At scale:

### Single-node throughput
- Async appenders amortize I/O cost.
- Avoid string concatenation on the hot path; use lazy lambdas.
- For Maximum throughput: log4j2 with the Disruptor (lock-free ring buffer) achieves >10M events/sec.

### Multi-service log aggregation
- Each service's appender ships JSON to a central pipeline (Fluentd, Vector, Logstash).
- Pipeline buffers, batches, ships to backends (Elasticsearch, S3, Loki).
- Trace IDs in MDC enable cross-service correlation.

### Sampling and cost control
- Production logs are expensive at scale. Sampling, level-tuning per logger, and per-environment thresholds.
- Trace-level events: enable on demand via remote config (without restart).

### Audit / compliance logs
- Separate pipeline from application logs.
- Lossless: synchronous write to durable storage (or replicated quorum).
- Immutability: append-only with cryptographic signing.

### Failure modes
- **Disk full** — appender stops; service shouldn't crash. Async appenders with bounded queues drop on backpressure.
- **Network partition to log backend** — local fallback (write to disk, ship later).
- **Backend rejects (rate-limited)** — exponential backoff; aggregate during partition.

---

## Persistence tradeoffs

A logger's "persistence" *is* its purpose — log events have to land somewhere durable. The choice of backing store shapes the entire pipeline.

### What persistent data exists
- **Log events**: append-only, high volume, time-ordered, often searched but rarely updated.
- **Configuration**: log levels, appender config (small, occasional reads).

We focus on the events.

### Option 1 — Local files (rolling)
Each service writes to a local file; rotated by size or date.
- **Pros**: simplest. No external dependency. Resilient to network partitions.
- **Cons**: not searchable across services; have to ship somewhere for analysis.
- **When to pick**: as the *first* hop, even when shipping elsewhere — local file is your buffer if the network goes down.

### Option 2 — Time-series stores (Loki, InfluxDB, TimescaleDB)
Indexed by labels (service, level, host) with a time-series shape.
- **Pros**: efficient for time-range queries, label-based filtering. Cheap storage (Loki uses object storage for the actual log lines, only labels are indexed).
- **Cons**: full-text search is slow or unavailable depending on the engine.
- **When to pick**: ops/observability use case where you mostly query "all errors in service X over the last hour."

### Option 3 — Search engines (Elasticsearch, OpenSearch)
Full-text indexing, faceting, aggregation.
- **Pros**: powerful queries. The default for the ELK stack and similar.
- **Cons**: storage cost (full inverted index). Index management (rolling indices, retention) is operational work.
- **When to pick**: you need ad-hoc text search ("find logs containing this stack trace fragment").

### Option 4 — Wide-column stores (Cassandra, ScyllaDB, BigTable)
Append-optimized; partition by service+time.
- **Pros**: extremely high write throughput. Cheap. Durable.
- **Cons**: limited query power without a separate index. Tombstones / compaction operational concerns.
- **When to pick**: very high-volume log producers (millions of events per second per node).

### Option 5 — Object storage (S3, GCS) for archival
Hot path goes to a search engine; old logs roll off to S3 / Glacier after N days.
- **Pros**: cheapest storage. Effectively unlimited.
- **Cons**: query latency (must download and scan).
- **When to pick**: compliance retention (years of logs at low cost).

### The typical real architecture (multi-tier)
```
Service → local file (buffer) → Fluentd/Vector (shipper) → Kafka (durable buffer)
                                                          → Elasticsearch (hot, last 7 days)
                                                          → S3 + Athena (cold, last 2 years)
```
- **Local file**: survives network partitions.
- **Shipper**: parses, enriches (host, environment), batches.
- **Kafka**: durable buffer; consumers can fail without losing messages.
- **Elasticsearch**: hot search.
- **S3**: cheap archival; queryable with Athena/Presto when needed.

### Audit logs vs application logs
These should be on **different pipelines**:
- **App logs**: best-effort. Drop-on-overflow is acceptable. Async appenders OK.
- **Audit logs** (security-relevant, regulatory): lossless. Synchronous writes to durable storage. Replicated quorum. Often immutable / append-only with cryptographic chaining.

Mixing them ("just log everything to ES") is a compliance failure waiting to happen.

### Recommendation
- **Sync to local file** (fast, durable buffer).
- **Async to network shipper** (batched, doesn't block app).
- **Kafka in the middle** if your scale warrants.
- **Elasticsearch for search; S3 for archive** as the dual sink.
- **Audit pipeline separate**, with synchronous DB writes (PostgreSQL append-only table) or a dedicated audit service.

### Concurrency under persistence
- **Local file**: synchronize per-file (one writer thread, queue feeders) OR open with `O_APPEND` and let the OS atomically append (single `write()` syscalls smaller than `PIPE_BUF` are atomic on POSIX).
- **Network shipper**: bounded queue + drop policy. Never block the application thread.
- **Across pipeline**: each hop has at-least-once semantics; consumers must be idempotent (dedupe by event ID).

### What about logs about the logger itself?
Self-logging is a real problem. If `FileAppender.write` fails and you call `log.error("write failed", e)` from inside it, you get infinite recursion. Solutions:
- A separate "internal log" that goes to stderr only.
- Suppress nested calls via a thread-local flag.
- Use SLF4J's `StatusLogger`-style internal channel.

---

## Talking points for the interview

- "Hot-path-disabled cost is a `volatile` read; that's the most important performance property of a logger."
- "Async via Decorator means *any* appender can be wrapped; the wrapped appender doesn't change."
- "Filters as Chain-of-Responsibility per appender — each appender independently decides what it cares about."
- "MDC is a `ThreadLocal` because per-request context shouldn't leak across requests; the snapshot in `LogEvent` makes it immutable once emitted."
- "I'm careful not to let an appender's exception escape the logger — logging should never break the caller."
- "Drop-on-full-async-queue is correct for application logs; for audit logs, I'd use synchronous writes or a fail-fast policy."
- "In production, log4j2 with Disruptor or Logback. The exercise here is to show I understand the layering, not to outperform mature libraries."

---

## Summary

Patterns load-bearing: **Strategy** (Appender, Formatter, Filter), **Decorator** (AsyncAppender), **Chain of Responsibility** (filter list per appender), **Singleton** (LoggerFactory), **Observer** (loggers fan out events to appenders).

The data flow: `log()` → level check → build event with MDC snapshot → fan out to appenders → filters → formatter → write.

The async path is Decorator-wrapped, so it's orthogonal: any appender can be sync or async by wrapping.

This design mirrors SLF4J + Logback in its bones — that's the reference implementation interviewers expect you to converge on.
