# Problem 3 — LRU Cache

The data-structure-fluency problem. Interviewers test whether you can compose a doubly-linked list with a HashMap, and whether you can reason about thread safety on top.

## Interview opener

> **Interviewer:** Design an LRU cache.

---

## Clarification phase

> **You:** Quick scoping. The cache has a fixed capacity, and when full, evicts the least recently used entry?
>
> **Interviewer:** Right.

> **You:** What does "use" mean — both `get` and `put` count?
>
> **Interviewer:** Yes — both reads and writes mark an entry as "most recently used."

> **You:** What's the expected complexity? O(1) for both `get` and `put`?
>
> **Interviewer:** Yes.

> **You:** Generic key-value, or specific types?
>
> **Interviewer:** Generic — `<K, V>`.

> **You:** Should `get(missingKey)` return null, throw, or return Optional?
>
> **Interviewer:** Return null is fine for this — match the standard library convention.

> **You:** Concurrent access — multiple threads hitting it?
>
> **Interviewer:** Yes; design should be thread-safe. Single-process is fine.

> **You:** TTL on entries, or pure size-based eviction?
>
> **Interviewer:** Just size-based for the core. Mention how you'd add TTL.

> **You:** Should I support pluggable eviction policies, or hardcode LRU?
>
> **Interviewer:** Just LRU for now. Mention how you'd extend.

> **You:** Eviction notifications? E.g., callback when an entry is evicted (write-behind cache to DB)?
>
> **Interviewer:** Mention as an extension; not core.

OK — the core is a thread-safe O(1) get/put cache with size-based LRU eviction.

---

## Refined requirements

### Functional
1. `V get(K key)` — return value if present, mark as MRU; return null on miss.
2. `void put(K key, V value)` — insert or update; evict LRU if at capacity.
3. (Bonus) `V remove(K key)`, `int size()`, `void clear()`.

### Non-functional
- O(1) `get` and `put` (amortized).
- Thread-safe.
- Bounded memory: at most `capacity` entries.

### Out of scope (mentioned as extensions)
- TTL.
- Multiple eviction policies (LFU, FIFO).
- Distributed caching.
- Eviction listeners.
- Cache statistics (hit rate).

---

## Domain model

This problem is unusual — the "domain" is just the data structure. There are no entities or value objects in the DDD sense. The artifacts are:

- A **doubly-linked list** of entries, with the most recently used at the head and the least recently used at the tail.
- A **HashMap** from key → list node, for O(1) lookup.

Both structures point at the same nodes, which is what gives the design its O(1) properties:
- `get(key)`: HashMap finds the node in O(1); the node is unlinked from its current position and re-linked at the head, both O(1) because we have direct pointers.
- `put(key, value)` on existing key: same as get plus update.
- `put(key, value)` on new key: create node, insert at head, add to map; if size exceeds capacity, drop the tail node and remove its key from the map.

---

## Class diagram

```
┌──────────────────────────────────┐
│         LRUCache<K, V>           │
├──────────────────────────────────┤
│ -capacity : int                  │
│ -map : Map<K, Node<K,V>>         │
│ -head, -tail : Node<K,V>         │  (sentinels)
│ -lock : ReentrantLock            │
├──────────────────────────────────┤
│ +get(K) : V                      │
│ +put(K, V) : void                │
│ +remove(K) : V                   │
│ +size() : int                    │
│ +clear() : void                  │
│ -moveToHead(Node)                │
│ -addToHead(Node)                 │
│ -removeNode(Node)                │
│ -evictLRU() : Node<K,V>          │
└──────────────────────────────────┘
            │ ◇ holds
            ▼
┌──────────────────────────────────┐
│         Node<K, V>               │
├──────────────────────────────────┤
│ -key : K                         │
│ -value : V                       │
│ -prev, -next : Node<K,V>         │
└──────────────────────────────────┘
```

Patterns:
- The doubly-linked list with head/tail sentinels is a textbook idiom for clean edge-case handling.
- For pluggable eviction (extension): introduce `EvictionPolicy` Strategy.

---

## Key abstractions

```java
public interface Cache<K, V> {
    V get(K key);
    void put(K key, V value);
    V remove(K key);
    int size();
    void clear();
}

// Optional Strategy for swappable policies (extension)
public interface EvictionPolicy<K, V> {
    void recordAccess(Node<K, V> n);
    void recordInsertion(Node<K, V> n);
    Node<K, V> selectVictim();
}
```

---

## Core code

### `Node`

```java
final class Node<K, V> {
    final K key;
    V value;
    Node<K, V> prev, next;

    Node(K k, V v) { this.key = k; this.value = v; }
}
```

The key is `final` (it's the identity in the map; never changes). The value is mutable so `put` on existing key can update in place.

### `LRUCache`

```java
public final class LRUCache<K, V> implements Cache<K, V> {
    private final int capacity;
    private final Map<K, Node<K, V>> map;
    private final Node<K, V> head;       // sentinel — most recent is head.next
    private final Node<K, V> tail;       // sentinel — least recent is tail.prev
    private final ReentrantLock lock = new ReentrantLock();

    public LRUCache(int capacity) {
        if (capacity <= 0) throw new IllegalArgumentException("capacity must be positive");
        this.capacity = capacity;
        this.map = new HashMap<>(capacity);
        this.head = new Node<>(null, null);
        this.tail = new Node<>(null, null);
        head.next = tail;
        tail.prev = head;
    }

    @Override
    public V get(K key) {
        lock.lock();
        try {
            Node<K, V> n = map.get(key);
            if (n == null) return null;
            moveToHead(n);
            return n.value;
        } finally { lock.unlock(); }
    }

    @Override
    public void put(K key, V value) {
        lock.lock();
        try {
            Node<K, V> n = map.get(key);
            if (n != null) {
                n.value = value;
                moveToHead(n);
                return;
            }
            Node<K, V> fresh = new Node<>(key, value);
            map.put(key, fresh);
            addToHead(fresh);
            if (map.size() > capacity) {
                Node<K, V> evicted = removeTail();
                map.remove(evicted.key);
            }
        } finally { lock.unlock(); }
    }

    @Override
    public V remove(K key) {
        lock.lock();
        try {
            Node<K, V> n = map.remove(key);
            if (n == null) return null;
            unlink(n);
            return n.value;
        } finally { lock.unlock(); }
    }

    @Override
    public int size() {
        lock.lock();
        try { return map.size(); } finally { lock.unlock(); }
    }

    @Override
    public void clear() {
        lock.lock();
        try {
            map.clear();
            head.next = tail;
            tail.prev = head;
        } finally { lock.unlock(); }
    }

    // ----- internal list ops (always called under lock) -----

    private void addToHead(Node<K, V> n) {
        n.prev = head;
        n.next = head.next;
        head.next.prev = n;
        head.next = n;
    }

    private void unlink(Node<K, V> n) {
        n.prev.next = n.next;
        n.next.prev = n.prev;
        n.prev = n.next = null;   // help GC
    }

    private void moveToHead(Node<K, V> n) {
        unlink(n);
        addToHead(n);
    }

    private Node<K, V> removeTail() {
        Node<K, V> lru = tail.prev;
        if (lru == head) throw new IllegalStateException("empty");
        unlink(lru);
        return lru;
    }
}
```

### Java shortcut: `LinkedHashMap`

For the same effect with one-fifth the code:

```java
public final class LRUCacheLite<K, V> extends LinkedHashMap<K, V> {
    private final int capacity;

    public LRUCacheLite(int capacity) {
        super(capacity, 0.75f, true);   // accessOrder = true → LinkedHashMap moves to tail on access
        this.capacity = capacity;
    }

    @Override
    protected boolean removeEldestEntry(Map.Entry<K, V> eldest) {
        return size() > capacity;
    }
}
```

Mention this in the interview as the "production" shortcut, but write the linked-list version when asked to implement from scratch — the interviewer wants to see the data structure manipulated by hand.

### Usage

```java
LRUCache<Integer, String> cache = new LRUCache<>(3);
cache.put(1, "one");      // [1]
cache.put(2, "two");      // [2, 1]
cache.put(3, "three");    // [3, 2, 1]
cache.get(1);             // [1, 3, 2]   — 1 is now MRU
cache.put(4, "four");     // [4, 1, 3]   — 2 evicted
System.out.println(cache.get(2));   // null
```

---

## Concurrency strategy

### What's contested
- The map (reads and writes from many threads).
- The doubly-linked list (mutation on every `get` and `put`).

### Why a single `ReentrantLock`
- The list mutation on `get` makes lock-free or read/write split *hard*. Even pure reads modify the structure (move to head). So a `ReadWriteLock` doesn't help — every operation is a "writer."
- A single lock around the whole cache is correct and simple.

### Higher-throughput options to mention

**1. Lock striping (sharded LRU)**
Split the cache into N independent LRUs, keyed by `key.hashCode() % N`. Each shard has its own lock. Reduces contention by N×.
```java
public final class StripedLRUCache<K, V> {
    private final LRUCache<K, V>[] shards;
    public StripedLRUCache(int totalCapacity, int shards) {
        this.shards = new LRUCache[shards];
        for (int i = 0; i < shards; i++) this.shards[i] = new LRUCache<>(totalCapacity / shards);
    }
    private LRUCache<K, V> shardFor(K key) {
        return shards[Math.floorMod(key.hashCode(), shards.length)];
    }
    public V get(K k)        { return shardFor(k).get(k); }
    public void put(K k, V v) { shardFor(k).put(k, v); }
}
```
Caveat: total LRU ordering is now approximate — eviction is per-shard. For most use cases this is fine.

**2. Caffeine / Guava Cache**
For production, never hand-roll. Caffeine uses a window-TinyLFU policy and is dramatically faster than even striped LRU. It supports TTL, weight, eviction listeners, async loading, and refresh-after-write.

**3. ConcurrentHashMap + sampled LRU**
Some implementations use `ConcurrentHashMap` for O(1) lookup and approximate LRU via per-entry timestamps and periodic eviction passes. Better concurrency, less precise.

---

## Edge cases / gotchas

| Case | Handling |
|---|---|
| **Capacity 0 or negative** | Constructor throws. |
| **`null` key** | If allowed, `HashMap` allows one null key — design choice. We could disallow with an explicit check. |
| **`null` value** | Indistinguishable from "absent" if `get` returns null on miss. Either disallow nulls, or return `Optional<V>`, or use a sentinel. |
| **Update on existing key** | Move-to-head, update value; do NOT count as new insertion. |
| **Reentrant call from a value's `equals`/`hashCode`** | `ReentrantLock` is reentrant — safe. |
| **Clear in mid-iteration** | Don't expose iterators on the internal map without copying first. |
| **Eviction during put** | Remove from list AND map together — easy to forget one. |
| **GC pressure on long-lived large cache** | `null` out `prev`/`next` of unlinked nodes to break references. |
| **Out-of-order operations under contention** | Lock guarantees serial semantics; relax only if you can prove it's safe (Caffeine does this). |

---

## Extensions / interview follow-ups

### "Add TTL (time-based expiration)."
- Add `expiresAt: Instant` to each `Node`.
- `get` checks `if (n.expiresAt != null && now() > n.expiresAt) { remove(n.key); return null; }`.
- For active cleanup: a background thread that scans periodically, OR a min-heap of `(expiresAt, key)` pairs and a sweeper.
- Caffeine uses **expire-after-write** and **expire-after-access** as separate policies — both are useful.

### "Switch to LFU."
- LFU evicts the least-frequently-used item. Implementation:
  - Map<key, node> for lookup.
  - Map<frequency, doubly-linked list of nodes> to find LFU candidates.
  - Maintain `minFreq`; on access, move node from freq-list `f` to `f+1`.
  - Evict by removing the head of the list at `minFreq`.
- All operations remain O(1).
- Refactor the existing class so the eviction logic is a Strategy: `EvictionPolicy<K,V>` with `recordAccess` / `recordInsertion` / `selectVictim`.

### "Add hit-rate metrics."
- Atomic counters for hits / misses / evictions / inserts.
- Expose `Stats stats()` returning a snapshot.

### "Eviction callbacks (write-behind)."
- Add `BiConsumer<K, V> onEvict` to the constructor.
- Call it in `evictLRU` after removing from the map. Run on a worker thread to avoid holding the lock during I/O.

### "Make it distributed."
- This is system-design territory. Sketch: a ring of cache nodes (like Memcached / Redis Cluster); consistent hashing routes keys to nodes; each node runs the local LRU.
- Eviction is local; no global LRU order.

### "What if values are huge — should I limit by total bytes, not entry count?"
- Add a `Weigher<V>` that returns the bytes of each value.
- Track `totalWeight`; evict until `totalWeight <= capacity`.
- Caffeine supports this directly.

### "Why a doubly-linked list and not an ArrayList with indices?"
- ArrayList: removing from middle is O(n). DLL: O(1) given a pointer to the node.
- The hashmap-to-node pointer lets us jump to the middle in O(1) and unlink in O(1).
- ArrayList of indices: would require shifting; or use a circular buffer with markers, but the bookkeeping complicates everything.

---

## Scaling

A single-process LRU cache is already at scale for most use cases — millions of operations per second on a single CPU. Beyond that:

### Single-process concurrency
- Lock striping gets you to ~10x improvement.
- Caffeine, with bounded operation queues and replay-on-eviction, gets you another order of magnitude.

### Multi-process / shared cache
- **Memcached** — distributed in-memory cache, sharded by consistent hashing. No replication. Each key lives on one node; that node's eviction is independent.
- **Redis** — same idea, with optional persistence and replication.
- For LLD purposes: design pivots from "data structure" to "service" — the cache becomes a network protocol over a shared backend.

### Tier-1/Tier-2 caching
- L1 in-process (small, fast, per-server).
- L2 distributed (Memcached/Redis, large, shared).
- L3 source of truth (DB).
- L1 hits avoid network; L1 misses fall through to L2; L2 misses fall through to DB.

### Cache stampede / thundering herd
- 1000 threads all miss the same key simultaneously, all hit the DB. Defenses:
  - **Single-flight**: per-key lock so only one thread loads; others wait.
  - **`ConcurrentHashMap.computeIfAbsent`**: if the load is in-process, this gives you single-flight for free.
  - **Stale-while-revalidate**: serve stale on miss while one thread refreshes.

### Consistency
- Distributed caches are **best-effort consistent** with the source of truth.
- Strategies: TTL, write-through (write to cache + DB), write-behind (cache asynchronously flushes to DB), or explicit invalidation on write.
- "Cache invalidation is one of the two hard problems in CS" — design the invalidation strategy as carefully as the cache itself.

---

## Persistence tradeoffs

LRU cache is unusual — its **whole purpose** is volatile, in-memory storage. So "persistence" here means one of three things, and the interviewer is usually probing whether you can tell them apart.

### 1. Persistent cache (cache survives restart)
You want the cache populated immediately after a deploy, not a cold start. Three approaches:

- **Disk snapshot**: serialize the cache periodically (every N minutes) to disk; reload on boot. Fast cold start, but data is stale by up to N minutes.
- **Redis as the cache itself**: not "in-memory in your process" but "in-memory in Redis." Survives your process restart. Loses data only on Redis restart.
- **Disk-backed LRU**: RocksDB / LevelDB store data on SSD with their own caching. Effectively "infinite memory" with lower throughput.

### 2. Distributed cache (cache shared across processes)
Single-process LRU is a non-starter for multi-instance services — each instance has its own cache, hit rate is low, and writes don't propagate.

- **Memcached**: distributed in-memory, sharded by consistent hashing. Each key lives on one node. No replication; nodes are ephemeral.
- **Redis Cluster**: sharded with optional replication. Persistence (RDB snapshots, AOF append) optional.
- **DynamoDB DAX**: Amazon's managed accelerator for DynamoDB. Read-through, write-through, transparent.
- **CDN edges**: for read-heavy public content, caching at the CDN layer (CloudFront, Fastly) is the highest-throughput option.

### 3. Cache + source of truth (the typical real shape)
Cache is **derived state**; the source of truth is a database. Write-paths choose:
- **Write-through**: write to DB then cache. Stale-cache window = 0; write latency = max of both.
- **Write-around**: write to DB only; let next read populate cache. Cache stays cold; useful for write-heavy data.
- **Write-back**: write to cache only; flush to DB asynchronously. Lowest write latency; risk of data loss on crash.
- **Cache-aside (read-through)**: app reads from cache; on miss, loads from DB and caches. Simplest, most common.

### Eviction policy in the persistent layer
- **Redis** has 8 eviction policies (`allkeys-lru`, `volatile-lru`, `noeviction`, etc.). Pick `allkeys-lru` to behave like the in-memory cache here.
- **Memcached** is LRU by default.
- **RocksDB / LevelDB** use SSD; eviction is "delete old levels" rather than LRU. Different model.

### TTL — orthogonal to LRU
- **TTL-only**: items expire on a clock; no size cap. Used for session stores, OTPs.
- **LRU-only**: size cap; eviction by recency. Used for code-level memoization.
- **TTL + LRU**: both, with whichever fires first evicting. Most production caches.

Redis supports all three natively. Caffeine (in-process) supports all three. Hand-rolled gets complex fast — use a library.

### Distributed-cache failure modes
- **Network partition between client and cache**: client either fails or falls through to DB. With many clients suddenly bypassing the cache, the DB can collapse — **cache stampede**.
- **Cache node dies**: rebalance keys to remaining nodes. With consistent hashing, only ~1/N of keys move. Without it, a full reshuffle (Memcached's old behavior).
- **Stale cache after DB failover**: the cache may have data from the now-promoted-replica's old timeline. Invalidate aggressively after failover.

### Recommendation
- **Single-process memoization**: Caffeine. In-process, lock-striped, TTL-aware, eviction-listened.
- **Service-to-service caching**: Redis (cluster). Treat as a remote service, with timeouts and circuit breakers.
- **CDN-cacheable HTTP responses**: CDN edges. Always.
- **Embedded "cache survives restart"**: snapshot approach if data is small; RocksDB if not.

### Concurrency under persistence
The hand-rolled `LRUCache<K, V>` we wrote is single-process. Once you move to Redis:
- Redis itself is single-threaded per key — no race conditions inside Redis.
- But "check-then-act" across a round trip is still racy: `if (redis.get(k) == null) { v = compute(); redis.set(k, v); }` allows multiple compute calls.
- Use **`SETNX` + load lock** pattern, or `redis.compute(k, supplier)` if your client supports it.
- For cluster: hash-tag related keys (`{user:42}.profile`, `{user:42}.cart`) so they live on the same shard if you need atomicity across them.

---

## Talking points for the interview

- "The two structures point at the same nodes — that's why both lookup and reordering are O(1)."
- "I'm using sentinels for head and tail to avoid null-checking edge cases — every real node always has both a prev and a next."
- "I'm locking the whole cache because the list mutation on `get` makes a read/write split unhelpful — every operation is a writer."
- "For high-throughput, I'd shard the cache. The trade-off is approximate LRU rather than global LRU — usually a fine trade."
- "In production, Caffeine. Window-TinyLFU outperforms LRU on real workloads because it's resistant to scan-thrashing patterns."
- "Cache invalidation is the source of most cache bugs. For TTL I'd implement expire-after-write and expire-after-access separately — they have different correctness implications."

---

## Summary

The skeleton: **doubly-linked list + HashMap, both pointing at the same nodes**. List ordering = LRU order; map provides O(1) jump-to-node. Get and put both touch the list and the map.

Concurrency: single lock is correct; lock striping is the first optimization; production is Caffeine.

Extensions are mostly **policy** changes (TTL, LFU, weight) — wrapping eviction in a Strategy interface makes them one-class swaps.
