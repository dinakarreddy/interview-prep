# Problem 13 — Rate Limiter

The algorithm + concurrency problem. Tests your understanding of token bucket vs sliding window, atomic state updates, and the distributed-vs-local rate-limiting distinction.

## Interview opener

> **Interviewer:** Design a rate limiter.

---

## Clarification phase

> **You:** Single-process or distributed?
>
> **Interviewer:** Both — let's start single-process and discuss distributed.

> **You:** Per what — per user, per IP, per API key?
>
> **Interviewer:** Per identity (let's say user ID); design should be flexible.

> **You:** Granularity — N requests per second? Per minute? Both?
>
> **Interviewer:** Both — configurable. Common patterns: 1000 req/sec, 100k req/day.

> **You:** Should I support burst allowance — let users burst above the steady rate briefly?
>
> **Interviewer:** Yes. Token bucket would be appropriate.

> **You:** What happens when limit is exceeded — return error, queue, throttle?
>
> **Interviewer:** Return error (HTTP 429 in API context). The caller decides what to do.

> **You:** Multiple algorithms (fixed window, sliding window, token bucket, leaky bucket) — implement all or one?
>
> **Interviewer:** Design for pluggable algorithms; implement at least token bucket. Discuss the others.

> **You:** Throughput requirements?
>
> **Interviewer:** Hot-path: sub-microsecond per check.

> **You:** Persistence?
>
> **Interviewer:** Discuss at end.

OK — design needs: pluggable algorithm Strategy, atomic state updates per identity, single-process correctness, distributed-mode using Redis.

---

## Refined requirements

### Functional
1. `tryAcquire(identity)` — returns true if request is allowed, false if rate-limited.
2. `tryAcquire(identity, n)` — for batch operations costing N tokens.
3. Configure: `(capacity, refillRate)` per identity (or via tier rules).
4. Multiple algorithms: token bucket, leaky bucket, fixed window, sliding window.

### Non-functional
- **Latency**: nanoseconds per check in steady state.
- **Concurrency**: many threads checking simultaneously per identity; correctness without coarse locks.
- **Correctness**: respect the rate exactly; no leaks, no over-allows.

### Out of scope (mentioned)
- Quota / monthly limits (a different problem).
- Pricing tiers (orthogonal — rate-limit configs come from elsewhere).

### Core operations
1. `boolean tryAcquire(String key)` — single token.
2. `boolean tryAcquire(String key, int n)` — n tokens.
3. `RateLimitState state(String key)` — current bucket state for telemetry.

---

## Domain model

This problem has minimal domain modeling — the "data" is just per-identity counters and timestamps. The interesting design is in the **algorithm**.

### The four classic algorithms

1. **Fixed window**: count requests per discrete window (each minute). Simple but has a 2× burst at the boundary.
2. **Sliding window log**: store a timestamp per request; trim old; count remaining. Accurate but memory-heavy.
3. **Sliding window counter**: blend last window + current window proportionally to time. Approximate but cheap.
4. **Token bucket**: tokens refill at a steady rate; each request consumes 1; bucket has a capacity (max burst). Simple, allows burst, accurate over long windows.
5. **Leaky bucket**: requests fill a queue; queue drains at a steady rate; full queue → reject. Smooths bursts.

For most APIs, **token bucket** is the right default.

### Patterns
- **Strategy** — pluggable algorithm.
- **Atomic per-key state** — `ConcurrentHashMap` of identity → atomic state.
- **Singleton** — typically one rate limiter per service instance (DI'd, not `getInstance()`).

---

## Class diagram

```
┌──────────────────────────────────┐
│   <<interface>> RateLimiter      │
├──────────────────────────────────┤
│ +tryAcquire(key) : boolean       │
│ +tryAcquire(key, n) : boolean    │
│ +state(key) : LimitState         │
└──────────────────────────────────┘
            △
   ┌────────┼─────────────┬──────────────┬──────────────┐
   │                      │              │              │
┌──────────────┐  ┌──────────────┐ ┌──────────────┐ ┌──────────────────┐
│TokenBucket   │  │LeakyBucket   │ │FixedWindow   │ │SlidingWindowCnt  │
│RateLimiter   │  │RateLimiter   │ │RateLimiter   │ │RateLimiter       │
└──────────────┘  └──────────────┘ └──────────────┘ └──────────────────┘

┌──────────────────────────────────┐
│  RateLimitConfig                 │
├──────────────────────────────────┤
│ -capacity : long                 │
│ -refillRate : double  (per sec)  │
└──────────────────────────────────┘
```

---

## Key abstractions

```java
public interface RateLimiter {
    boolean tryAcquire(String key);
    boolean tryAcquire(String key, int n);
    /** Return current state for telemetry — not for control flow. */
    LimitState state(String key);
}

public interface ConfigProvider {
    RateLimitConfig configFor(String key);
}

public record RateLimitConfig(long capacity, double refillRatePerSec) { }
public record LimitState(double availableTokens, long lastRefillNs) { }
```

`ConfigProvider` lets you have different limits per user / tier without baking them into the limiter.

---

## Core code — Token Bucket

The classic. Tokens refill at a steady rate; bucket has a max capacity (the burst); each request takes one (or N).

### Per-key bucket state (atomic)

```java
final class TokenBucketState {
    /** Available tokens, fractional to handle sub-second refills accurately. */
    final AtomicReference<State> ref;

    record State(double tokens, long lastRefillNs) { }

    TokenBucketState(double initial, long now) {
        this.ref = new AtomicReference<>(new State(initial, now));
    }
}
```

Why `AtomicReference<State>` instead of separate atomics? Because the two fields (`tokens`, `lastRefillNs`) must update together. CAS on a single record gives us atomicity for both.

### `TokenBucketRateLimiter`

```java
public final class TokenBucketRateLimiter implements RateLimiter {
    private final ConfigProvider configs;
    private final ConcurrentHashMap<String, TokenBucketState> buckets = new ConcurrentHashMap<>();
    private final Clock clock;

    public TokenBucketRateLimiter(ConfigProvider configs, Clock clock) {
        this.configs = configs;
        this.clock = clock;
    }

    @Override
    public boolean tryAcquire(String key) { return tryAcquire(key, 1); }

    @Override
    public boolean tryAcquire(String key, int n) {
        if (n <= 0) throw new IllegalArgumentException();
        RateLimitConfig cfg = configs.configFor(key);
        TokenBucketState bucket = buckets.computeIfAbsent(key,
            k -> new TokenBucketState(cfg.capacity(), clock.nanos()));

        // CAS loop: read current state, compute refill + take, attempt CAS
        while (true) {
            TokenBucketState.State curr = bucket.ref.get();
            long now = clock.nanos();
            double refilled = Math.min(
                cfg.capacity(),
                curr.tokens() + (now - curr.lastRefillNs()) * cfg.refillRatePerSec() / 1_000_000_000.0
            );
            if (refilled < n) return false;   // not enough tokens, even after refill
            TokenBucketState.State next = new TokenBucketState.State(refilled - n, now);
            if (bucket.ref.compareAndSet(curr, next)) return true;
            // CAS lost — retry
        }
    }

    @Override
    public LimitState state(String key) {
        TokenBucketState b = buckets.get(key);
        if (b == null) return new LimitState(configs.configFor(key).capacity(), clock.nanos());
        TokenBucketState.State s = b.ref.get();
        long now = clock.nanos();
        RateLimitConfig cfg = configs.configFor(key);
        double refilled = Math.min(cfg.capacity(),
            s.tokens() + (now - s.lastRefillNs()) * cfg.refillRatePerSec() / 1_000_000_000.0);
        return new LimitState(refilled, now);
    }
}
```

Key properties:
- **Lazy refill**: we don't run a background thread filling buckets. We compute refill *on demand* using elapsed time since `lastRefillNs`.
- **CAS loop**: the state transition is atomic. Two threads racing on the same key will serialize through the CAS.
- **Sub-microsecond hot path**: `ConcurrentHashMap.get`, atomic read, arithmetic, CAS. No locks.

### Leaky Bucket

Functionally similar to token bucket but with different semantics: the leak rate determines the steady output rate, regardless of arrival burstiness.

```java
public final class LeakyBucketRateLimiter implements RateLimiter {
    private final ConfigProvider configs;
    private final ConcurrentHashMap<String, AtomicReference<LeakState>> buckets = new ConcurrentHashMap<>();
    private final Clock clock;

    record LeakState(double waterLevel, long lastDrainNs) { }

    @Override
    public boolean tryAcquire(String key, int n) {
        RateLimitConfig cfg = configs.configFor(key);
        AtomicReference<LeakState> ref = buckets.computeIfAbsent(key,
            k -> new AtomicReference<>(new LeakState(0, clock.nanos())));

        while (true) {
            LeakState curr = ref.get();
            long now = clock.nanos();
            double drained = Math.max(0,
                curr.waterLevel() - (now - curr.lastDrainNs()) * cfg.refillRatePerSec() / 1_000_000_000.0);
            if (drained + n > cfg.capacity()) return false;
            LeakState next = new LeakState(drained + n, now);
            if (ref.compareAndSet(curr, next)) return true;
        }
    }

    @Override public boolean tryAcquire(String key) { return tryAcquire(key, 1); }
    @Override public LimitState state(String key) { /* similar */ return null; }
}
```

The "water level" rises with requests and drains with time; rejects when it would overflow.

### Fixed Window

```java
public final class FixedWindowRateLimiter implements RateLimiter {
    private final ConfigProvider configs;
    private final ConcurrentHashMap<String, WindowCounter> windows = new ConcurrentHashMap<>();
    private final Duration window;
    private final Clock clock;

    record WindowCounter(long windowStart, AtomicLong count) { }

    public FixedWindowRateLimiter(ConfigProvider configs, Duration window, Clock clock) {
        this.configs = configs; this.window = window; this.clock = clock;
    }

    @Override public boolean tryAcquire(String key) { return tryAcquire(key, 1); }

    @Override
    public boolean tryAcquire(String key, int n) {
        RateLimitConfig cfg = configs.configFor(key);
        long now = clock.millis();
        long currentWindow = now / window.toMillis();

        while (true) {
            WindowCounter wc = windows.computeIfAbsent(key,
                k -> new WindowCounter(currentWindow, new AtomicLong(0)));
            if (wc.windowStart() != currentWindow) {
                // Window rolled over — replace
                if (windows.replace(key, wc, new WindowCounter(currentWindow, new AtomicLong(0)))) {
                    // Successfully reset
                } else {
                    continue;   // someone else reset; retry
                }
            }
            long current = wc.count().get();
            if (current + n > cfg.capacity()) return false;
            if (wc.count().compareAndSet(current, current + n)) return true;
        }
    }

    @Override public LimitState state(String key) { /* similar */ return null; }
}
```

The fixed window is simple but has the **edge burst problem**: at the window boundary, a user could fire 2× the limit in 1 second by hitting the end of one window and the start of the next.

### Sliding Window Counter (best of both)

Approximate sliding window. Combines: counts from the current full window + a portion of the previous window.

```java
public final class SlidingWindowCounterRateLimiter implements RateLimiter {
    private final ConfigProvider configs;
    private final ConcurrentHashMap<String, AtomicReference<SlidingState>> windows = new ConcurrentHashMap<>();
    private final long windowSizeMs;
    private final Clock clock;

    record SlidingState(long windowStart, long previousCount, long currentCount) { }

    @Override
    public boolean tryAcquire(String key, int n) {
        RateLimitConfig cfg = configs.configFor(key);
        long now = clock.millis();
        long currentWindow = now / windowSizeMs;
        long elapsedInWindow = now - currentWindow * windowSizeMs;
        double previousFraction = 1.0 - (double) elapsedInWindow / windowSizeMs;

        AtomicReference<SlidingState> ref = windows.computeIfAbsent(key,
            k -> new AtomicReference<>(new SlidingState(currentWindow, 0, 0)));

        while (true) {
            SlidingState curr = ref.get();
            SlidingState working = curr;
            // Window roll
            if (working.windowStart() != currentWindow) {
                long gap = currentWindow - working.windowStart();
                long newPrevious = (gap == 1) ? working.currentCount() : 0;
                working = new SlidingState(currentWindow, newPrevious, 0);
            }
            double effectiveCount = working.currentCount() + working.previousCount() * previousFraction;
            if (effectiveCount + n > cfg.capacity()) return false;
            SlidingState next = new SlidingState(working.windowStart(),
                                                  working.previousCount(),
                                                  working.currentCount() + n);
            if (ref.compareAndSet(curr, next)) return true;
        }
    }

    @Override public boolean tryAcquire(String key) { return tryAcquire(key, 1); }
    @Override public LimitState state(String key) { /* similar */ return null; }
}
```

This is what most production rate limiters use because it's accurate enough and doesn't store per-request timestamps.

### Sliding Window Log (most accurate, most expensive)

```java
public final class SlidingWindowLogRateLimiter implements RateLimiter {
    private final ConfigProvider configs;
    private final ConcurrentHashMap<String, Deque<Long>> logs = new ConcurrentHashMap<>();
    private final long windowSizeMs;
    private final Clock clock;

    @Override
    public boolean tryAcquire(String key, int n) {
        RateLimitConfig cfg = configs.configFor(key);
        long now = clock.millis();
        long cutoff = now - windowSizeMs;
        Deque<Long> log = logs.computeIfAbsent(key, k -> new ArrayDeque<>());
        synchronized (log) {
            while (!log.isEmpty() && log.peekFirst() < cutoff) log.pollFirst();
            if (log.size() + n > cfg.capacity()) return false;
            for (int i = 0; i < n; i++) log.offerLast(now);
            return true;
        }
    }

    @Override public boolean tryAcquire(String key) { return tryAcquire(key, 1); }
    @Override public LimitState state(String key) { /* similar */ return null; }
}
```

Memory: O(N) per key where N = capacity. For high-volume keys, prohibitive.

---

## Concurrency strategy

### Single-process

- **`ConcurrentHashMap`** — per-key atomicity for state lookup.
- **`AtomicReference<State>`** with **CAS loop** — atomic compound update of multiple fields.
- **No locks on the hot path** — CAS retries on contention, which is rare for typical loads.

### Why CAS over `synchronized` per key?

- For high-throughput limiters (millions of req/sec), `synchronized` introduces context-switching cost on contention.
- CAS is non-blocking; threads making progress don't block waiting threads.
- For low-contention keys (most production keys), CAS succeeds first try — equivalent cost to a regular update.

### Why `AtomicReference<Record>` over separate atomics?

- The two fields (`tokens`, `lastRefillNs`) form a logical unit.
- Updating them with two separate CAS operations leaves a window of inconsistency.
- A single `AtomicReference<State>` makes the pair atomic.

### Distributed

- In a multi-instance service, each instance has its own `ConcurrentHashMap`. Per-user limit becomes "per-instance," and a user with 10 instances effectively has 10× the limit.
- Solutions:
  1. **Sticky routing**: route by user → one instance owns the bucket. Simple, but rebalance is hard.
  2. **Shared state in Redis**: every check round-trips to Redis. Slower (~ms) but correct.
  3. **Approximate per-instance**: each instance gets `capacity/N` tokens; sum ≈ total. Trades accuracy for latency.

---

## Edge cases / gotchas

| Case | Handling |
|---|---|
| **Negative or zero `n`** | Throw `IllegalArgumentException`. |
| **Bucket initialization** | First request seeds the bucket at full capacity. |
| **Clock skew (system time changes)** | Use `clock.nanos()` (monotonic), not `System.currentTimeMillis()`. |
| **Window-boundary burst (fixed window)** | Inherent. Use sliding window if accuracy matters. |
| **Memory growth** | `ConcurrentHashMap` grows unboundedly — add LRU eviction or per-key TTL. |
| **Configs change at runtime** | `ConfigProvider` always returns latest; existing buckets adjust on next refill. |
| **Per-key contention** | CAS loop iterates a few times on hot keys; usually negligible. |
| **`refillRate` too high causing tokens > capacity overflow** | `Math.min(capacity, ...)` clamps. |
| **Thread interrupted during retry loop** | The loop is finite per attempt; check `Thread.interrupted()` if needed. |

---

## Extensions / interview follow-ups

### "Add tier-based limits."
- `ConfigProvider` returns a config based on user tier. Already supported.

### "Add daily/monthly quotas in addition to per-second rate."
- Compose two rate limiters: a fast one (per-second) and a slow one (per-day). Both must allow.
- `CompositeRateLimiter` ANDs them.

### "Add waiting (block until tokens available) instead of just rejecting."
- New method: `acquire(key)` that blocks. Compute time until next refill; sleep; retry.
- Risk: slow callers tie up threads. Use sparingly.

### "Add graceful degradation — when limit is hit, return 'retry-after' header."
- `tryAcquire` returns a richer `Decision` with `boolean allowed` + `Duration retryAfter`.
- Compute `retryAfter = (n - currentTokens) / refillRate`.

### "Tier-bypass for emergency / admin requests."
- Inject a `Predicate<String>` that bypasses the limit for certain keys.

### "Why token bucket over fixed window?"
- Fixed window: 2× burst at boundary; coarse granularity.
- Token bucket: smooth, supports burst, no boundary issues, sub-second refill.

### "Why approximate sliding window over exact?"
- Exact requires storing per-request timestamps — O(capacity) memory per key.
- Approximate (counter-based) is O(1) per key with negligible accuracy loss.

### "What about leaky bucket vs token bucket?"
- Token bucket: caller-side; allows burst.
- Leaky bucket: queue-side; smooths output to a steady rate regardless of arrival pattern.
- For "limit the number of API calls a user makes," token bucket. For "smooth bursty traffic before forwarding upstream," leaky bucket.

### "How do you make this distributed?"
- Use Redis as the shared bucket store.
- Atomic refill + take in a Lua script (Redis evals atomically).
- See "Persistence" section.

---

## Scaling

Single-process: 100s of millions of checks/sec on commodity hardware.

### Multi-instance (one service deployed N times)

Three approaches, in increasing accuracy and decreasing latency advantage:

#### Sticky routing
- Hash user ID → instance.
- Each user always hits the same instance.
- Per-instance limiter is exact for that user.
- **Pros**: latency unchanged (in-memory).
- **Cons**: rebalancing on instance add/remove redistributes users; brief inaccuracy. Hot users land on hot instances.

#### Shared state via Redis
- Every `tryAcquire` is a Redis call.
- Atomic update via Lua script.
- **Pros**: globally accurate.
- **Cons**: ~1ms per check vs nanoseconds in-memory; Redis becomes a bottleneck.

#### Local-first with periodic reconciliation
- Each instance has a local bucket allocated `capacity / N`.
- On limit hit, instance asks Redis for "more tokens" or rebalances.
- **Pros**: low latency, mostly accurate.
- **Cons**: complex; brief inaccuracy on bursts.

### Lua script for Redis-backed token bucket

```lua
-- KEYS[1] = bucket key
-- ARGV[1] = capacity
-- ARGV[2] = refill_rate_per_sec
-- ARGV[3] = now_micros
-- ARGV[4] = n (tokens to take)
local data = redis.call("HMGET", KEYS[1], "tokens", "last_refill_us")
local tokens     = tonumber(data[1]) or tonumber(ARGV[1])
local last_us    = tonumber(data[2]) or tonumber(ARGV[3])
local capacity   = tonumber(ARGV[1])
local rate       = tonumber(ARGV[2])
local now_us     = tonumber(ARGV[3])
local n          = tonumber(ARGV[4])

local elapsed_us = now_us - last_us
local refilled   = math.min(capacity, tokens + (elapsed_us * rate / 1000000))

if refilled < n then
  return 0
end
redis.call("HMSET", KEYS[1], "tokens", refilled - n, "last_refill_us", now_us)
redis.call("EXPIRE", KEYS[1], 3600)
return 1
```

Atomic by virtue of Redis's single-threaded execution model. Per-key TTL prevents memory leaks for inactive users.

---

## Persistence tradeoffs

For rate limiting, "persistence" splits into two questions: *do we need to survive a restart?* and *do we need to share state across instances?*

### What persistent data exists
- **Configurations** (capacity, rate per user/tier): definitely persistent — DB or config service.
- **Bucket state** (current tokens, last refill): possibly persistent depending on requirements.
- **Audit / metrics**: separate concern; high-volume; usually time-series.

### Survival across restart — when does it matter?

For most APIs, **bucket state is ephemeral**: if the service restarts, the bucket starts fresh (full capacity). This is fine because:
- Rate limits are heuristics, not contracts.
- A restart-induced "reset" gives the user at most one extra burst.
- The simpler design wins.

For strict-quota cases (paid API tiers with "1M calls/month" billing), persistence matters:
- Store `monthly_used` per user; DB-backed.
- Per-second checks can stay ephemeral; the quota is the binding constraint.

### Single instance — no persistence required

In-memory `ConcurrentHashMap` is sufficient. Bucket state lost on restart; users get at most one full bucket free.

### Multi-instance — Redis or sticky routing

#### Redis (recommended)
- Bucket state in Redis; Lua script handles atomic update.
- TTL on keys (3600s) for memory hygiene.
- **Pros**: simple, accurate.
- **Cons**: per-check latency (~1ms); Redis as single point of failure.

#### Sticky routing
- Bucket state in instance memory; no DB.
- Routing layer ensures user → instance.
- **Pros**: in-memory speed.
- **Cons**: complex routing layer; rebalancing.

#### DynamoDB / similar
- Conditional updates (`UpdateItem` with condition).
- Higher latency than Redis; useful in cloud-native deployments.

#### Local-first with sync
- Per-instance bucket; periodic reconciliation via Redis.
- Hybrid: low latency, eventual accuracy.

### Configurations
- Stored in DB (PostgreSQL) or config service (Consul, etcd).
- Cached in-memory with periodic refresh.
- Rare updates → high cache hit rate.

### Audit / metrics
- Each `tryAcquire` outcome (allowed/denied) emitted as a metric.
- Time-series store (Prometheus / Datadog) for monitoring.
- Sample if volume is too high.

### Distributed quota (monthly limits)
- Stored in DB with version field.
- Updated transactionally per call.
- Cached in Redis with periodic write-back.
- Risk: brief over-allowance if cache is stale; usually acceptable.

### Recommendation

| Scenario | Recommendation |
|---|---|
| Single-instance API | In-memory token bucket. No persistence. |
| Multi-instance API, soft limits | In-memory + sticky routing, OR Redis token bucket. |
| Multi-instance, strict billing quotas | Redis for hot path + DB for billable counter. |
| Edge / CDN | Per-edge local; soft enforcement; anti-DDoS at gateway. |

### Concurrency under persistence
| In-memory | Redis | DynamoDB |
|---|---|---|
| `AtomicReference` + CAS | Lua script (atomic) | Conditional `UpdateItem` |
| Per-key | Per-key | Per-key |
| Nanoseconds | ~1ms | ~5ms |

---

## Talking points for the interview

- "Token bucket because it's accurate, supports burst, and uses O(1) memory per key."
- "CAS with `AtomicReference<State>` for atomic compound updates — both `tokens` and `lastRefill` move together."
- "Lazy refill: we don't run a background thread; we compute the refill on demand from elapsed time."
- "For distributed mode: Redis with a Lua script. Atomicity comes for free from Redis's single-threaded execution."
- "Memory hygiene: per-key TTL evicts inactive users. Without it, the map grows unboundedly."
- "Sliding window counter is the production sweet spot — accuracy of sliding window with O(1) memory."
- "I'd use `clock.nanos()` (monotonic), not `currentTimeMillis()` — system clock changes shouldn't break the limiter."
- "For strict billing quotas, layer a per-day/per-month limiter on top of the per-second one. Both must allow."

---

## Summary

Patterns load-bearing: **Strategy** (algorithm choice), **atomic CAS** (per-key state).

The mental model: per-identity bucket; check + decrement atomically; refill on-demand from elapsed time. The CAS loop is the core primitive — non-blocking, lock-free, tens-of-nanoseconds per check.

The senior signal here is recognizing the algorithm trade-offs (token bucket vs sliding window vs fixed window) and the distributed-mode complications (sticky routing vs shared state vs hybrid). Naive answers reach for `synchronized` or run a background refill thread; senior answers do CAS loop + lazy refill.
