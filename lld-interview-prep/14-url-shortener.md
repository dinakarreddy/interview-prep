# Problem 14 — URL Shortener (TinyURL / bit.ly)

The ID-generation problem. Tests whether you can pick the right slug-generation algorithm, handle collisions and idempotency, and reason about the read-heavy nature of the workload.

## Interview opener

> **Interviewer:** Design a URL shortener.

---

## Clarification phase

> **You:** What's in scope — shortening, redirecting, analytics?
>
> **Interviewer:** Shorten + redirect for the core. Analytics as a discussion point.

> **You:** Custom slugs (vanity URLs)?
>
> **Interviewer:** Yes — users can request a specific slug if available.

> **You:** Should the same long URL always shorten to the same slug?
>
> **Interviewer:** Configurable. Some services return the same slug; others issue new ones. Design for both.

> **You:** Slug character set and length?
>
> **Interviewer:** Base62 (alphanumeric), 7 characters by default — enough for ~3.5 trillion URLs.

> **You:** Expiration / TTL on URLs?
>
> **Interviewer:** Yes — optional per URL.

> **You:** User accounts / ownership?
>
> **Interviewer:** Yes — authenticated users see their own URLs; anonymous shortening also allowed.

> **You:** Read vs write ratio?
>
> **Interviewer:** Heavily read-skewed. ~100:1.

> **You:** Latency targets?
>
> **Interviewer:** Redirect should be sub-50ms p99.

> **You:** Throughput?
>
> **Interviewer:** Tens of thousands of redirects per second.

> **You:** Persistence?
>
> **Interviewer:** Discuss at the end.

OK — design needs: slug generation Strategy, atomic counter or hash-based, collision handling, custom slugs, TTL, fast redirect path, owner attribution.

---

## Refined requirements

### Functional
1. `shorten(longUrl, customSlug?, ttl?, ownerId?)` → `slug`.
2. `resolve(slug)` → `longUrl` or 404.
3. `delete(slug, ownerId)` → owner-only delete.
4. `analytics(slug)` → click count, last access time, etc.

### Non-functional
- **Read latency**: redirect under 50ms p99.
- **Throughput**: tens of thousands of reads/sec; thousands of writes/sec.
- **Slug uniqueness**: globally.
- **Read-skewed workload**: ~100:1 reads vs writes.

### Out of scope (mentioned)
- Detailed analytics dashboards.
- Anti-spam / phishing detection.
- QR code generation.

### Core operations
1. `String shorten(String longUrl, String customSlug, Duration ttl, String ownerId)`
2. `Optional<String> resolve(String slug)` — returns long URL or empty for 404 / expired.
3. `boolean delete(String slug, String ownerId)`

---

## Domain model

### Entities (have ID)
- **`UrlMapping`** — slug → long URL + metadata.

### Value objects (immutable)
- **`Url(longUrl)`** — could be a wrapper for validation.
- **`AnalyticsSnapshot(clickCount, lastAccessed, ...)`**.

### Enums
- `SlugSource` — `GENERATED`, `CUSTOM`.

### Patterns
- **Strategy** — slug generation algorithm.
- **Factory** — pick the right generator.
- **Atomic counter** — for counter-based generators.
- **Cache-aside** — Redis for hot slugs, DB as source of truth.
- **Repository** — persistence abstraction.

---

## Class diagram

```
┌──────────────────────────────────┐
│   <<interface>>                  │
│   SlugGenerator                  │
├──────────────────────────────────┤
│ +generate(longUrl) : String      │
└──────────────────────────────────┘
            △
   ┌────────┼─────────────┬──────────────┐
   │                      │              │
┌──────────────────┐ ┌──────────────────┐ ┌──────────────────┐
│ Base62Counter    │ │ HashSlugGen      │ │ RandomSlugGen    │
│ SlugGenerator    │ │ (md5/sha256)     │ │ (random base62)  │
└──────────────────┘ └──────────────────┘ └──────────────────┘

┌─────────────────────────────────────┐
│         UrlShortenerService         │
├─────────────────────────────────────┤
│ -mappingRepo : UrlMappingRepository │
│ -slugGen : SlugGenerator            │
│ -cache : Cache<String, String>      │
│ -clock : Clock                      │
├─────────────────────────────────────┤
│ +shorten(longUrl, ...) : String     │
│ +resolve(slug) : Optional<String>   │
│ +delete(slug, ownerId)              │
└─────────────────────────────────────┘

┌──────────────────────────────────┐
│         UrlMapping               │
├──────────────────────────────────┤
│ -slug : String                   │
│ -longUrl : String                │
│ -ownerId : String (nullable)     │
│ -createdAt : Instant             │
│ -expiresAt : Instant (nullable)  │
│ -clickCount : long               │
│ -source : SlugSource             │
└──────────────────────────────────┘

┌──────────────────────────────────────────┐
│    <<interface>> UrlMappingRepository    │
├──────────────────────────────────────────┤
│ +findBySlug(slug) : Optional<UrlMapping> │
│ +save(mapping) : void                    │
│ +saveIfAbsent(mapping) : boolean         │
│ +incrementClicks(slug) : void            │
│ +findByLongUrl(longUrl) : Optional<...>  │
└──────────────────────────────────────────┘
```

---

## Key abstractions

```java
public interface SlugGenerator {
    /**
     * Generate a slug. May depend on the long URL (hash-based) or be independent (counter/random).
     * The caller handles collision retry.
     */
    String generate(String longUrl);
}

public interface UrlMappingRepository {
    Optional<UrlMapping> findBySlug(String slug);
    /** Atomic insert-if-absent. Returns true if inserted, false if slug existed. */
    boolean saveIfAbsent(UrlMapping mapping);
    void save(UrlMapping mapping);
    void incrementClicks(String slug);
    Optional<UrlMapping> findByLongUrl(String longUrl);
    void delete(String slug);
}
```

---

## Core code

### Value objects + enums

```java
public enum SlugSource { GENERATED, CUSTOM }

public final class UrlMapping {
    private final String slug;
    private final String longUrl;
    private final String ownerId;          // nullable
    private final Instant createdAt;
    private final Instant expiresAt;       // nullable
    private long clickCount;
    private final SlugSource source;

    public UrlMapping(String slug, String longUrl, String ownerId,
                      Instant createdAt, Instant expiresAt, SlugSource source) {
        this.slug = slug; this.longUrl = longUrl; this.ownerId = ownerId;
        this.createdAt = createdAt; this.expiresAt = expiresAt;
        this.source = source;
    }

    public String slug()                   { return slug; }
    public String longUrl()                { return longUrl; }
    public String ownerId()                { return ownerId; }
    public Instant createdAt()             { return createdAt; }
    public Instant expiresAt()             { return expiresAt; }
    public long clickCount()               { return clickCount; }
    public void incrementClicks()          { clickCount++; }
    public SlugSource source()             { return source; }
    public boolean isExpired(Instant now)  { return expiresAt != null && now.isAfter(expiresAt); }
}
```

### Slug generators (Strategy)

#### Counter-based (Base62)
The cleanest approach. A central atomic counter; convert to Base62.

```java
public final class Base62CounterSlugGenerator implements SlugGenerator {
    private static final String ALPHABET = "0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz";
    private final AtomicLong counter;

    public Base62CounterSlugGenerator(long startValue) {
        this.counter = new AtomicLong(startValue);
    }

    @Override
    public String generate(String longUrl) {
        long n = counter.getAndIncrement();
        return encodeBase62(n);
    }

    private String encodeBase62(long n) {
        if (n == 0) return String.valueOf(ALPHABET.charAt(0));
        StringBuilder sb = new StringBuilder();
        while (n > 0) {
            sb.append(ALPHABET.charAt((int) (n % 62)));
            n /= 62;
        }
        return sb.reverse().toString();
    }
}
```

**Pros**: no collisions ever. Sequential and compact. Easy to back with a DB sequence in distributed deployments.
**Cons**: predictable (someone can enumerate URLs by guessing). For privacy-sensitive cases, mask with a small hash or randomize the bit order.

#### Hash-based
Useful when "same long URL = same slug" is desired.

```java
public final class HashSlugGenerator implements SlugGenerator {
    private final int slugLength;

    public HashSlugGenerator(int slugLength) { this.slugLength = slugLength; }

    @Override
    public String generate(String longUrl) {
        try {
            MessageDigest md = MessageDigest.getInstance("SHA-256");
            byte[] hash = md.digest(longUrl.getBytes(StandardCharsets.UTF_8));
            // Take leading bytes and convert to Base62
            BigInteger bi = new BigInteger(1, hash);
            return toBase62(bi).substring(0, slugLength);
        } catch (NoSuchAlgorithmException e) {
            throw new RuntimeException(e);
        }
    }

    private String toBase62(BigInteger n) {
        // Implementation similar to Base62 encode but for BigInteger
        StringBuilder sb = new StringBuilder();
        BigInteger sixtyTwo = BigInteger.valueOf(62);
        while (n.signum() > 0) {
            sb.append("0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz".charAt(n.mod(sixtyTwo).intValue()));
            n = n.divide(sixtyTwo);
        }
        return sb.reverse().toString();
    }
}
```

**Pros**: deterministic — useful for de-duplicating identical URLs.
**Cons**: collisions possible (two different URLs → same slug). Service must detect and disambiguate (rehash with a salt).

#### Random
For maximum unpredictability; collisions handled by retry.

```java
public final class RandomSlugGenerator implements SlugGenerator {
    private static final String ALPHABET = "0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz";
    private final int slugLength;
    private final Random rng;

    public RandomSlugGenerator(int slugLength) {
        this.slugLength = slugLength;
        this.rng = ThreadLocalRandom.current();
    }

    @Override
    public String generate(String longUrl) {
        char[] buf = new char[slugLength];
        for (int i = 0; i < slugLength; i++) buf[i] = ALPHABET.charAt(rng.nextInt(62));
        return new String(buf);
    }
}
```

**Pros**: unpredictable; doesn't reveal information.
**Cons**: collisions possible at scale; need atomic insert-if-absent and retry.

### `UrlShortenerService`

```java
public final class UrlShortenerService {
    private final UrlMappingRepository repo;
    private final SlugGenerator slugGen;
    private final Cache<String, String> hotCache;        // slug → longUrl
    private final Clock clock;
    private final UrlValidator validator;
    private final boolean dedupeIdenticalUrls;

    public UrlShortenerService(UrlMappingRepository repo, SlugGenerator slugGen,
                               Cache<String, String> cache, Clock clock,
                               UrlValidator validator, boolean dedupeIdenticalUrls) {
        this.repo = repo; this.slugGen = slugGen;
        this.hotCache = cache; this.clock = clock;
        this.validator = validator; this.dedupeIdenticalUrls = dedupeIdenticalUrls;
    }

    public String shorten(String longUrl, String customSlug, Duration ttl, String ownerId) {
        validator.validate(longUrl);

        // Custom slug path
        if (customSlug != null) {
            UrlMapping mapping = buildMapping(customSlug, longUrl, ownerId, ttl, SlugSource.CUSTOM);
            if (!repo.saveIfAbsent(mapping)) {
                throw new SlugTakenException(customSlug);
            }
            return customSlug;
        }

        // Dedupe identical URLs (optional)
        if (dedupeIdenticalUrls) {
            Optional<UrlMapping> existing = repo.findByLongUrl(longUrl);
            if (existing.isPresent() && !existing.get().isExpired(clock.instant())) {
                return existing.get().slug();
            }
        }

        // Generate with retry on collision
        for (int attempt = 0; attempt < 5; attempt++) {
            String slug = slugGen.generate(longUrl);
            UrlMapping mapping = buildMapping(slug, longUrl, ownerId, ttl, SlugSource.GENERATED);
            if (repo.saveIfAbsent(mapping)) return slug;
            // Collision — retry. For hash-based generators, salt to vary.
        }
        throw new RuntimeException("Failed to generate unique slug after retries");
    }

    public Optional<String> resolve(String slug) {
        // Hot path: check cache first
        String cached = hotCache.getIfPresent(slug);
        if (cached != null) {
            asyncIncrementClicks(slug);
            return Optional.of(cached);
        }
        // Miss: load from DB
        Optional<UrlMapping> mapping = repo.findBySlug(slug);
        if (mapping.isEmpty()) return Optional.empty();
        UrlMapping m = mapping.get();
        if (m.isExpired(clock.instant())) return Optional.empty();
        hotCache.put(slug, m.longUrl());
        asyncIncrementClicks(slug);
        return Optional.of(m.longUrl());
    }

    public boolean delete(String slug, String ownerId) {
        UrlMapping mapping = repo.findBySlug(slug).orElseThrow(() -> new NotFoundException(slug));
        if (mapping.ownerId() == null || !mapping.ownerId().equals(ownerId))
            throw new ForbiddenException();
        repo.delete(slug);
        hotCache.invalidate(slug);
        return true;
    }

    private void asyncIncrementClicks(String slug) {
        // Fire-and-forget; eventual consistency on click count is fine.
        ForkJoinPool.commonPool().submit(() -> {
            try { repo.incrementClicks(slug); } catch (Exception ignored) {}
        });
    }

    private UrlMapping buildMapping(String slug, String longUrl, String ownerId,
                                    Duration ttl, SlugSource source) {
        Instant now = clock.instant();
        Instant expiresAt = ttl == null ? null : now.plus(ttl);
        return new UrlMapping(slug, longUrl, ownerId, now, expiresAt, source);
    }
}
```

### URL validator

```java
public final class UrlValidator {
    public void validate(String url) {
        if (url == null || url.isBlank()) throw new IllegalArgumentException();
        if (url.length() > 2048) throw new IllegalArgumentException("URL too long");
        try {
            URI uri = new URI(url);
            if (uri.getScheme() == null || (!uri.getScheme().equals("http") && !uri.getScheme().equals("https")))
                throw new IllegalArgumentException("Only http/https allowed");
            if (uri.getHost() == null) throw new IllegalArgumentException("No host");
        } catch (URISyntaxException e) {
            throw new IllegalArgumentException("Invalid URL", e);
        }
    }
}
```

---

## Concurrency strategy

### What's contested
- Slug counter (in counter-based generator).
- The mapping table (concurrent inserts).
- Click counter (every redirect bumps it).

### Strategy
- **`AtomicLong` for the counter** — lock-free.
- **`saveIfAbsent`** for atomic slug claiming. In SQL, this is `INSERT ... ON CONFLICT DO NOTHING` (PostgreSQL) or `INSERT IGNORE` (MySQL). Returns true if inserted, false if slug existed.
- **Async click counter increments** — fire-and-forget; eventual consistency is fine (a few lost clicks under failure are acceptable).
- **Hot cache** for read path — cache-aside pattern with Caffeine or Redis. Most slugs are read once and never read again; popular ones are read a million times.

### Distributed counter
For multi-instance deployments, a shared `AtomicLong` doesn't work. Options:
- **DB sequence** (PostgreSQL `BIGSERIAL`) — central; ~1ms per call.
- **Pre-allocated ranges**: each instance fetches a block of 1000 IDs from DB; allocates locally; refills when low. ~1k× throughput improvement.
- **Snowflake-style IDs**: 64-bit, time-based + machine-id — generated locally without coordination. Slightly larger slugs.
- **UUID v7**: time-ordered, no coordination, longer.

For most LLD interview answers: **range-based pre-allocation** is the right answer.

---

## Edge cases / gotchas

| Case | Handling |
|---|---|
| **Custom slug already taken** | `saveIfAbsent` returns false → throw clear error. |
| **Hash-based collision (different URLs, same slug)** | Detect via `saveIfAbsent`; retry with salt. |
| **Same URL shortened twice** | Configurable: dedupe → return existing; otherwise generate new. |
| **Empty / invalid URL** | Validate at entry; reject. |
| **URL with sensitive query params** | Out of scope; production systems strip tracking params. |
| **Phishing / malicious URLs** | Out of LLD scope; production has a separate scanner. |
| **Expired URL** | `resolve` returns empty; redirect → 410 Gone. |
| **Click counter under high read** | Async increment; can lose a few on crash; acceptable. Strict counts use a write-behind queue. |
| **Cache stampede on hot miss** | `Cache.getIfPresent + computeIfAbsent` consolidates concurrent loads. |
| **Slug case sensitivity** | Base62 uses both cases; treat slug lookups as case-sensitive. |
| **Reserved slugs** ("admin", "login") | Maintain a blocklist; reject custom requests for them. |
| **TTL precision** | Per-second is fine; don't worry about ms. |

---

## Extensions / interview follow-ups

### "Add detailed analytics."
- Each redirect emits a click event (timestamp, IP/country, user-agent, referer) to a queue.
- Async pipeline aggregates into per-slug daily/weekly counters.
- Time-series DB or warehouse for the events; Redis for hot per-slug counters.

### "Why not just `nanoTime()` % 62^7 as the slug?"
- Time isn't unique under concurrent calls.
- Modulo collisions over time.
- Counter or hash is the right primitive.

### "Why Base62 not Base64?"
- Base64 includes `+` and `/` (URL-unsafe in path) and `=` (padding).
- Base62 is alphanumeric only — URL-safe by default.
- Some shorteners use Base58 (Bitcoin's choice) to drop visually-confusable characters (`0`, `O`, `I`, `l`).

### "Add a 'preview' page (don't redirect immediately)."
- New endpoint `/preview/{slug}` returns the long URL + metadata.
- Useful for security: users see where they're going before clicking.

### "How do you handle a single very-popular URL (one slug, billions of redirects)?"
- The cache makes single-key reads dominant — local in-process cache hits first.
- For viral spikes: CDN edge caching of redirects (HTTP 301/302 responses are cacheable).

### "How do you scale writes?"
- Range-based pre-allocation (each instance owns 1000 IDs at a time).
- Sharded counter tables (per-region or per-prefix).

### "How do you scale reads?"
- Heavy caching (Redis or in-memory).
- Read replicas of the DB.
- CDN for the redirect responses.

### "What if the cache is cold and a popular slug gets a million reads?"
- Use single-flight: `cache.computeIfAbsent` lets only one thread fetch from DB; others wait for that result.
- Caffeine and Guava both support this idiom.

### "How do you delete millions of expired URLs?"
- DB index on `expires_at`; sweeper runs nightly.
- Or DB-native TTL (DynamoDB, Redis, Cassandra TTL).

### "What about anti-phishing?"
- Out of LLD scope. Real systems integrate with Google Safe Browsing or a similar URL reputation service before shortening.

---

## Scaling

The read path is the dominant concern at scale.

### Read path optimization
- **Multi-tier cache**:
  - L1: in-process (Caffeine), ~ns per read.
  - L2: Redis cluster, ~ms per read.
  - L3: DB, ~10ms per read.
  - L4: CDN edge cache for HTTP responses (301/302), if redirect is the only purpose.
- 99% of reads should hit L1 or L2.

### Write path optimization
- Rare relative to reads.
- Range-allocated counters scale to many instances.
- Custom slugs use `INSERT ... ON CONFLICT` for atomicity.

### Geographic distribution
- DB primary in one region; read replicas in many.
- Slugs are immutable post-creation, so replication lag doesn't matter for reads (only for "I just shortened, why doesn't it resolve?" — typically <1 second).
- For ultra-low-latency reads worldwide: replicate the slug → URL map to edge KV stores (Cloudflare KV, Workers, Lambda@Edge).

### Click counts at scale
- Async via Kafka or similar:
  - Redirect handler emits click event.
  - Batched aggregator writes to a per-day counter.
  - Real-time dashboard reads from the counter store.
- Lose a few on crash; acceptable for click counters.

---

## Persistence tradeoffs

This is a read-heavy, simple-schema, mostly-immutable workload — perfect for caches.

### What persistent data exists
- **Mappings**: slug → longUrl + metadata. Source of truth.
- **Click events**: append-only stream.
- **Counters / aggregates**: derived; cached.

### Option 1 — Relational (PostgreSQL / MySQL)
```sql
url_mappings(slug PRIMARY KEY, long_url, owner_id, created_at, expires_at, click_count, source)
INDEX (long_url)         -- for dedupe lookup
INDEX (owner_id)         -- for "my URLs" listing
INDEX (expires_at)       -- for sweeper
```
- **Pros**: ACID; secondary indexes; familiar.
- **Cons**: high read load needs replicas + caching. Click count update is a hot row.
- **When to pick**: starting point for any deployment. Scales to millions of URLs comfortably.

### Option 2 — Key-value (DynamoDB / Cassandra)
- Partition key = `slug`. Point lookup is the dominant op — perfect.
- Click count updated via atomic increment.
- **Pros**: horizontal scale; predictable read latency; per-key TTL native.
- **Cons**: secondary indexes (find by long_url, list by owner) cost extra. Range queries hard.
- **When to pick**: very high-scale deployments where partition-key reads dominate.

### Option 3 — Redis as primary store
- Slugs as Redis keys; values are JSON or HASH.
- TTL native.
- **Pros**: extremely fast reads; TTL handled by Redis.
- **Cons**: durability — Redis with persistence enabled; or pair with a DB for permanence.
- **When to pick**: when the entire dataset fits in memory and persistence is OK at Redis-AOF level.

### Option 4 — Hybrid (recommended for production)
- **DynamoDB or PostgreSQL** as source of truth.
- **Redis** as hot read cache with TTL.
- **Kafka** for click event stream.
- **Time-series DB** (or warehouse) for aggregated analytics.

### Click counter strategies

#### Inline increment
```sql
UPDATE url_mappings SET click_count = click_count + 1 WHERE slug = ?
```
- Simple but creates a hot row; concurrent updates serialize.
- OK for low-traffic slugs; bottleneck for viral ones.

#### Async write-behind
- Click event → Kafka.
- Aggregator consumes; periodically updates DB.
- Better throughput; eventual consistency.

#### Per-day shards
- Counter rows: `(slug, date)`. Updates spread across many rows.
- Total = sum across dates.
- Avoids hot-row entirely.

### Concurrency under persistence
- **`saveIfAbsent`**: SQL `INSERT ... ON CONFLICT DO NOTHING`; DynamoDB conditional `PutItem`.
- **Counter increment**: SQL `UPDATE ... SET click_count = click_count + 1`; DynamoDB `UpdateItem` with `ADD click_count :n`.
- Both are atomic at the DB level.

### Recommendation
- **Default**: PostgreSQL primary + Redis cache. Async click counts via Kafka.
- **At scale**: DynamoDB primary + Redis hot cache + click-count sharding.
- **Edge-deployed**: Cloudflare Workers KV for the read cache, deployed to every edge.

---

## Talking points for the interview

- "Counter-based slug generation has zero collisions — the counter is the unique identifier. Hashing is for de-dup, not for generation."
- "For distributed counters, range pre-allocation: each instance grabs 1000 IDs at a time, generates locally, refills when low."
- "Read path is dominant — 100:1. Cache aggressively. CDN-cache the redirects if possible."
- "Click counter is async; we accept losing a handful on crash. Strict counts use Kafka + aggregator."
- "Custom slugs use atomic insert-if-absent. Generated slugs use the same primitive — handles collisions automatically."
- "Slug lookups are pure key-value — DynamoDB shines here. PostgreSQL also works at moderate scale."
- "Reserved slugs (admin, login, api) need a blocklist to prevent abuse."

---

## Summary

Patterns load-bearing: **Strategy** (slug generation), **atomic insert-if-absent** (for collision handling), **cache-aside** (for read scaling).

The mental model: a cheap-to-look-up key-value store, with the only interesting work being **slug generation**. The choice of generator (counter, hash, random) drives everything else: counter-based has no collisions, hash-based dedupes identical URLs, random is unpredictable. Once a slug is allocated, the mapping is essentially immutable; reads are dominant; cache aggressively.

The senior signal here is recognizing the workload shape — read-heavy, immutable, simple key-value — and reaching for the right primitives: atomic counter + cache + async click counter.
