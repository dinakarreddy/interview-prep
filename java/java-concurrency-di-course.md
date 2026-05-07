# Java Concurrency & Dependency Injection — Interview Prep Course

A self-contained reference distilled from a deep-dive coaching session. Aimed at intermediate Java engineers preparing for senior backend interviews.

---

## Table of Contents

1. [Module 1 — Java Memory Model & Threading Foundations](#module-1)
2. [Module 2 — `j.u.c` Toolkit + Six Interview Scenarios](#module-2)
3. [Module 3 — Concurrency Patterns & Anti-Patterns](#module-3)
4. [Module 4 — Dependency Injection Principles](#module-4)
5. [Module 5 — Spring (with Mental Model)](#module-5)
6. [Rabbit Holes](#rabbit-holes)
   - Reading Thread Dumps
   - AQS Internals
   - Virtual Threads (Project Loom)
   - `ConcurrentHashMap` Java 7 → 8 Evolution
   - Virtual Threads vs Reactive (Mono/Flux)
7. [Bonus — The Enum Singleton](#enum-singleton)
8. [Mock Interview — Round 1 (Rapid-fire) with Grading](#mock-1)
9. [Mock Interview — Round 2 (Design) with Grading](#mock-2)
10. [Cheatsheets](#cheatsheets)

---

<a id="module-1"></a>
# Module 1 — Java Memory Model & Threading Foundations

## 1.1 Why the JMM exists

Modern CPUs reorder instructions and cache values in registers/L1. Without rules, thread A's writes might never be visible to thread B — or might appear out of order. The **JMM** is a contract: *if you use these synchronization primitives correctly, I guarantee these visibility/ordering properties.*

Three distinct concerns — interviewers love testing whether you can separate them:

| Property | Question it answers | Primitive |
|---|---|---|
| **Atomicity** | "Can this op be torn?" | `synchronized`, `Atomic*`, locks |
| **Visibility** | "Will another thread see my write?" | `volatile`, `synchronized`, `final` |
| **Ordering** | "Can the JVM/CPU reorder these?" | `volatile`, `synchronized` |

Key trap: `volatile` gives **visibility + ordering** but **not atomicity**. So `volatile int counter; counter++;` is still broken — read-modify-write is three operations.

## 1.2 happens-before — the only rule that matters

If action A *happens-before* B, then A's effects are visible to B. The rules:

1. **Program order** — within a single thread, statements happen-before later statements
2. **Monitor lock** — unlock on monitor M happens-before every subsequent lock on M
3. **Volatile** — write to volatile v happens-before every subsequent read of v
4. **Thread start** — `Thread.start()` happens-before any action in the started thread
5. **Thread join** — all actions in a thread happen-before another thread's `t.join()` returns
6. **Transitivity** — if A hb B and B hb C, then A hb C

Everything in `j.u.c` (latches, queues, executors) is built on these.

## 1.3 `synchronized` — what it actually does

```java
synchronized (lock) {
    // critical section
}
```

Compiles to `monitorenter` / `monitorexit` bytecode. Three guarantees:
- **Mutual exclusion** — only one thread holds the monitor
- **Visibility** — entering reads memory fresh; exiting flushes
- **Reentrancy** — same thread can re-acquire

### Where the lock lives — the object header

Every Java object has a 12–16 byte header with the **mark word**, encoding lock state:

| State | Mark word holds |
|---|---|
| Unlocked | hash code, GC age |
| Thin lock | pointer to lock record on owning thread's stack |
| Fat lock (inflated) | pointer to a heap-allocated `ObjectMonitor` |
| Biased (legacy, removed JDK 18+) | thread ID of biased owner |

Lock progression: unlocked → thin → fat. Once inflated to fat, an `ObjectMonitor` struct holds the wait-set, entry-set, and owner.

### Reentrancy

Each monitor has an internal counter. Same thread acquiring twice → counter = 2. Each `monitorexit` decrements; lock fully releases at 0.

### `synchronized` + `wait/notify`

`Object.wait()` *requires* you hold the monitor. It atomically:
1. Releases the monitor
2. Suspends on the wait-set
3. On wake, re-acquires before returning

**Always wait in a loop**, never an `if`:

```java
synchronized (queue) {
    while (queue.isEmpty()) {        // ✅ while, not if
        queue.wait();
    }
    return queue.removeFirst();
}
```

Reasons: spurious wakeups + stale conditions.

### Common `synchronized` traps

| Trap | Why bad |
|---|---|
| `synchronized(stringLiteral)` | Strings interned globally |
| `synchronized(autoboxedInteger)` | `Integer.valueOf(1)` returns shared cached instance |
| Locking on `this` while exposing it | Outsiders can grab your monitor |
| Wide critical sections | Throughput killer |
| Holding lock during alien callbacks | Deadlock recipe |

### `synchronized` vs `ReentrantLock`

|  | `synchronized` | `ReentrantLock` |
|---|---|---|
| Acquire | Implicit, scoped | Explicit `lock()/unlock()` |
| Interruptible wait | No | Yes |
| Timed acquire | No | `tryLock(timeout)` |
| Fairness | No | Optional |
| Multiple condition vars | One wait-set | `newCondition()` — many |

Default to `synchronized`. Reach for `ReentrantLock` for interruptibility, timed acquire, fairness, or multiple conditions.

## 1.4 `volatile` deep dive

### Memory barriers

`volatile` is implemented as memory fences:

| Operation | Barrier inserted |
|---|---|
| `volatile` **read** | LoadLoad + LoadStore *after* the read |
| `volatile` **write** | LoadStore + StoreStore *before*, StoreLoad *after* |

Conceptually:
- Volatile **write** = release semantics — everything done before it is published
- Volatile **read** = acquire semantics — everything you do after sees the published state

### Piggybacking on synchronization

```java
class Holder {
    int x, y, z;             // plain
    volatile boolean ready;  // the gate

    void publish() {
        x = 1; y = 2; z = 3; // (A) plain writes
        ready = true;        // (B) volatile write — releases A
    }
    void consume() {
        if (ready) {         // (C) volatile read — acquires
            // Guaranteed to see x=1, y=2, z=3
            use(x, y, z);
        }
    }
}
```

The non-volatile fields get visibility for free via the volatile fence.

### Why `volatile counter++` is broken

`counter++` is read → add → write. Volatile makes each atomic *individually* but not as a unit. Two threads can both read 5, both compute 6, both write 6 — lost increment. Use `AtomicInteger` or a lock.

### Good uses of `volatile`

1. Status flags (`volatile boolean shutdown`)
2. One-time safe publication (DCL singleton)
3. Independent observation (`lastHeartbeat`)
4. Wait-free swap of immutable references (config snapshot)

### `volatile` traps

```java
volatile int[] arr;
arr[0] = 42; // ❌ NOT a volatile write of arr[0] — only the reference is volatile
```

Use `AtomicIntegerArray` or `VarHandle` for per-element semantics.

## 1.5 `final` field semantics

When a constructor finishes, all `final` fields are guaranteed visible to any thread that gets a reference — **even without synchronization**. This is why immutable objects (`String`, `LocalDate`) are safe to share freely.

## 1.6 Thread lifecycle

States: `NEW → RUNNABLE ⇄ (BLOCKED | WAITING | TIMED_WAITING) → TERMINATED`.

| State | Meaning |
|---|---|
| **NEW** | Constructed; `start()` not called |
| **RUNNABLE** | Eligible for CPU OR in OS-blocking syscall (read/recv) |
| **BLOCKED** | Waiting to acquire a monitor lock |
| **WAITING** | Indefinite wait: `Object.wait()`, `Thread.join()`, `LockSupport.park()` |
| **TIMED_WAITING** | Same triggers with timeout, plus `Thread.sleep` |
| **TERMINATED** | `run()` returned or threw; cannot restart |

### BLOCKED vs WAITING (interview gold)

| | BLOCKED | WAITING |
|---|---|---|
| Cause | Contending for monitor | Voluntary suspension |
| Wakes via | Lock becoming available | Notify / unpark / join target dies |
| Held by | OS lock contention | JVM wait-set / parking |

Lots of BLOCKED in a thread dump → look for lock contention. Lots of WAITING → likely thread pool idle, or condition not being signaled.

### `park`/`unpark`

`LockSupport.park()` / `unpark(thread)` is what `j.u.c` uses. Differences from `wait/notify`:
- No monitor required
- Permit-based — `unpark` issued before park makes park return immediately
- Targets a specific thread, not a wait-set

### Interruption

`t.interrupt()` does two things:
1. Sets the interrupt flag on `t`
2. If `t` is in WAITING/TIMED_WAITING/sleep, wakes it with `InterruptedException`. **The flag is cleared when the exception is thrown.**

What does *not* respond to interrupt:
- Plain `synchronized` acquisition (use `ReentrantLock.lockInterruptibly`)
- Blocking I/O on `InputStream` (use `InterruptibleChannel` / NIO)

The 3-rule interrupt protocol:
1. **If you can throw, throw `InterruptedException`** — propagate
2. **If you can't throw, restore the flag**: `Thread.currentThread().interrupt()`
3. **Never silently swallow** — destroys cancellation

### `sleep` doesn't release the lock

```java
synchronized (lock) {
    Thread.sleep(10_000);  // still holding lock — others wait
}
```

If you wanted others to proceed, you needed `wait()`, which releases.

### Daemon vs user threads

Daemon threads don't keep the JVM alive. Set with `t.setDaemon(true)` *before* `start()`. **Daemons are killed without `finally` blocks running** at JVM exit.

## 1.7 Exercise — DCL Singleton

The classic broken version:

```java
public class BrokenSingleton {
    private static BrokenSingleton instance;   // ❌

    public static BrokenSingleton getInstance() {
        if (instance == null) {
            synchronized (BrokenSingleton.class) {
                if (instance == null) {
                    instance = new BrokenSingleton();
                }
            }
        }
        return instance;
    }
}
```

**Why broken:** `new BrokenSingleton()` is allocate → construct → assign. The JVM may reorder so the reference is published *before* construction completes. A second thread's lock-free read sees a non-null but partially-constructed object.

### Fix A — `volatile`

```java
private static volatile BrokenSingleton instance;
```

Volatile prevents the publication reordering. The JLS guarantees writes before the volatile write are visible to readers that see the volatile write.

### Fix B — Initialization-on-Demand Holder

```java
public class Singleton {
    private static class Holder { static final Singleton I = new Singleton(); }
    public static Singleton get() { return Holder.I; }
}
```

JVM class-initialization lock guarantees once-only init, lazy-on-first-reference, with happens-before semantics. **No `volatile`, no `synchronized` needed.** This is the recommended idiom.

### Fix C — Enum singleton

```java
public enum Singleton { INSTANCE; ... }
```

Same JVM init guarantees, plus reflection-safe, serialization-safe, clone-safe. Joshua Bloch's recommendation in *Effective Java*.

---

<a id="module-2"></a>
# Module 2 — `j.u.c` Toolkit

## 2.1 What is `j.u.c`?

`java.util.concurrent` — the standard library package introduced in Java 5 (JSR-166, Doug Lea) that ships professional-grade concurrency primitives.

## 2.2 The six categories

### 1. Executors & thread pools
- `Executor`, `ExecutorService`, `ScheduledExecutorService`
- `Executors` (factory), `ThreadPoolExecutor`, `ForkJoinPool`
- `Future`, `CompletableFuture`

### 2. Concurrent collections
- `ConcurrentHashMap` — striped/CAS-based
- `CopyOnWriteArrayList`, `CopyOnWriteArraySet` — read-mostly
- `ConcurrentLinkedQueue` — lock-free
- `ConcurrentSkipListMap` — sorted

### 3. Blocking queues
- `ArrayBlockingQueue` — bounded, array-backed
- `LinkedBlockingQueue` — optionally bounded, separate head/tail locks
- `SynchronousQueue` — zero capacity, hand-off
- `PriorityBlockingQueue`, `DelayQueue`, `LinkedTransferQueue`

### 4. Locks & synchronizers (`java.util.concurrent.locks`)
- `ReentrantLock`, `ReentrantReadWriteLock`, `StampedLock`
- `Condition`, `LockSupport`, `AbstractQueuedSynchronizer` (AQS)

### 5. Coordination primitives
- `CountDownLatch`, `CyclicBarrier`, `Semaphore`, `Phaser`, `Exchanger`

### 6. Atomics (`java.util.concurrent.atomic`)
- `AtomicInteger/Long/Reference/Boolean`
- `AtomicIntegerArray`, `AtomicReferenceArray`
- `LongAdder`, `LongAccumulator` — high-contention counters
- `AtomicStampedReference` — solves ABA

Almost every blocking primitive is built on AQS — see Rabbit Hole 2.

## 2.3 Six interview scenarios

### Scenario 1 — "Wait until 5 services are ready" → `CountDownLatch`

```java
CountDownLatch ready = new CountDownLatch(5);
// Each init thread: ready.countDown();
// Main: ready.await();
startHttpServer();
```

**One-shot.** Once it hits 0, stays 0. Can't reset — use `CyclicBarrier` or `Phaser` if you need reset.

**Why not alternatives:**
- `CyclicBarrier(5)` — barriers wait for N parties to *meet*; here the waiter isn't one of the 5
- `Semaphore(0)` + 5 releases — works but obscures intent

### Scenario 2 — "Limit DB connections to 20" → `Semaphore`

```java
Semaphore pool = new Semaphore(20);
void query() throws InterruptedException {
    pool.acquire();
    try { /* use connection */ }
    finally { pool.release(); }
}
```

Semaphore vs binary mutex: **semaphores have no notion of ownership** — any thread can release. This makes them usable for hand-off patterns.

Fair vs unfair: `new Semaphore(20, true)` = FIFO, prevents starvation, ~2x slower.

### Scenario 3 — Producer/consumer with backpressure → `BlockingQueue`

Method matrix (memorize):

|  | Throws ex | Special value | Blocks | Times out |
|---|---|---|---|---|
| **insert** | `add(e)` | `offer(e)` | `put(e)` | `offer(e, t, u)` |
| **remove** | `remove()` | `poll()` | `take()` | `poll(t, u)` |
| **examine** | `element()` | `peek()` | — | — |

| Impl | Use when |
|---|---|
| `ArrayBlockingQueue` | Fixed capacity, locality |
| `LinkedBlockingQueue` | Higher throughput (separate locks) |
| `SynchronousQueue` | Zero capacity hand-off |
| `PriorityBlockingQueue` | Priority order; **unbounded** |
| `DelayQueue` | Delayed elements |

### Scenario 4 — High-contention counter → `LongAdder`

`AtomicLong.incrementAndGet()` is a CAS loop on a single memory location. With many threads, you get **cache-line ping-pong**.

`LongAdder` solves this with **striping**: an array of `Cell`s (padded to avoid false sharing). Each thread hashes to a cell and CAS-es it. Reads sum all cells.

Trade-off: writes scale linearly with cores; reads are O(stripes) and not atomic w.r.t. writes (weak snapshot). Perfect for counters, bad if you need exact reads under writes.

| Want | Use |
|---|---|
| Single-writer counter | plain `long` |
| Low-contention RMW | `AtomicLong` |
| High-contention counter | `LongAdder` |
| Atomic max/min/custom op | `LongAccumulator` |

### Scenario 5 — Read-mostly cache → `ConcurrentHashMap` or `RWLock`

**`ConcurrentHashMap` (preferred default):**

```java
ConcurrentHashMap<String, User> cache = new ConcurrentHashMap<>();
User u = cache.computeIfAbsent(id, k -> loadFromDb(k));
```

- Reads are **lock-free** (volatile reads of bucket head)
- Writes lock only the affected bucket
- Resize is concurrent (multiple threads cooperate)
- `size()` is approximate (striped counter)

Atomic compound ops: `putIfAbsent`, `computeIfAbsent`, `compute`, `merge`. Critical: the function in `compute*` runs **while the bucket is locked**. No I/O, no callbacks into the same map.

**`ReentrantReadWriteLock`** — when you need to coordinate access to *something other than a Map*. Many readers OR one writer. Cannot upgrade read → write. **`StampedLock`** adds optimistic read mode (no locking on fast path; validate after).

### Scenario 6 — 1000 parallel HTTP calls → `CompletableFuture` + `ExecutorService`

```java
ExecutorService io = Executors.newFixedThreadPool(50);
List<CompletableFuture<Response>> futures = urls.stream()
    .map(url -> CompletableFuture.supplyAsync(() -> http.get(url), io))
    .toList();
CompletableFuture.allOf(futures.toArray(CompletableFuture[]::new)).join();
List<Response> results = futures.stream().map(CompletableFuture::join).toList();
```

| Method | Effect |
|---|---|
| `thenApply(fn)` | Sync transform A → B |
| `thenCompose(fn)` | Flatmap — fn returns a CF |
| `thenCombine(other, bi)` | Combine two CFs |
| `exceptionally(fn)` | Recover from failure |
| `handle(bi)` | Always runs |
| `allOf(...)` / `anyOf(...)` | Aggregate |

`*Async` variants run on an executor instead of pinning the previous-stage thread. Critical for separating CPU and I/O.

**Pitfalls:**
1. **Default executor is `ForkJoinPool.commonPool()`** — sized to `cores - 1`. Blocking I/O starves it. Always pass an explicit executor.
2. **Exception swallowing** — must call `.exceptionally`, `.handle`, or eventually `.join()` to surface errors.
3. **`allOf` returns `CompletableFuture<Void>`** — join it, then collect from originals.
4. **Cancellation isn't real** — `cancel(true)` doesn't interrupt the underlying task; just marks the CF.

---

<a id="module-3"></a>
# Module 3 — Concurrency Patterns & Anti-Patterns

## 3.1 Five canonical patterns

### Producer/Consumer
Decouple via queue. Producers push, consumers pull. Queue absorbs rate mismatch and provides backpressure.

```java
BlockingQueue<Task> q = new ArrayBlockingQueue<>(1000);
// Producer: q.put(source.next());
// Consumer: Task t = q.take(); if (t == POISON) break; t.process();
```

Shutdown via **poison pill** or `ExecutorService.shutdown()` + interruption.

### Worker Pool
Standard `ExecutorService`. **Always construct `ThreadPoolExecutor` directly** with bounded queue + sensible rejection policy:

```java
new ThreadPoolExecutor(
    8, 8, 0L, TimeUnit.MILLISECONDS,
    new LinkedBlockingQueue<>(1000),
    new ThreadPoolExecutor.CallerRunsPolicy()  // free backpressure
);
```

`Executors.newFixedThreadPool` uses unbounded queue → OOM risk.

| Rejection policy | Behavior |
|---|---|
| `AbortPolicy` (default) | Throws `RejectedExecutionException` |
| `CallerRunsPolicy` | Submitting thread runs the task — naturally throttles |
| `DiscardPolicy` | Silently drops |
| `DiscardOldestPolicy` | Drops head, retries submit |

### Fan-out / Fan-in
Parallel subtasks; combine results. `CompletableFuture.allOf(...)` for "wait all," `anyOf(...)` for "fastest wins" (great for tail-latency mitigation).

`ForkJoinPool` for recursive divide & conquer with **work-stealing**: idle workers steal from busy workers' deques.

### Thread Confinement
The fastest synchronization is none. Confine mutable state to a single thread; communicate via messages.
- Swing/JavaFX EDT
- Per-thread `SimpleDateFormat` via `ThreadLocal`
- Single-writer event loops (Netty, Vert.x, Disruptor)

**`ThreadLocal` traps:**
- Memory leak in app servers — thread pools reuse threads forever; always `remove()`
- Doesn't propagate to child threads — use `InheritableThreadLocal` or `ScopedValue` (Java 21+)

### Immutability + Safe Publication
Recipe:
1. All fields `final`
2. All fields private
3. No setters
4. No mutable state escapes (defensive copy on read)
5. Class itself `final`

Returns new instance for "updates" (e.g., `user.withName("Alice")`). Eliminates whole bug categories — torn reads, mid-construction visibility, sharing coordination.

## 3.2 The four sins

### Deadlock
Coffman conditions (all must hold):
1. Mutual exclusion
2. Hold-and-wait
3. No preemption
4. Circular wait

Break any one → impossible.

**Defenses:**
- **Lock ordering** — globally agreed total order (e.g., by hash code). Always acquire in that order
- **`tryLock` with timeout + back-off**
- **Open calls** — never invoke alien code while holding a lock

JVM auto-detects deadlocks in thread dumps.

### Livelock
Threads aren't blocked but make no progress — keep reacting to each other. Cause: retry loops without back-off. Fix: randomized exponential back-off, jitter.

### Starvation
Thread never gets CPU/lock/resource. Causes: unfair locks, priority inversion, spinning under contention. Defenses: fair locks, bounded queues, avoid priorities.

### Race conditions
Three flavors:

**Check-then-act:**
```java
if (!cache.containsKey(k)) cache.put(k, expensiveLoad(k));   // ❌
// Fix: cache.computeIfAbsent(k, this::expensiveLoad);
```

**Read-modify-write:**
```java
counter = counter + 1;   // ❌ not atomic
// Fix: AtomicInteger.incrementAndGet()
```

**Multi-step invariant:**
```java
account.balance -= 100; other.balance += 100;  // total wrong between steps
// Fix: lock covering both
```

The *invariant* defines the critical section, not any one variable.

## 3.3 Performance & correctness anti-patterns

1. **Holding a lock during I/O / callbacks** — prepare data inside lock, call out outside
2. **Object pooling everything** — pool only truly expensive resources (connections, threads)
3. **`Thread.sleep` as synchronization** — brittle; use `CountDownLatch` or polling
4. **Catching `InterruptedException` and ignoring** — destroys cancellation
5. **Premature lock-splitting** — measure first
6. **`volatile` for compound state** — only protects the reference
7. **Synchronizing on mutable fields** — lock fields must be `final`
8. **`Vector`/`Hashtable`/`Collections.synchronizedX`** — single global lock; iteration not atomic
9. **Unbounded queues in pools** — OOM
10. **Double-checked locking without `volatile`** — partially constructed object visible

## 3.4 Three classic interview puzzles

### FooBar (alternating print)
Best with Semaphore hand-off:
```java
Semaphore foo = new Semaphore(1);
Semaphore bar = new Semaphore(0);
void foo() { for (int i=0;i<n;i++) { foo.acquire(); print("Foo"); bar.release(); } }
void bar() { for (int i=0;i<n;i++) { bar.acquire(); print("Bar"); foo.release(); } }
```

### Dining Philosophers
Solutions:
1. **Resource ordering** — pick up lower-numbered first; breaks circular wait
2. **Waiter** — mediator allows ≤4 simultaneously; breaks hold-and-wait
3. **`tryLock` with back-off** — breaks no-preemption; risk livelock without randomization

Interviewer wants you to name the Coffman condition each solution breaks.

### Building H₂O (CyclicBarrier)
```java
Semaphore h = new Semaphore(2);   // 2 H slots
Semaphore o = new Semaphore(1);   // 1 O slot
CyclicBarrier b = new CyclicBarrier(3);
```

`Semaphore` enforces ratio; `CyclicBarrier` synchronizes molecule formation.

## 3.5 Diagnostic checklist

For any concurrent code, mentally run through:

1. **What state is shared?** List every mutable field accessible from multiple threads
2. **What's the invariant?** Often spans multiple fields
3. **What's the synchronization policy?** For each shared field: immutable / confined / guarded by which lock / volatile / atomic
4. **Are compound actions atomic?** Find every check-then-act, RMW, multi-field invariant
5. **Do I hold a lock during alien calls?** Listeners, callbacks, I/O
6. **Is there a lock-ordering convention?**
7. **Can blocked threads be cancelled?** Are interrupts handled correctly?
8. **What's the rejection / overload policy?**
9. **What's the shutdown story?**
10. **What does a thread dump look like under load?**

## 3.6 Exercise — `StatsCache` review

Original buggy class:

```java
public class StatsCache {
    private static StatsCache INSTANCE;
    public static StatsCache get() {
        if (INSTANCE == null) INSTANCE = new StatsCache();
        return INSTANCE;
    }
    private final Map<String, Long> counts = new HashMap<>();
    private long total = 0;

    public synchronized void record(String key) {
        if (!counts.containsKey(key)) counts.put(key, 0L);
        counts.put(key, counts.get(key) + 1);
        total = total + 1;
        notifyListeners();   // ❌ alien call under lock
    }
    public long getCount(String key) { return counts.getOrDefault(key, 0L); }
    public Map<String, Long> snapshot() { return counts; }   // ❌ leaks live map
    public void shutdown() {
        try { Thread.sleep(100); } catch (InterruptedException e) {}   // ❌
    }
}
```

**Bugs:**
1. Lazy singleton race — DCL needed or use Holder idiom
2. `HashMap` not thread-safe for `getCount` (unsynchronized read)
3. `containsKey` + `get` + `put` is check-then-act → use `merge(key, 1L, Long::sum)`
4. **Open-call violation** — `notifyListeners()` while holding lock
5. **`getCount()` unsynchronized** — concurrent modification with `record()`
6. **`snapshot()` leaks live map** — return `Map.copyOf(counts)`
7. **`shutdown()` swallows InterruptedException + uses sleep as sync**
8. `total` field independent of counts — invariant broken

**Cleaned-up version:**

```java
public final class StatsCache {
    private static final class Holder { static final StatsCache I = new StatsCache(); }
    public static StatsCache get() { return Holder.I; }

    private final ConcurrentHashMap<String, Long> counts = new ConcurrentHashMap<>();
    private final LongAdder total = new LongAdder();
    private final List<Listener> listeners = new CopyOnWriteArrayList<>();
    private volatile boolean shuttingDown = false;

    public void record(String key) {
        if (shuttingDown) return;
        counts.merge(key, 1L, Long::sum);
        total.increment();
        for (Listener l : listeners) l.onRecord(key);  // open call OUTSIDE lock
    }
    public long getCount(String key) { return counts.getOrDefault(key, 0L); }
    public long total() { return total.sum(); }
    public Map<String, Long> snapshot() { return Map.copyOf(counts); }
    public void shutdown() { shuttingDown = true; }
}
```

---

<a id="module-4"></a>
# Module 4 — Dependency Injection Principles

## 4.1 The problem

```java
class OrderService {
    private final EmailClient email = new SmtpEmailClient("smtp.foo.com", 25);
    private final PaymentGateway pay = new StripeGateway(System.getenv("STRIPE_KEY"));
    private final OrderRepo repo = new PostgresOrderRepo(loadDbConfig());
    // ...
}
```

Six issues:
1. Untestable
2. Tight coupling to concrete impls
3. Hidden dependencies (constructor signature lies)
4. Lifecycle confusion
5. Mixed concerns (construction + business)
6. Configuration buried

Unifying problem: the class is responsible for both *what to do* and *who to use*. DI separates these.

## 4.2 Vocabulary

### Inversion of Control (IoC)
General principle: framework calls your code. "Don't call us, we'll call you."

### Dependency Injection (DI)
Specific application of IoC: a class **declares** its dependencies; something else **provides** them.

### Dependency Inversion Principle (DIP — "D" in SOLID)
> High-level modules depend on abstractions, not concretes. Abstractions don't depend on details.

DI is the **mechanism**; DIP is the **principle**. **Inject interfaces.**

### Service Locator (anti-pattern)

```java
class OrderService {
    void placeOrder(Order o) {
        EmailClient email = ServiceLocator.get(EmailClient.class);  // ❌
    }
}
```

| Concern | DI | Service Locator |
|---|---|---|
| Dependencies visible? | Yes | No |
| Testable without setup? | Yes | No |
| Compile-time check? | Yes | No |
| Hidden global state? | No | Yes |

## 4.3 Forms of injection

### Constructor injection (95% of the time)

```java
class OrderService {
    private final OrderRepo repo;
    private final PaymentGateway pay;
    OrderService(OrderRepo repo, PaymentGateway pay) {
        this.repo = repo; this.pay = pay;
    }
}
```

**Why it dominates:**
1. **Immutability** — `final` fields, automatic thread-safety
2. **Fail-fast** — missing dep = construction error
3. **Explicit contract** — constructor signature IS documentation
4. **Forces design pressure** — 12-arg ctor reveals "this class does too much"
5. **Works with any framework or none**

### Setter injection
Use only when dependency is genuinely optional. Drawback: mutable, half-built state.

### Field injection (avoid)

```java
class OrderService {
    @Autowired private OrderRepo repo;     // ❌
}
```

Cons: no `final`, untestable without reflection, hidden dependency.

## 4.4 Composition root

The single, well-known place where the concrete object graph is built:

```java
public class Application {
    public static void main(String[] args) {
        Config config = Config.load();
        OrderRepo repo = new PostgresOrderRepo(config.dbUrl);
        PaymentGateway pay = new StripeGateway(config.stripeKey);
        EmailClient email = new SmtpEmailClient(config.smtpHost);
        OrderService orders = new OrderService(repo, pay, email);
        new Server(orders).start();
    }
}
```

**Only the composition root knows concrete classes.** Everything else uses interfaces. Spring's `ApplicationContext` automates this.

## 4.5 Scopes (object lifetimes)

| Scope | When |
|---|---|
| Singleton | One per app |
| Prototype | Fresh every injection |
| Per-request (web) | One per HTTP request |
| Per-session | One per HTTP session |
| Per-thread | `ThreadLocal`-like |

**Scope mismatch trap:** injecting per-request bean into singleton — the singleton captures one instance forever. Solutions: scoped proxy, `Provider<T>`, or pass as method args.

## 4.6 Testability

```java
@Test void chargesCustomerWhenOrderSaved() {
    OrderRepo fakeRepo = new InMemoryOrderRepo();
    PaymentGateway mockPay = Mockito.mock(PaymentGateway.class);
    OrderService svc = new OrderService(fakeRepo, mockPay, msg -> {});
    svc.placeOrder(testOrder);
    verify(mockPay).charge(any());
}
```

No framework, no `@SpringBootTest`. ~10ms per test. The *property of explicit dependencies* enables this.

## 4.7 When DI is overkill

For tiny CLI scripts, prototypes, throwaway code — just `new` things up. DI's value compounds with multiple environments, real test coverage, and team size.

## 4.8 Manual vs container

| App size | Approach |
|---|---|
| < 10 classes | Manual |
| 10–50 | Manual or lightweight (Guice) |
| 50+ | Container (Spring) |
| Enterprise web | Spring Boot |

**Build small services manually first.** Spring stops feeling like magic when it's automating something you've done by hand.

## 4.9 Anti-patterns

1. Service Locator
2. Container as parameter (`OrderService(SpringContext ctx)`)
3. Required setters
4. Field injection on required deps
5. **God Object** (15-arg constructor)
6. Newing up inside methods
7. Static method calls for collaborators (except ambient like logging)
8. Disguised registry lookups

**Diagnostic:** if I deleted the framework, could I still construct and use this class?

## 4.10 Exercise — `ReportGenerator` refactor

Original violations:
1. **Hardcoded `Connection`** — DIP violation, untestable, class-load hazard
2. **`ServiceRegistry.lookup(...)`** — Service Locator
3. **Newing `MysqlConnection`/`SlackNotifier` mid-method** — hidden deps
4. **`@Autowired` field for logger** — field injection
5. **`new Date()` mid-method** — temporal coupling; inject `Clock`
6. **`if (type.equals("monthly"))`** — Strategy pattern smell; replace with `List<ReportStrategy>`
7. **SRP violation** — class queries DB, builds report, emails, notifies, timestamps
8. **`System.getenv` inside class** — config concern leaking into domain
9. **Public mutable static `INSTANCE`** — broken singleton

**Refactored:**

```java
public final class ReportService {
    private final OrderRepo repo;
    private final ReportEmailer emailer;
    private final ReportNotifier notifier;
    private final Clock clock;
    private final List<ReportStrategy> strategies;

    public ReportService(OrderRepo repo, ReportEmailer emailer,
                         ReportNotifier notifier, Clock clock,
                         List<ReportStrategy> strategies) {
        this.repo = repo;
        this.emailer = emailer;
        this.notifier = notifier;
        this.clock = clock;
        this.strategies = List.copyOf(strategies);
    }

    public Report generate(String type) {
        ReportStrategy s = strategies.stream()
                .filter(x -> x.handles(type))
                .findFirst()
                .orElseThrow(() -> new IllegalArgumentException("Unknown type: " + type));
        Report r = s.build(repo.findAll());
        emailer.send(r.summary());
        notifier.notify(r.summary(), clock.instant());
        return r;
    }
}
```

The constructor signature now tells the whole story.


---

<a id="module-5"></a>
# Module 5 — Spring (with Mental Model Woven In)

**Headline:** Spring is your composition root, automated. Every annotation is shorthand for "register this in the object graph with these wiring rules." Reflection reads your annotations, builds the graph, instantiates in topological order, hands you the assembled application.

## 5.1 `ApplicationContext` mental model

```
ApplicationContext (the container)
├── Map<String, BeanDefinition>   ← "recipes" parsed from annotations
├── Map<String, Object>           ← actual instances (singleton cache)
├── Dependency graph              ← who depends on whom
├── Lifecycle hooks               ← @PostConstruct, @PreDestroy
└── Environment                   ← properties, profiles
```

**Boot sequence:**

1. **Scan & parse** — find every `@Component`/`@Configuration`/`@Bean`; build `BeanDefinition`s
2. **`BeanFactoryPostProcessor`s run** — they mutate definitions (placeholder resolution, etc.)
3. **Instantiate singletons** — topological order, leaf deps first
4. **Inject** — constructor args, then setters/fields
5. **`BeanPostProcessor.postProcessBeforeInitialization`** — `@PostConstruct` runs; AOP proxies wrap beans
6. **Init callbacks** — `InitializingBean.afterPropertiesSet`, custom init methods
7. **`postProcessAfterInitialization`** — more proxy wrapping
8. **Context ready** — `ApplicationRunner`/`CommandLineRunner` fire
9. **Web container starts** accepting traffic
10. **Shutdown** — `@PreDestroy` → `DisposableBean.destroy` → `@Bean(destroyMethod=)`

`BeanFactory` = bare core (lazy, minimal). `ApplicationContext` = `BeanFactory` + i18n + events + autowiring config. **You use `ApplicationContext`.**

## 5.2 Stereotype annotations

```java
@Component   // generic
@Service     // service-layer marker
@Repository  // data-access; ALSO enables exception translation
@Controller  // web; routes to view name
@RestController  // @Controller + @ResponseBody
```

Differences are mostly semantic except `@Repository`, which translates JDBC/Hibernate exceptions to Spring's `DataAccessException`.

## 5.3 `@Configuration` + `@Bean`

For wiring third-party code you can't annotate:

```java
@Configuration
public class InfrastructureConfig {
    @Bean
    public DataSource dataSource(DbProperties props) {
        return new HikariDataSource(toHikariConfig(props));
    }
    @Bean
    public JdbcTemplate jdbcTemplate(DataSource ds) {
        return new JdbcTemplate(ds);
    }
}
```

Each `@Bean` method = one line of `main()`.

### CGLIB trick

By default, `@Configuration` classes are CGLIB-subclassed at runtime so calls to `@Bean` methods return the cached singleton:

```java
@Configuration
class Cfg {
    @Bean A a() { return new A(); }
    @Bean B b() { return new B(a()); }   // intercepted → returns cached A
}
```

`@Configuration(proxyBeanMethods = false)` (lite mode) — no proxy, faster startup, but you must inject collaborators as method args.

## 5.4 Wiring

### Constructor injection (no annotation needed since Spring 4.3)

```java
@Service
public class OrderService {
    private final OrderRepo repo;
    private final PaymentGateway pay;
    public OrderService(OrderRepo repo, PaymentGateway pay) { ... }   // ✅ no @Autowired
}
```

Single constructor → auto-wired. Always prefer this.

### Disambiguation

```java
@Service @Primary class StripeGateway implements PaymentGateway {}
@Service class AdyenGateway implements PaymentGateway {}

// At injection site:
OrderService(@Qualifier("adyenGateway") PaymentGateway pay) { ... }

// Inject all:
OrderService(List<PaymentGateway> all) { ... }
OrderService(Map<String, PaymentGateway> byName) { ... }
```

`List<T>` / `Map<String, T>` injection = built-in Strategy pattern.

### `@Value` + `@ConfigurationProperties`

```java
@Service
class S3Uploader {
    public S3Uploader(@Value("${app.s3.bucket}") String bucket) { ... }
}

// Better for grouped config:
@ConfigurationProperties(prefix = "app.s3")
public record S3Properties(String bucket, String region, int maxRetries) {}
```

## 5.5 Bean lifecycle and scopes

| Scope | When |
|---|---|
| `singleton` (default) | Stateless services, repos |
| `prototype` | Fresh every injection |
| `request` (web) | One per HTTP request |
| `session` (web) | One per HTTP session |

### Scope mismatch trap

```java
@Service                                    // singleton
class OrderProcessor {
    OrderProcessor(@Scope("request") RequestContext ctx) { ... }  // ❌ frozen forever
}
```

**Fixes:**
1. Scoped proxy: `@Scope(value = "request", proxyMode = ScopedProxyMode.TARGET_CLASS)`
2. Inject `ObjectProvider<T>` and call `getObject()` per request
3. Pass per-request data as method args (cleanest)

### Lifecycle hooks

```java
@Service
class Worker {
    @PostConstruct void start() { /* runs after deps injected */ }
    @PreDestroy void stop() { /* runs at shutdown */ }
}
```

Heavy init → `@PostConstruct`, not constructor (bean isn't fully wired yet).

## 5.6 Component scanning

`@SpringBootApplication` bundles `@Configuration` + `@EnableAutoConfiguration` + `@ComponentScan` (scoped to its package).

## 5.7 Profiles & conditionals

```java
@Service @Profile("test")  class InMemoryRepo implements Repo {}
@Service @Profile("!test") class JdbcRepo     implements Repo {}

@Bean @ConditionalOnProperty("feature.payments.stripe.enabled")
PaymentGateway stripe() { ... }

@Bean @ConditionalOnMissingBean(PaymentGateway.class)
PaymentGateway noOp() { ... }
```

Auto-configuration is built on `@Conditional*` — beans only fire when preconditions hold.

## 5.8 Spring Boot specifics

### Auto-configuration

Hundreds of `@Configuration` classes ship with conditions like:
> "If `DataSource` is on classpath AND no `DataSource` bean exists AND `spring.datasource.url` is set → register `HikariDataSource`."

Override by defining your own bean — `@ConditionalOnMissingBean` makes Boot back off.

### `application.yml`

```yaml
spring:
  datasource:
    url: jdbc:postgres://localhost/app
    username: app
    password: ${DB_PW}     # env var resolution
server:
  port: 8080
```

## 5.9 Testing

**Most unit tests don't need Spring** thanks to constructor injection:

```java
class OrderServiceTest {
    @Test void chargesCustomer() {
        OrderService svc = new OrderService(new InMemoryOrderRepo(), mock(PaymentGateway.class));
        svc.placeOrder(testOrder);
        verify(pay).charge(any());
    }
}
```

When you do need Spring:

| Annotation | Loads | Use for |
|---|---|---|
| `@SpringBootTest` | Whole app context | E2E integration |
| `@WebMvcTest(MyController.class)` | MVC + controller | Controllers without DB |
| `@DataJpaTest` | JPA + in-memory DB | Repository queries |
| `@JsonTest` | Jackson | Serialization |

`@MockitoBean` (modern; older `@MockBean` deprecated) for swapping in mocks inside the context.

## 5.10 AOP, proxies, and gotchas

### Proxy mechanisms

| | JDK dynamic proxy | CGLIB |
|---|---|---|
| Implements | Interfaces only | Subclasses concrete class |
| Requires | Bean implements interface | Class non-final, methods non-final |
| Default in modern Boot | — | ✅ |

### `@Transactional` self-invocation trap

```java
@Service
class OrderService {
    @Transactional
    public void placeOrder(Order o) { internalSave(o); }

    @Transactional(propagation = REQUIRES_NEW)
    public void internalSave(Order o) { ... }   // ❌ NOT in new transaction
}
```

Why: `placeOrder` is called via the proxy (transaction starts). Inside, `internalSave` is called via `this.internalSave(...)` — bypassing the proxy. `@Transactional` on `internalSave` is invisible.

Same trap applies to `@Async`, `@Cacheable`, `@Retryable`.

**Fixes (best to worst):**
1. **Extract to separate bean** — clean, no proxy gymnastics
2. **AspectJ load-time/compile-time weaving** — bytecode-level interception, no proxy
3. **Self-injection** (`@Autowired private OrderService self;` then `self.method()`) — works but ugly

### `@Transactional` other gotchas

- Only works on **public** methods (proxy can't intercept private/protected; `@Transactional` on private silently does nothing)
- Checked exceptions don't trigger rollback by default; only unchecked. Override with `rollbackFor = Exception.class`

## 5.11 Concurrency intersection

### Singleton beans are shared by ALL threads

Default scope is singleton. Stateless services with only `final` injected deps are safe by construction.

❌ Singleton bean with mutable instance state → concurrent corruption (or worse, semantically wrong shared state across requests).

When you see shared mutable state on a singleton, ask **why** before making it thread-safe. The fix often isn't synchronization — it's removing the sharing.

### `@Async` and Spring's threading model

```java
@EnableAsync
@Configuration
class AsyncConfig {
    @Bean(name = "taskExecutor")
    Executor taskExecutor() {
        ThreadPoolTaskExecutor x = new ThreadPoolTaskExecutor();
        x.setCorePoolSize(8);
        x.setMaxPoolSize(8);
        x.setQueueCapacity(1000);
        x.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
        x.initialize();
        return x;
    }
}
```

Spring's default async executor is `SimpleAsyncTaskExecutor` — creates a new thread per call, pathological under load. Always override.

`@Async` has the same self-invocation bug as `@Transactional`.

### Virtual threads in Spring Boot 3.2+

```yaml
spring:
  threads:
    virtual:
      enabled: true
```

Tomcat request handling and `@Async` switch to virtual threads. Audit singleton beans for `synchronized` methods that do I/O — they pin carriers (until JDK 24+).

## 5.12 Interview trap inventory

1. **`@Transactional` self-invocation** — highest-frequency Spring gotcha
2. **Circular dependency** — constructor injection fails at startup (good!); setter/field injection silently resolves but is a smell
3. **`@Component` vs `@Bean`** — `@Component` for classes you own; `@Bean` for code you don't
4. **`BeanFactory` vs `ApplicationContext`** — context is a richer factory; rarely use raw `BeanFactory`
5. **Field vs constructor injection** — constructor wins for all Module 4 reasons
6. **Eager vs lazy init** — singletons are eager by default; `@Lazy` defers
7. **`@Primary` vs `@Qualifier`** — primary sets default; qualifier picks at injection site
8. **Bean scope mismatch** — see Part 5.5
9. **JDK proxy vs CGLIB** — modern Boot defaults to CGLIB
10. **Why `@SpringBootApplication`?** — bundles `@Configuration` + `@EnableAutoConfiguration` + `@ComponentScan`

## 5.13 Exercise — URL Shortener review (key learnings)

Common mistakes seen in implementation:
- `@Controller` instead of `@RestController` for REST endpoints
- Class named `RestController` shadowing the annotation
- Hidden compile errors (undeclared variable, invalid class declaration)
- Missing exercise pieces: two ID generators with `@Qualifier`, `@Profile`-based repo selection, both unit + integration tests
- "Random" ID generator using `AtomicLong` (sequential, not random) — naming mismatch
- Interface named after one variant (`RandomIdGenerator`) instead of abstraction (`IdGenerator`)
- `String getUrl(String code)` returning nullable instead of `Optional<String>`
- Open-redirect vulnerability — no URL validation
- Two state fields (tokens + lastTime) in separate maps — not atomically updatable

Two patterns to internalize:

1. **Interface name is the abstraction; class name is the variant.** `IdGenerator` interface, `Random*` and `Sequential*` impls.
2. **`Optional` at every boundary where "not found" is valid.**


---

<a id="rabbit-holes"></a>
# Rabbit Holes

## RH1 — Reading Thread Dumps

### Capture

| Method | When |
|---|---|
| `jstack <pid>` | Standard |
| `kill -3 <pid>` (Unix) | Sends SIGQUIT; JVM dumps to stdout |
| `jcmd <pid> Thread.print -l` | Modern, recommended; `-l` includes ownable synchronizers |
| `Thread.getAllStackTraces()` | Programmatic |

**Best practice:** capture **3 dumps, 5 seconds apart**. A single dump is a snapshot — three reveal motion.

### Anatomy of one entry

```
"http-nio-8080-exec-7" #42 daemon prio=5 tid=0x... nid=0x...
   java.lang.Thread.State: TIMED_WAITING (parking)
        at jdk.internal.misc.Unsafe.park(Native Method)
        - parking to wait for  <0x000abc> (a ...AbstractQueuedSynchronizer$ConditionObject)
        at java.base/.../LockSupport.parkNanos(...)
        at java.base/.../LinkedBlockingQueue.poll(...)
   Locked ownable synchronizers:
        - None
```

Decode: name + ID, daemon flag, state, top frame, monitor address, locks held.

Critical: **RUNNABLE doesn't mean actively running.** Threads in `socketRead0` are RUNNABLE — Java treats kernel-blocked I/O as runnable.

### Five canonical patterns

**A. Hot lock contention** — N threads BLOCKED on `<0x...abc>`, one RUNNABLE owning it. Look at what the owner is doing under the lock.

**B. Lost/missing signal** — many threads `WAITING (parking)` on the same Condition, no one to notify. Producer crashed or signal was lost.

**C. Deadlock** — JVM auto-prints at the bottom: "Found one Java-level deadlock: ..."

**D. Pool exhaustion** — all workers RUNNABLE in same downstream call (e.g., `socketRead0` on the same DB), queue full, submitters BLOCKED.

**E. GC thrashing** — GC threads doing repeated work; app threads pause.

### Tools

- **fastthread.io** — paste a dump, get visualizations and detected anti-patterns
- **jvisualvm / JMC** — live + historical
- **async-profiler** — for "RUNNABLE but doing what?" cases
- `grep "java.lang.Thread.State" dump.txt | sort | uniq -c` — quick state histogram

## RH2 — AQS Internals

`AbstractQueuedSynchronizer` is the foundation under `ReentrantLock`, `Semaphore`, `CountDownLatch`, `ReentrantReadWriteLock`, `FutureTask`, `SynchronousQueue`, `Phaser`.

### Three-part anatomy

```
AQS:
1. volatile int state    ← subclass defines meaning (CAS-updated)
2. Wait queue: head ─→ Node ─→ Node ─→ tail (parked threads, FIFO-ish)
3. LockSupport.park / unpark (suspend/resume threads)
```

**`state`** — single atomic int. Subclass interprets:
- `ReentrantLock`: hold count
- `Semaphore`: available permits
- `CountDownLatch`: remaining count (0 = open)
- `ReentrantReadWriteLock`: high 16 bits = readers, low 16 bits = writers

### Subclass contract (override these)

| Method | Question |
|---|---|
| `tryAcquire(int)` | Can we grab `state` exclusively? |
| `tryRelease(int)` | Release exclusively |
| `tryAcquireShared(int)` | Can we grab in shared mode? |
| `tryReleaseShared(int)` | Release shared |
| `isHeldExclusively()` | (Optional) for `ConditionObject` |

### Build your own — `Mutex`

```java
public class Mutex implements Lock {
    private static final class Sync extends AbstractQueuedSynchronizer {
        protected boolean tryAcquire(int unused) {
            if (compareAndSetState(0, 1)) {
                setExclusiveOwnerThread(Thread.currentThread());
                return true;
            }
            return false;
        }
        protected boolean tryRelease(int unused) {
            if (getState() == 0) throw new IllegalMonitorStateException();
            setExclusiveOwnerThread(null);
            setState(0);
            return true;
        }
        protected boolean isHeldExclusively() { return getState() == 1; }
        Condition newCondition() { return new ConditionObject(); }
    }
    private final Sync sync = new Sync();
    public void lock() { sync.acquire(1); }
    public void unlock() { sync.release(1); }
    public boolean tryLock() { return sync.tryAcquire(1); }
    public Condition newCondition() { return sync.newCondition(); }
    // + lockInterruptibly, tryLock(timeout)
}
```

That's a fully-featured non-reentrant lock with timeouts, interruptibility, conditions — for the cost of overriding two methods.

### Acquire algorithm (simplified)

```java
public final void acquire(int arg) {
    if (!tryAcquire(arg)) {
        Node node = addWaiter(EXCLUSIVE);
        for (;;) {
            Node p = node.predecessor();
            if (p == head && tryAcquire(arg)) {
                head = node;
                return;
            }
            if (shouldParkAfterFailedAcquire(p, node)) {
                LockSupport.park(this);
            }
        }
    }
}
```

Key: **head is "ghost"** representing current owner, not a waiter.

### Conditions

`AQS.ConditionObject` is a separate FIFO list. `condition.await()`:
1. Adds to condition queue
2. Fully releases the lock (saves hold count)
3. Parks
4. On `signal()`, transfers from condition queue to acquire queue
5. Re-acquires the lock before returning

## RH3 — Virtual Threads (Project Loom)

### The premise

Platform thread = ~1 MB stack, ~µs context switch. Limit: thousands.
Virtual thread = JVM-implemented, few hundred bytes. Limit: millions.

Write **plain blocking code**; the JVM makes it cheap.

### Mechanism: continuations + carriers

```
ForkJoinPool (carriers, ~#cores platform threads)
   carrier-1 ←── mounting ──→ vthread-A
   carrier-2 ←── mounting ──→ vthread-B

Park heap (millions of unmounted vthreads):
   vthread-C (parked, waiting on socket read)
   ...
```

Hit a blocking call → JVM **unmounts** the vthread (saves stack as `Continuation` in heap), frees the carrier. Re-mount on (any) carrier when I/O is ready.

### API

```java
Thread.startVirtualThread(() -> doWork());

try (ExecutorService exec = Executors.newVirtualThreadPerTaskExecutor()) {
    for (Url u : urls) exec.submit(() -> http.get(u));
}

Thread vt = Thread.ofVirtual().name("worker-", 0).start(() -> ...);
```

### Pinning gotcha

A virtual thread can't unmount while:
1. Inside a `synchronized` block — pre-Java 24
2. Inside a native (JNI) frame
3. Holding certain JVM internal locks

Pinned = carrier stuck → if all carriers pin, vthread system stalls.

```java
synchronized (lock) {
    socket.read();    // ❌ PINS the carrier
}
```

Fix: use `ReentrantLock`.

```java
lock.lock();
try { socket.read(); }   // ✅ unmounts cleanly
finally { lock.unlock(); }
```

Detect: `-Djdk.tracePinnedThreads=full`. Java 24+ lifts the synchronized restriction.

### Sizing changes

**Old rule:** pool size = `cores * (1 + wait_time/cpu_time)`. Always tuned, always wrong.
**New rule:** for I/O-bound, **don't pool** — `newVirtualThreadPerTaskExecutor()`. CPU-bound: still use fixed platform pool.

### Structured concurrency (Java 21+)

```java
try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
    Subtask<User> user = scope.fork(() -> userService.find(id));
    Subtask<List<Order>> orders = scope.fork(() -> orderService.findFor(id));
    scope.join();
    scope.throwIfFailed();
    return new Profile(user.get(), orders.get());
}
```

### When to use vthreads

✅ I/O-bound web servers, batch processors, fan-out clients
✅ Code that wants to look synchronous
❌ CPU-bound (use fixed platform pool)
❌ Long-lived `synchronized` across blocking I/O (pre-Java 24)
❌ Heavy `ThreadLocal` use (millions of vthreads → memory issue; use `ScopedValue`)

## RH4 — `ConcurrentHashMap` Java 7 → 8 Evolution

### Java 7: Segments + lock striping

```
ConcurrentHashMap (Java 7)
├── Segment[0..15]  (each extends ReentrantLock + has its own table)
```

Concurrency cap = 16 (default). `put` locks one segment; reads are lock-free (volatile entries).

**Limits:** hard concurrency ceiling, two-level dereference (worse cache locality), expensive `size()`, per-segment resize.

### Java 8: Per-bucket locking + CAS

```
ConcurrentHashMap (Java 8)
└── Node[]   (single bucket array)
    ├── bucket 0: null
    ├── bucket 1: Node → Node → Node          (linked list)
    ├── bucket 2: TreeBin → red-black tree    (when chain > 8)
    ├── bucket 3: ForwardingNode               (during resize)
```

Insert algorithm:
- If bucket empty → CAS new node (lock-free fast path)
- If forwarding → help with the resize
- Else → `synchronized (bucketHead)` and walk

### Tree bins

When chain length > 8, convert to red-black tree. Bounds worst-case lookup at O(log n) — defends against hash flooding DoS.

### Concurrent resize cooperation

1. Resize-triggering thread allocates new (2x) table
2. Migrates buckets, leaves `ForwardingNode` in old buckets
3. **Other writers** that hit a `ForwardingNode` help migrate before proceeding
4. More writers = faster resize

### `size()` and `mappingCount()`

Striped counter (`CounterCell[]`, like `LongAdder`). `size()` walks cells and sums — approximate during concurrent modification. `mappingCount()` returns `long` for huge maps.

### Side-by-side

| | Java 7 | Java 8 |
|---|---|---|
| Concurrency | `concurrencyLevel` (16) | Number of buckets |
| Empty insert | Segment lock | CAS, lock-free |
| Non-empty insert | Segment lock | `synchronized` on bucket head |
| Reads | Lock-free | Lock-free |
| Resize | Per-segment, blocking | Cooperative, multi-threaded |
| Worst lookup | O(n) chain | O(log n) tree |
| `size()` | All-segments scan | Striped counter |

### What to remember

1. Java 7 = lock striping by segment
2. Java 8 = per-bucket sync + CAS + tree bins + cooperative resize
3. Reads are lock-free in both
4. Atomic compound ops (`computeIfAbsent`, `merge`) are correct usage
5. Function in `compute*` runs under bucket lock — no I/O, no callbacks

## RH5 — Virtual Threads vs Reactive (Mono/Flux)

### Reactive in 2 minutes

`Mono<T>` = 0 or 1 future T. `Flux<T>` = 0..N. Lazy — nothing happens until `subscribe`. Build pipelines of operators.

```java
Mono<User> findUser(String id) {
    return userRepo.findById(id)
        .flatMap(u -> enrichmentService.enrich(u))
        .timeout(Duration.ofSeconds(2))
        .onErrorResume(TimeoutException.class, e -> Mono.just(User.unknown(id)));
}
```

Properties: async by construction, first-class backpressure, composable. Cost: **viral** (every layer must be reactive; can't `block()` without losing benefits).

### Virtual threads in 2 minutes

Write blocking; JVM makes it cheap.

```java
User findUser(String id) {
    User user = userRepo.findById(id);
    return enrichmentService.enrich(user);
}
```

### Comparison

| Dimension | Reactive | Virtual Threads |
|---|---|---|
| **Code style** | Pipeline | Imperative blocking |
| **Stack traces** | Often useless | Full, normal |
| **Debugging** | Hard | Easy |
| **Backpressure** | First-class | Implicit (queue/executor bounds) |
| **Throughput (I/O)** | Excellent | Excellent |
| **Existing blocking libs** | Must wrap | Just call them |
| **`ThreadLocal`** | Doesn't propagate | Works (use `ScopedValue`) |
| **Color (function color problem)** | All-or-nothing virality | None |
| **Best fit** | Streaming, backpressure-critical | Request-response, fan-out |

### When reactive wins

1. True streaming workloads (SSE, WebSockets, Kafka with explicit backpressure)
2. Operator-rich pipelines (`window`, `buffer`, `groupBy`, `retry`)
3. Massive event-stream processing
4. Existing reactive code (don't rewrite)

### When virtual threads win (most cases now)

1. Request-response services (the 80% case)
2. Code with existing blocking dependencies (JDBC, blocking SDKs)
3. Teams new to async — flat learning curve
4. Ops investigations — thread dumps, debugger, profiler all work

### Senior take

**Pre-Loom**, reactive was the only sane way to build high-concurrency Java. **Post-Loom**, virtual threads on Spring MVC match WebFlux throughput with vastly simpler code.

Reasonable team policy in 2025:
- **Default:** Spring MVC + virtual threads
- **Reactive:** when you have streaming semantics, complex backpressure, or extensive existing reactive infra
- **Hybrid:** call reactive APIs from vthread code via `block()` — now safe (blocking is cheap)

### Soundbite

> "Reactive solved 'too many blocking threads is too expensive.' Virtual threads attack the same problem from the other side — make blocking cheap. For typical request/response services, virtual threads now win on simplicity, debuggability, and integration with blocking libraries, with comparable performance. Reactive remains right when you have genuine streaming semantics or first-class backpressure needs."

---

<a id="enum-singleton"></a>
# Bonus — The Enum Singleton

```java
public enum StatsCache {
    INSTANCE;
    private final ConcurrentHashMap<String, Long> counts = new ConcurrentHashMap<>();
    private final LongAdder total = new LongAdder();
    public void record(String key) { counts.merge(key, 1L, Long::sum); total.increment(); }
}
```

### Why it works

Compiler generates roughly:
```java
public final class StatsCache extends Enum<StatsCache> {
    public static final StatsCache INSTANCE = new StatsCache(...);
    // ...
}
```

`INSTANCE` initialized in `<clinit>` — same JVM guarantees as the holder idiom: once-only, thread-safe, lazy on class first reference.

### Four unique advantages

1. **Reflection-safe** — JVM special-cases `Constructor.newInstance` for enums:
   ```java
   if ((clazz.getModifiers() & Modifier.ENUM) != 0)
       throw new IllegalArgumentException("Cannot reflectively create enum objects");
   ```
2. **Serialization-safe** — `ObjectInputStream` stores only the name and looks up via `valueOf`. Standard singletons leak via deserialization unless you override `readResolve`.
3. **Clone-safe** — `Enum.clone()` is `final` and throws `CloneNotSupportedException`.
4. **Concise** — three lines.

### Drawbacks

- ❌ Can't extend a class (already extends `java.lang.Enum`)
- ❌ Constructor can't take runtime parameters
- ❌ Can't be generic
- ❌ Awkward to mock in tests (static state)

### When to use

Use enum when: fixed config, implements 0+ interfaces but doesn't extend a class, might be serialized, want minimum implementation.

Use Holder idiom when: need to extend a class, runtime ctor args, need generics.

Use DI when: building anything beyond a small app — singletons hide dependencies and break testability.

### Strategy enum (bonus pattern)

```java
public enum Operation {
    PLUS  { public int apply(int a, int b) { return a + b; } },
    MINUS { public int apply(int a, int b) { return a - b; } };
    public abstract int apply(int a, int b);
}
```

Each constant is its own subclass with its own implementation. Clean replacement for switch statements.


---

<a id="mock-1"></a>
# Mock Interview — Round 1: Rapid-fire Fundamentals

8 questions, 1-3 sentence answers expected, mixed difficulty.

## Q1 — `volatile` vs `synchronized`. Give one situation where you'd use each.

**Your answer:** volatile ensures writes are done when read by different thread; synchronized = one thread at a time in a critical section.

**Grade: ⚠️**

The substance is right but the framing is shallow. A senior answer separates **three concerns**:
- **Atomicity** — synchronized gives it; volatile does not
- **Visibility** — both give it
- **Ordering** — both prevent reorderings

Then ground in examples:
- `volatile`: status flags, one-shot publication, single-variable independent reads
- `synchronized`: compound actions, multi-field invariants, anything where two ops must appear as one

Miss: didn't mention atomicity, the cleanest distinction.

## Q2 — Why is `volatile counter++` broken?

**Your answer:** Both threads can read 1, both update to 2, both write 2. Read/write isn't atomic.

**Grade: ✅**

Perfect. Senior addition: "compound read-modify-write operation." Volatile guarantees individual reads/writes are atomic and visible — does not bind them as a transaction. That's why `AtomicInteger.incrementAndGet()` exists (CAS-based).

## Q3 — Thread dump: 50 BLOCKED on `<0x000abc>`, 1 RUNNABLE in `Cache.put`. One-line diagnosis?

**Your answer:** Hot lock contention; that one thread holds the monitor.

**Grade: ✅**

Right. Senior addition: "I'd look at what `Cache.put` is doing — almost certainly I/O, callback, or expensive computation under the lock. The fix is shrinking the critical section, switching to `ConcurrentHashMap`, or `ReadWriteLock` if reads dominate."

## Q4 — Caught `InterruptedException`, can't rethrow. Exact correct handling?

**Your answer:** Call `Thread.currentThread().interrupt()`.

**Grade: ✅**

Exactly right. Senior addition (the *why*): "InterruptedException's contract is that it clears the flag when thrown. If I can't propagate, restoring the flag preserves the cancellation signal so callers up the stack can still see and react. Silently swallowing destroys the protocol."

## Q5 — Why constructor over field injection? At least two reasons.

**Your answer:** Constructor ensures full load + cycle detection at compile time; field-injected deps may not be documented.

**Grade: ⚠️**

Right ideas, fuzzy framing. Cycles are caught at **startup**, not compile time (Spring throws `BeanCurrentlyInCreationException`). The full senior answer hits four reasons:

1. **Immutability** — `final` fields, automatic thread-safety
2. **Testability without framework** — instantiate with `new` + mocks; no reflection, no Spring boot
3. **Fail-fast** — missing dep = startup failure, not 3am NPE
4. **Design pressure** — 12-arg constructor reveals "this class does too much"

## Q6 — Singleton `@Service` with `private List<Item> items = new ArrayList<>()` mutated by every request. What happens; how to fix without breaking singleton scope?

**Your answer:** Use a thread-safe (concurrent) list since `ArrayList` isn't thread-safe.

**Grade: ❌**

This is the one to learn from. **Right answer to the wrong question.**

Yes, swapping to `CopyOnWriteArrayList` makes operations thread-safe. But the deeper question is: **should this state be shared across requests at all?** Almost certainly not — you've made every user's items pile up in the same list. A thread-safe shared list is still semantically wrong for per-request data.

A senior interviewer is testing whether you separate **thread safety** from **state ownership**. The right diagnosis:

> "Step one: is this state actually meant to be shared across all requests? If per-request data, the bug isn't synchronization — it's that state lives on the singleton at all. Move it to a method-local variable, pass through method args, or make it request-scoped. If it really is global state (e.g., a leaderboard), then yes I'd use `CopyOnWriteArrayList` and document the synchronization policy."

When you see shared mutable state on a singleton, ask **why** before making it thread-safe.

## Q7 — `@Transactional` on private method silently doesn't work. Why in mechanism terms?

**Your answer:** Spring AOP only sees public methods; private is ignored. Want more detail.

**Grade: ✅**

Right + good intellectual honesty. The mechanism:

Spring wraps your bean in a **proxy** at startup — JDK dynamic proxy (if class implements an interface) or CGLIB subclass (otherwise). The proxy intercepts method calls.

- **JDK proxy** can only intercept **interface methods** — interfaces don't have private methods, so private is invisible
- **CGLIB subclass** can override only methods the subclass can see — `private` methods are not virtual, can't be overridden

Same mechanism breaks self-invocation: `this.someMethod()` bypasses the proxy because `this` is the *real* object, not the proxy.

## Q8 — Spring Boot 3.2 with virtual threads on. `@Service` has a `synchronized` method that does HTTP call. What happens to throughput?

**Your answer:** Thread can't be unstaged from context during synchronized; loses virtual thread capability; throughput suffers.

**Grade: ✅**

Excellent — got the conceptual mechanism without using the term. The vocabulary is **pinning**:

- Virtual thread normally **unmounts** from carrier during blocking I/O
- Inside `synchronized`, JVM **cannot unmount** — vthread is *pinned* to its carrier
- Carrier sits idle. If enough vthreads pin, all carriers stall → request handling halts globally

Detect: `-Djdk.tracePinnedThreads=full`. Fix: replace `synchronized` with `ReentrantLock`. Java 24+ lifts the restriction.

## Round 1 Summary

| Q | Topic | Grade |
|---|---|---|
| 1 | volatile vs synchronized | ⚠️ — substance right, framing shallow |
| 2 | volatile counter++ | ✅ |
| 3 | thread dump diagnosis | ✅ |
| 4 | InterruptedException | ✅ |
| 5 | constructor injection | ⚠️ — missed immutability + testability |
| 6 | singleton mutable list | ❌ — answered "thread safe" instead of "should it be shared?" |
| 7 | @Transactional private | ✅ |
| 8 | vthread pinning | ✅ |

**Pattern:** strong on mechanism (Q2, Q4, Q7, Q8). Weaker on multi-dimensional answers (Q1, Q5) and on stepping back from the literal question (Q6).

---

<a id="mock-2"></a>
# Mock Interview — Round 2: Design & Trade-offs

4 questions, paragraph-level answers expected.

## Q9 — Walk me through Spring `ApplicationContext` boot sequence

**Your answer:**
- Find all classes/bean definitions and dependencies
- Construct dependency graph
- Init singleton beans starting from leaf nodes
- Each bean has proxy definition for annotations and aspects
- Run CommandRunner/ApplicationRunner

**Grade: ⚠️**

Skeleton is right but you skipped two consequential phases and got one term slightly wrong. Full sequence:

1. **Scan + parse** — collect into `BeanDefinition`s. Recipes only.
2. **`BeanFactoryPostProcessor`s run** — *mutate definitions* before instantiation. Property placeholder resolution, `@ConfigurationProperties` population, conditional bean filtering. ← **Missed entirely.**
3. **Topological instantiation of singletons** — leaf-up. ✅
4. **Dependency injection** — constructor args, then setters/fields.
5. **`BeanPostProcessor.postProcessBeforeInitialization`** — `@PostConstruct` runs; **AOP proxies actually wrap the bean here** (via `AnnotationAwareAspectJAutoProxyCreator`). Subtle but interview-relevant: proxies are *applied by* post-processors, not "definitions associated with the bean."
6. **`InitializingBean.afterPropertiesSet()`** + custom init methods.
7. **`postProcessAfterInitialization`** — second hook.
8. **Context refresh complete event.**
9. **`ApplicationRunner`/`CommandLineRunner`.** ✅
10. **Web container starts accepting traffic.**

The two missed phases (BFPP, BPP) explain how Spring features actually work — property resolution, AOP, `@PostConstruct`, `@Async` — all hooked through these.

## Q10 — Thread-safe in-memory rate limiter as Spring `@Service`. Sketch constructor + fields. Defend synchronization choice.

**Your answer:** `ConcurrentHashMap<String, LongAdder>` for tokens; `ConcurrentHashMap<String, Long>` for lastRequestTime; synchronized block to compute time delta; `LongAdder.compute(...)` ...

**Grade: ❌**

Most important to learn from. Right intent (token bucket with lazy refill), several technical errors and one design flaw.

**Errors:**
1. **`LongAdder.compute(...)` doesn't exist** — confused with `ConcurrentHashMap.compute`. LongAdder has `add`, `increment`, `decrement`, `sum`, `reset`.
2. **`LongAdder` is wrong primitive** — optimized for many writers doing `++` on single counter. For per-user RMW, use `AtomicLong` or immutable bucket record.
3. **Two separate maps = no atomicity across them** — token update and lastRefillTime update aren't atomic together. The two pieces of state must live in **one entry**.
4. **No initialization story** for first-time users.
5. **Memory leak** — users who never return accumulate map entries forever.
6. **Unspecified lock target** — `synchronized (this)` would serialize everything globally.

**Clean answer:**

```java
@Service
public class RateLimiter {
    private final int capacity;
    private final Duration window;
    private final Clock clock;
    private final ConcurrentHashMap<String, Bucket> buckets = new ConcurrentHashMap<>();

    private record Bucket(double tokens, long lastRefillMillis) {}

    public RateLimiter(@Value("${rate.capacity:60}") int capacity,
                       @Value("${rate.windowSeconds:60}") int windowSeconds,
                       Clock clock) {
        this.capacity = capacity;
        this.window = Duration.ofSeconds(windowSeconds);
        this.clock = clock;
    }

    public boolean allow(String userId) {
        long now = clock.millis();
        long[] outcome = new long[]{0};
        buckets.compute(userId, (k, b) -> {
            if (b == null) { outcome[0] = 1; return new Bucket(capacity - 1, now); }
            double refillRate = (double) capacity / window.toMillis();
            double refilled = Math.min(capacity, b.tokens + (now - b.lastRefillMillis) * refillRate);
            if (refilled >= 1) { outcome[0] = 1; return new Bucket(refilled - 1, now); }
            return new Bucket(refilled, now);
        });
        return outcome[0] == 1;
    }
}
```

Why right:
- One `Bucket` record holds tokens + lastRefill together → atomic update via `compute`
- `ConcurrentHashMap.compute` locks only the affected bucket (per-bucket sync)
- Immutable `Bucket` record → no mutation hazards
- `Clock` injected → testable
- Configurable via `@Value` / `@ConfigurationProperties`

Senior bonuses:
- "If this needs to span instances, this fails — same user hitting two pods isn't rate-limited together. Move to Redis with `INCR`+`EXPIRE` or sliding-window Lua."
- "I'd add eviction (Caffeine with `expireAfterAccess`) so dormant users free their memory."
- "Token bucket vs sliding-window vs fixed-window — pick based on requirements."

This Q stress-tests CHM.compute (Module 2), atomicity-of-compound-state (Module 3), DI testability (Module 4), Spring conventions (Module 5).

## Q11 — `@Transactional` self-invocation: three ways to fix. Rank.

**Your answer:**
1. Autowire self → call bean.method (Worst, smell)
2. Refactor to new bean (Best)

**Grade: ⚠️**

Two of three. Missing **AspectJ load-time/compile-time weaving** — the staff-level answer.

| Rank | Approach | Verdict |
|---|---|---|
| 🥇 #1 | **Extract to a separate bean** | Best by far. Clean separation, no proxy gymnastics. |
| 🥈 #2 | **AspectJ weaving** (`@EnableLoadTimeWeaving` or compile-time) | Bypasses proxies entirely — bytecode is woven so all method calls (including self) are intercepted. Heavier setup; needs agent or build plugin. |
| 🥉 #3 | **Self-injection** | Works but ugly. Needs careful naming to avoid circular dep. Code smell. |

You correctly identified best and worst — adding "AspectJ weaving" between them shows you understand the proxy *isn't a law of physics*.

## Q12 — Singleton `@Service` legitimately needs per-request data (current user's locale). Three valid approaches; pick cleanest.

**Your answer:** "Pass as functional arguments. Apart from that not sure."

**Grade: ⚠️**

The funny thing: that's the **right** answer most of the time. You undersold yourself.

Three valid approaches:

| Rank | Approach | When |
|---|---|---|
| 🥇 #1 | **Pass per-request data as method args** | Almost always. Cleanest, most testable, zero magic. The data flow is visible in the call signature. |
| 🥈 #2 | **Inject request-scoped bean with `proxyMode = TARGET_CLASS`** | When per-request data is widely needed and method-arg passing creates parameter pollution. |
| 🥉 #3 | **Inject `ObjectProvider<T>` and call `getObject()`** | Same use case as #2 but explicit — lookup happens at the call site. |

Honorable mentions:
- **`RequestContextHolder`** (Spring's `ThreadLocal` for current request) — works but service-locator smell. Acceptable for cross-cutting concerns.
- **`ScopedValue`** (Java 21+) — modern thread-local replacement; the right answer once virtual threads are the norm.

**Why method args is the senior pick:** the singleton/request scope mismatch is a *design* problem disguised as a *framework* problem. Method arguments make the dependency explicit. Scoped proxies and `ThreadLocal` are escape hatches. Senior engineering is often about *not* reaching for magic.

You had #1 (the best answer). Lesson: when you've found the simple answer, **trust it** — interviewers rate "passes data as args" higher than "uses three Spring features for the same result."

## Round 2 Summary

| Q | Topic | Grade |
|---|---|---|
| 9 | Boot sequence | ⚠️ — missed BFPP and BPP phases |
| 10 | Rate limiter | ❌ — wrong primitive (LongAdder), state split across maps |
| 11 | @Transactional self-invocation | ⚠️ — missed AspectJ |
| 12 | Singleton + request data | ⚠️ — had best answer, didn't trust it |

**Overall pattern across both rounds:**

✅ **Strong:** mechanism explanations once you know the topic.
⚠️ **Medium:** multi-dimensional answers and ranking trade-offs.
❌ **Weakness:** design questions where you build something from scratch.

That's a normal mid-level profile. The fix isn't more memorization — it's **structured practice on design questions**. They reward different skills: composing primitives under uncertainty, often with the best answer involving stepping back from the problem.

## Pre-interview suggestions

1. **Re-do Q10 from scratch tomorrow without looking** — design questions only stick after you've actually written the code.
2. **Solve one of these design problems cold:**
   - Thread-safe LRU cache with TTL
   - In-memory job scheduler that triggers tasks at specific times
   - Connection pool with bounded capacity, timeout, health-check
3. **Night before interview:** re-read the Module 3 diagnostic checklist + Round 1 vocabulary distinctions (atomicity vs visibility vs ordering, BLOCKED vs WAITING, JDK vs CGLIB proxy).


---

<a id="cheatsheets"></a>
# Cheatsheets

## JMM properties matrix

| Property | What it answers | Primitive |
|---|---|---|
| **Atomicity** | "Can this op be torn?" | `synchronized`, `Atomic*`, locks |
| **Visibility** | "Will another thread see my write?" | `volatile`, `synchronized`, `final` |
| **Ordering** | "Can the JVM/CPU reorder these?" | `volatile`, `synchronized` |

## Thread state vocabulary

| Dump phrase | `Thread.State` | Meaning |
|---|---|---|
| `runnable` | RUNNABLE | On CPU OR in OS-blocking syscall |
| `waiting for monitor entry` | BLOCKED | Trying to enter `synchronized` |
| `in Object.wait()` | WAITING/TIMED_WAITING | `wait()` on a monitor |
| `waiting on condition` / `parking` | WAITING/TIMED_WAITING | `LockSupport.park` |

**BLOCKED ≠ WAITING.** Blocked = contending for monitor (lock contention). Waiting = voluntary suspension (parking).

## Interrupt protocol (3 rules)

1. **If you can throw, throw `InterruptedException`** — propagate
2. **If you can't throw, restore the flag**: `Thread.currentThread().interrupt()`
3. **Never silently swallow**

## `j.u.c` quick-pick

| Need | Use |
|---|---|
| Wait for N events | `CountDownLatch` |
| Limit concurrent access to N | `Semaphore(N)` |
| Producer/consumer with backpressure | `BlockingQueue` (bounded) |
| High-contention counter | `LongAdder` |
| Read-mostly map | `ConcurrentHashMap` |
| Atomic compound update | `CHM.compute*` / `merge` |
| Coordinated barrier (reusable) | `CyclicBarrier` |
| Async fan-out + combine | `CompletableFuture` |
| Recursive divide & conquer | `ForkJoinPool` |
| Single-variable atomic update | `AtomicInteger`/`AtomicReference` |

## Pool sizing

- I/O-bound on platform threads: `cores * (1 + wait_time/cpu_time)`
- I/O-bound on virtual threads: don't pool — `newVirtualThreadPerTaskExecutor()`
- CPU-bound: fixed pool sized to `Runtime.getRuntime().availableProcessors()`
- Always: bounded queue + sensible rejection policy (`CallerRunsPolicy` is the senior pick)

## Diagnostic checklist (apply to every concurrent class)

1. What state is shared?
2. What's the invariant?
3. What's the synchronization policy for each shared field?
4. Are compound actions atomic?
5. Do I hold a lock during alien calls?
6. Is there a lock-ordering convention?
7. Can blocked threads be cancelled?
8. What's the rejection / overload policy?
9. What's the shutdown story?
10. What does a thread dump look like under load?

## Singleton patterns ranked

| Pattern | Lazy | Thread-safe | Reflection-safe | Serialization-safe | Notes |
|---|---|---|---|---|---|
| Eager static | ❌ | ✅ | ❌ | ❌ | Simple |
| DCL | ✅ | ✅ (with `volatile`) | ❌ | ❌ | Easy to get wrong |
| **Holder idiom** | ✅ | ✅ | ❌ | ❌ | Recommended for most cases |
| **Enum** | ✅ | ✅ | ✅ | ✅ | Bloch's pick |
| DI-managed | ✅ | ✅ | n/a | n/a | The right answer at app scale |

## DI principles

- **Constructor injection > setter > field**
- **Inject interfaces, not concretes** (DIP)
- **Single composition root** — only place that knows concretes
- **Fail-fast at startup** > runtime NPE
- **Test without the framework** — that's the testability dividend

## Spring annotations cheat

| Annotation | Purpose |
|---|---|
| `@Component`/`@Service`/`@Repository` | Register as bean (Repository adds exception translation) |
| `@RestController` | `@Controller` + `@ResponseBody` |
| `@Configuration` + `@Bean` | Wire third-party code |
| `@Autowired` | Optional on single-ctor classes since Spring 4.3 |
| `@Qualifier("name")` | Disambiguate at injection site |
| `@Primary` | Default when multiple candidates |
| `@Value("${prop}")` | Property injection |
| `@ConfigurationProperties` | Type-safe grouped config |
| `@Profile("env")` | Conditional registration |
| `@ConditionalOnProperty` / `@ConditionalOnMissingBean` | Auto-config conditions |
| `@Scope("prototype")` | Non-singleton scope |
| `@PostConstruct` / `@PreDestroy` | Lifecycle hooks |
| `@Transactional` | Public methods only; self-invocation bypasses proxy |
| `@Async` | Same proxy gotcha; configure custom `TaskExecutor` |

## Spring proxy gotchas (memorize)

1. `@Transactional`/`@Async`/`@Cacheable` only intercept **external calls** — `this.method()` bypasses
2. Same annotations on **private** methods are silently ignored
3. Default modern Boot proxy is **CGLIB** (subclasses concrete class)
4. Class must be non-final; methods must be non-final (for CGLIB)
5. Self-invocation fix: extract to separate bean (best), AspectJ weaving (heavy), self-injection (smell)

## Concurrency interview soundbites

- **"Atomicity, visibility, ordering — three different concerns. Volatile gives visibility + ordering; synchronized gives all three."**
- **"`volatile counter++` is broken because it's a compound read-modify-write; volatile only protects each individual operation."**
- **"BLOCKED is contending for a monitor; WAITING is voluntary suspension via park or wait."**
- **"Open-call rule: never invoke alien code while holding a lock."**
- **"`LongAdder` beats `AtomicLong` under contention via stripe-and-sum; reads are weakly consistent but writes scale linearly with cores."**
- **"`ConcurrentHashMap.compute` is atomic per-bucket — use `merge` for counter increments, `computeIfAbsent` for memoization."**
- **"Don't memoize `Thread.sleep` as synchronization — use `CountDownLatch` against an explicit ready signal."**

## DI / Spring soundbites

- **"DI is the mechanism; DIP is the principle. Inject interfaces, not concretes."**
- **"Constructor injection: immutability, fail-fast, explicit contract, design pressure. Field injection breaks all four."**
- **"Singletons hide dependencies and break testability — DI is the antidote."**
- **"Spring is your composition root, automated. Every annotation is shorthand for a line in `main()`."**
- **"`@Transactional` self-invocation fails because `this.method()` bypasses the proxy; fix by extracting to a separate bean."**
- **"Singleton beans are shared by all threads — stateless or thread-safe state, never both."**

## Virtual threads vs reactive — the senior take

> "Reactive solved 'too many blocking threads is too expensive.' Virtual threads attack the same problem from the other side — make blocking cheap. For typical request/response services, virtual threads now win on simplicity, debuggability, and integration with blocking libraries, with comparable performance. Reactive remains right for genuine streaming semantics or first-class backpressure needs."

---

## Suggested study order

1. Module 1 → Module 2 → Module 3 (build mental model bottom-up)
2. Module 4 (palate cleanser)
3. Module 5 (Spring on top of DI principles)
4. Rabbit holes in any order — pick by interest
5. Mock interview rounds — re-attempt cold the day before interview

**Highest interview yield:** Modules 1–3 (concurrency foundation), Module 5 Section 5.10 (proxies and `@Transactional`), Rabbit Hole 1 (thread dumps), Rabbit Hole 3 (virtual threads).

---

*End of course material.*
