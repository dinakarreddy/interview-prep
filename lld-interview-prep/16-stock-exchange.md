# Problem 16 — Stock Exchange / Order Matching Engine

The most algorithmically dense LLD problem. Tests order book data structures, price-time priority matching, partial fills, single-writer-per-symbol concurrency, and the discipline to keep matching deterministic and fast.

## Interview opener

> **Interviewer:** Design a stock exchange — focus on the order matching engine.

---

## Clarification phase

> **You:** Order types — market, limit? Or also stop, stop-limit, IOC, FOK?
>
> **Interviewer:** Limit orders for the core. Mention market and the "time in force" variants.

> **You:** Matching rule — price-time priority?
>
> **Interviewer:** Yes — best price wins; ties broken by earliest time.

> **You:** Partial fills allowed?
>
> **Interviewer:** Yes. An order can be partially filled and remain on the book.

> **You:** Multiple symbols (AAPL, GOOG, etc.)?
>
> **Interviewer:** Yes. Each symbol has its own order book.

> **You:** Cancel orders?
>
> **Interviewer:** Yes — cancel before fully filled.

> **You:** Self-trade prevention — same user as buyer and seller?
>
> **Interviewer:** Yes — block self-trades.

> **You:** Real-time market data — order book snapshots, trade ticker?
>
> **Interviewer:** Yes — model how subscribers get updates.

> **You:** Throughput?
>
> **Interviewer:** Hundreds of thousands of orders per second per symbol on busy days.

> **You:** Latency?
>
> **Interviewer:** Sub-millisecond match-to-trade for high-frequency trading.

> **You:** Persistence and audit?
>
> **Interviewer:** Yes — every order and trade must be reconstructible. Discuss at the end.

> **You:** Out of scope — settlement, market-maker incentives, regulatory reporting?
>
> **Interviewer:** Yes, all out of scope. Just the matching engine.

OK — design needs: per-symbol order book with price-time priority, limit order matching, partial fills, cancellation, self-trade prevention, single-writer-per-symbol, low-latency, full audit.

---

## Refined requirements

### Functional
1. `placeOrder(order)` — submit a new order; matches against the book; remaining qty rests on the book if not fully filled.
2. `cancelOrder(orderId, userId)` — remove from book if still present.
3. `bookSnapshot(symbol)` — top-N levels per side for market data.
4. Real-time event stream: `OrderAccepted`, `Trade`, `OrderCancelled`, `OrderRejected`.

### Non-functional
- **Determinism**: same order sequence → same trades. (Critical for replay/audit.)
- **Latency**: micro-to-milliseconds per order.
- **Throughput**: 100k+ orders/sec per symbol.
- **Single-writer per symbol**: matching is inherently sequential per symbol (no two orders can interleave).

### Out of scope (mentioned)
- Market data fan-out details (multicast, FIX feed).
- Settlement, clearing, custody.
- Regulatory reporting.
- Fees / rebates.
- Cross-venue routing.

### Order types in scope
- **Limit** — buy/sell at a specific price or better.
- **Market** (mentioned) — accept best available price; fills immediately or rejected if no liquidity.
- **IOC** (Immediate-or-Cancel) — fill what you can, cancel the rest.
- **FOK** (Fill-or-Kill) — fill entirely or reject; no partials.

---

## Domain model

### Entities (have ID)
- **`Order`** — has lifecycle (NEW → PARTIAL_FILLED → FILLED / CANCELLED).
- **`Trade`** — immutable record of an executed match.
- **`OrderBook`** — aggregate root for a symbol's bids and asks.

### Value objects (immutable)
- **`Price`** — fixed-point decimal (cents/ticks, NOT `double`).
- **`Quantity`** — integer (shares are integral).

### Enums
- `Side` — `BUY`, `SELL`.
- `OrderType` — `LIMIT`, `MARKET`.
- `TimeInForce` — `GTC` (good-till-cancelled), `IOC`, `FOK`, `DAY`.
- `OrderStatus` — `NEW`, `PARTIALLY_FILLED`, `FILLED`, `CANCELLED`, `REJECTED`.

### Patterns
- **Strategy** — order types share a matching API but differ in time-in-force semantics.
- **Priority queue / sorted map** — order book per side.
- **Single-writer (event loop) per symbol** — concurrency model.
- **Observer / event bus** — publishes trades and book updates to subscribers.
- **Command** — orders are commands; cancels are commands.

---

## Class diagram

```
┌──────────────────────────────────────────┐
│              Order                       │
├──────────────────────────────────────────┤
│ -id : long                               │
│ -userId : String                         │
│ -symbol : String                         │
│ -side : Side                             │
│ -type : OrderType                        │
│ -tif : TimeInForce                       │
│ -limitPrice : long  (ticks)              │
│ -originalQty : long                      │
│ -remainingQty : long                     │
│ -status : OrderStatus                    │
│ -timestamp : long                        │
└──────────────────────────────────────────┘

┌──────────────────────────────────────────┐
│          OrderBook                       │
├──────────────────────────────────────────┤
│ -symbol : String                         │
│ -bids : TreeMap<Price, Deque<Order>>     │   (descending — best bid first)
│ -asks : TreeMap<Price, Deque<Order>>     │   (ascending — best ask first)
│ -ordersById : Map<Long, Order>           │
├──────────────────────────────────────────┤
│ +addOrder(Order) : List<Trade>           │
│ +cancelOrder(orderId) : boolean          │
│ +snapshot(depth) : BookSnapshot          │
└──────────────────────────────────────────┘

┌──────────────────────────────────────────┐
│           Trade                          │
├──────────────────────────────────────────┤
│ -id : long; -symbol                      │
│ -buyOrderId, -sellOrderId                │
│ -price : long ; -qty : long              │
│ -timestamp : long                        │
└──────────────────────────────────────────┘

┌──────────────────────────────────────────┐
│       MatchingEngine                     │
├──────────────────────────────────────────┤
│ -bookBySymbol : Map<String, SymbolThread>│
│ -eventBus : EventBus                     │
├──────────────────────────────────────────┤
│ +submit(Order) : void  (async)           │
│ +cancel(orderId, userId) : void          │
└──────────────────────────────────────────┘

  Each SymbolThread owns one OrderBook and processes events sequentially.
```

---

## Key abstractions

```java
public final class Order { /* see Domain section */ }
public final class Trade { /* immutable record */ }

public interface OrderBookListener {
    void onTrade(Trade t);
    void onOrderAccepted(Order o);
    void onOrderCancelled(Order o);
    void onOrderRejected(Order o, String reason);
    void onBookUpdate(String symbol, BookSnapshot snapshot);
}
```

Listeners are how market data fans out — every state change of the book becomes an event.

---

## Core code

### Value objects + enums

```java
public enum Side          { BUY, SELL }
public enum OrderType     { LIMIT, MARKET }
public enum TimeInForce   { GTC, IOC, FOK, DAY }
public enum OrderStatus   { NEW, PARTIALLY_FILLED, FILLED, CANCELLED, REJECTED }

/**
 * Prices in ticks (smallest currency unit, e.g., cents).
 * Quantities in whole shares.
 * Both are long for speed and avoidance of float errors.
 */
public final class Order {
    private final long id;
    private final String userId;
    private final String symbol;
    private final Side side;
    private final OrderType type;
    private final TimeInForce tif;
    private final long limitPrice;       // 0 for MARKET
    private final long originalQty;
    private long remainingQty;
    private OrderStatus status;
    private final long timestamp;        // nanos for tie-breaking

    public Order(long id, String userId, String symbol, Side side, OrderType type, TimeInForce tif,
                 long limitPrice, long qty, long timestamp) {
        this.id = id; this.userId = userId; this.symbol = symbol;
        this.side = side; this.type = type; this.tif = tif;
        this.limitPrice = limitPrice;
        this.originalQty = qty;
        this.remainingQty = qty;
        this.status = OrderStatus.NEW;
        this.timestamp = timestamp;
    }

    // accessors omitted for brevity
    public long id() { return id; }
    public String userId() { return userId; }
    public String symbol() { return symbol; }
    public Side side() { return side; }
    public OrderType type() { return type; }
    public TimeInForce tif() { return tif; }
    public long limitPrice() { return limitPrice; }
    public long originalQty() { return originalQty; }
    public long remainingQty() { return remainingQty; }
    public OrderStatus status() { return status; }
    public long timestamp() { return timestamp; }

    void fill(long qty) {
        this.remainingQty -= qty;
        this.status = (remainingQty == 0) ? OrderStatus.FILLED : OrderStatus.PARTIALLY_FILLED;
    }

    void markCancelled() { this.status = OrderStatus.CANCELLED; }
    void markRejected()  { this.status = OrderStatus.REJECTED; }
}

public record Trade(long id, String symbol, long buyOrderId, long sellOrderId,
                    String buyerId, String sellerId, long price, long qty, long timestamp) { }
```

### `OrderBook` — the matching engine core

```java
public final class OrderBook {
    private final String symbol;
    /**
     * BUY book: sorted by price descending (highest bid first).
     * Each price level holds a Deque of orders in time order (earliest first).
     */
    private final TreeMap<Long, Deque<Order>> bids = new TreeMap<>(Comparator.reverseOrder());

    /**
     * SELL book: sorted by price ascending (lowest ask first).
     */
    private final TreeMap<Long, Deque<Order>> asks = new TreeMap<>();

    /** Map for O(1) cancel-by-id. */
    private final Map<Long, Order> ordersById = new HashMap<>();

    private final IdGenerator tradeIdGen;

    public OrderBook(String symbol, IdGenerator tradeIdGen) {
        this.symbol = symbol; this.tradeIdGen = tradeIdGen;
    }

    /**
     * Process a new order: match against the opposite side, generate trades,
     * rest the remainder if applicable. Returns the trades created.
     */
    public List<Trade> addOrder(Order order) {
        if (!order.symbol().equals(symbol)) throw new IllegalArgumentException();
        var trades = new ArrayList<Trade>();

        // FOK: pre-check sufficient liquidity at acceptable prices.
        if (order.tif() == TimeInForce.FOK && !canFullyFill(order)) {
            order.markRejected();
            return trades;
        }

        // Match against opposite book
        TreeMap<Long, Deque<Order>> oppositeBook = (order.side() == Side.BUY) ? asks : bids;

        while (order.remainingQty() > 0 && !oppositeBook.isEmpty()) {
            Map.Entry<Long, Deque<Order>> bestLevel = oppositeBook.firstEntry();
            long bestPrice = bestLevel.getKey();

            // Price condition: BUY crosses if limit >= bestAsk; SELL crosses if limit <= bestBid.
            // Market orders cross any price.
            boolean crosses = order.type() == OrderType.MARKET
                || (order.side() == Side.BUY  && order.limitPrice() >= bestPrice)
                || (order.side() == Side.SELL && order.limitPrice() <= bestPrice);
            if (!crosses) break;

            Deque<Order> orders = bestLevel.getValue();
            // Match against orders at this level in time order.
            while (order.remainingQty() > 0 && !orders.isEmpty()) {
                Order resting = orders.peekFirst();

                // Self-trade prevention
                if (resting.userId().equals(order.userId())) {
                    // Policy: cancel resting (most common); could also reject incoming or skip.
                    orders.pollFirst();
                    ordersById.remove(resting.id());
                    resting.markCancelled();
                    continue;
                }

                long matchedQty = Math.min(order.remainingQty(), resting.remainingQty());
                long executionPrice = resting.limitPrice();   // Resting order's price wins (price improvement for taker).

                Trade trade = new Trade(
                    tradeIdGen.nextLong(), symbol,
                    order.side() == Side.BUY ? order.id() : resting.id(),
                    order.side() == Side.SELL ? order.id() : resting.id(),
                    order.side() == Side.BUY ? order.userId() : resting.userId(),
                    order.side() == Side.SELL ? order.userId() : resting.userId(),
                    executionPrice, matchedQty,
                    System.nanoTime()
                );
                trades.add(trade);

                order.fill(matchedQty);
                resting.fill(matchedQty);

                if (resting.remainingQty() == 0) {
                    orders.pollFirst();
                    ordersById.remove(resting.id());
                }
            }
            if (orders.isEmpty()) oppositeBook.remove(bestPrice);
        }

        // Rest the remainder?
        if (order.remainingQty() > 0) {
            if (order.type() == OrderType.MARKET || order.tif() == TimeInForce.IOC) {
                // Market orders don't rest; IOC cancels remainder.
                order.markCancelled();
            } else {
                // Limit GTC — rest on the book.
                addToBook(order);
                ordersById.put(order.id(), order);
            }
        }

        return trades;
    }

    public boolean cancelOrder(long orderId, String userId) {
        Order order = ordersById.get(orderId);
        if (order == null) return false;
        if (!order.userId().equals(userId)) throw new ForbiddenException();

        TreeMap<Long, Deque<Order>> book = (order.side() == Side.BUY) ? bids : asks;
        Deque<Order> level = book.get(order.limitPrice());
        if (level != null) {
            level.remove(order);
            if (level.isEmpty()) book.remove(order.limitPrice());
        }
        ordersById.remove(orderId);
        order.markCancelled();
        return true;
    }

    public BookSnapshot snapshot(int depth) {
        var bidLevels = new ArrayList<PriceLevel>();
        var askLevels = new ArrayList<PriceLevel>();
        bids.entrySet().stream().limit(depth).forEach(e -> bidLevels.add(toLevel(e)));
        asks.entrySet().stream().limit(depth).forEach(e -> askLevels.add(toLevel(e)));
        return new BookSnapshot(symbol, bidLevels, askLevels, System.nanoTime());
    }

    private PriceLevel toLevel(Map.Entry<Long, Deque<Order>> entry) {
        long totalQty = entry.getValue().stream().mapToLong(Order::remainingQty).sum();
        return new PriceLevel(entry.getKey(), totalQty, entry.getValue().size());
    }

    private void addToBook(Order order) {
        TreeMap<Long, Deque<Order>> book = (order.side() == Side.BUY) ? bids : asks;
        book.computeIfAbsent(order.limitPrice(), k -> new ArrayDeque<>()).addLast(order);
    }

    private boolean canFullyFill(Order order) {
        // Walk the opposite book; sum qty at acceptable prices; check it covers order.qty
        long needed = order.remainingQty();
        TreeMap<Long, Deque<Order>> oppositeBook = (order.side() == Side.BUY) ? asks : bids;
        for (var entry : oppositeBook.entrySet()) {
            long price = entry.getKey();
            boolean crosses = order.type() == OrderType.MARKET
                || (order.side() == Side.BUY  && order.limitPrice() >= price)
                || (order.side() == Side.SELL && order.limitPrice() <= price);
            if (!crosses) break;
            for (Order o : entry.getValue()) {
                // Skip self-trades when assessing fillability
                if (o.userId().equals(order.userId())) continue;
                needed -= o.remainingQty();
                if (needed <= 0) return true;
            }
        }
        return needed <= 0;
    }
}

public record PriceLevel(long price, long totalQty, int orderCount) { }
public record BookSnapshot(String symbol, List<PriceLevel> bids, List<PriceLevel> asks, long timestamp) { }
```

Several invariants worth pointing out:

1. **Best price wins**: `bids` is descending so `firstEntry()` is the highest bid; `asks` is ascending so `firstEntry()` is the lowest ask.
2. **Price-time priority**: orders at the same price are stored in a `Deque`; we match `peekFirst()` first (oldest).
3. **Resting price wins**: when matching, the trade executes at the resting order's price (price improvement for the taker).
4. **Self-trade prevention**: resting order with same userId is cancelled (one of several policies; specify in interview).
5. **Cancel uses the orders-by-id map** for O(1) lookup, then O(level-size) removal from the level's `Deque`.

### `MatchingEngine` — single-writer-per-symbol

```java
public final class MatchingEngine {
    private final Map<String, SymbolWorker> workers = new ConcurrentHashMap<>();
    private final OrderBookListener listener;
    private final IdGenerator orderIdGen;
    private final IdGenerator tradeIdGen;

    public MatchingEngine(OrderBookListener l, IdGenerator orderIds, IdGenerator tradeIds) {
        this.listener = l; this.orderIdGen = orderIds; this.tradeIdGen = tradeIds;
    }

    public void submit(Order order) {
        SymbolWorker w = workers.computeIfAbsent(order.symbol(),
            s -> new SymbolWorker(s, listener, tradeIdGen));
        w.enqueue(new SubmitCommand(order));
    }

    public void cancel(String symbol, long orderId, String userId) {
        SymbolWorker w = workers.get(symbol);
        if (w == null) return;
        w.enqueue(new CancelCommand(orderId, userId));
    }

    public void shutdown() {
        for (SymbolWorker w : workers.values()) w.shutdown();
    }

    sealed interface Command {}
    record SubmitCommand(Order order) implements Command {}
    record CancelCommand(long orderId, String userId) implements Command {}
}
```

### `SymbolWorker` — per-symbol single-threaded executor

```java
public final class SymbolWorker {
    private final String symbol;
    private final OrderBook book;
    private final OrderBookListener listener;
    private final BlockingQueue<MatchingEngine.Command> queue = new ArrayBlockingQueue<>(100_000);
    private final Thread thread;
    private volatile boolean running = true;

    public SymbolWorker(String symbol, OrderBookListener l, IdGenerator tradeIdGen) {
        this.symbol = symbol; this.listener = l;
        this.book = new OrderBook(symbol, tradeIdGen);
        this.thread = new Thread(this::loop, "matcher-" + symbol);
        this.thread.setDaemon(true);
        this.thread.start();
    }

    public void enqueue(MatchingEngine.Command cmd) {
        if (!queue.offer(cmd)) throw new BackpressureException();
    }

    private void loop() {
        while (running) {
            try {
                MatchingEngine.Command cmd = queue.poll(100, TimeUnit.MILLISECONDS);
                if (cmd == null) continue;
                process(cmd);
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
                return;
            } catch (Exception e) {
                // Don't let one bad order kill the engine
            }
        }
    }

    private void process(MatchingEngine.Command cmd) {
        if (cmd instanceof MatchingEngine.SubmitCommand sc) {
            handleSubmit(sc.order());
        } else if (cmd instanceof MatchingEngine.CancelCommand cc) {
            handleCancel(cc.orderId(), cc.userId());
        }
    }

    private void handleSubmit(Order order) {
        listener.onOrderAccepted(order);
        List<Trade> trades = book.addOrder(order);
        for (Trade t : trades) listener.onTrade(t);
        if (order.status() == OrderStatus.REJECTED) {
            listener.onOrderRejected(order, "FOK could not fill");
        } else if (order.status() == OrderStatus.CANCELLED) {
            // IOC remainder or self-trade-prevented
            listener.onOrderCancelled(order);
        }
        listener.onBookUpdate(symbol, book.snapshot(10));
    }

    private void handleCancel(long orderId, String userId) {
        // Look up the order; book.cancelOrder returns true if cancelled
        // For simplicity, the worker doesn't pre-check; the book validates.
    }

    public void shutdown() {
        running = false;
        thread.interrupt();
        try { thread.join(5000); } catch (InterruptedException ignored) {}
    }
}
```

The **single-writer-per-symbol** pattern is the entire concurrency model: each symbol has its own worker thread; orders for that symbol queue and process sequentially. No locks inside the matcher because there's only one thread inside.

This is how real exchanges (LMAX Disruptor, NASDAQ ITCH) work. Sequential matching is determinism: same order sequence → same trades.

### Putting it together

```java
OrderBookListener listener = new OrderBookListener() {
    public void onTrade(Trade t) {
        System.out.printf("TRADE %s buy=%d sell=%d %d@%d%n",
            t.symbol(), t.buyOrderId(), t.sellOrderId(), t.qty(), t.price());
    }
    public void onOrderAccepted(Order o)            { /* ... */ }
    public void onOrderCancelled(Order o)            { /* ... */ }
    public void onOrderRejected(Order o, String r)   { /* ... */ }
    public void onBookUpdate(String s, BookSnapshot snap) { /* push to market data */ }
};

MatchingEngine engine = new MatchingEngine(listener,
    new SequentialIdGenerator(), new SequentialIdGenerator());

// Resting buy at 100
engine.submit(new Order(1, "alice", "AAPL", Side.BUY, OrderType.LIMIT, TimeInForce.GTC, 100, 50, System.nanoTime()));
// Resting sell at 102
engine.submit(new Order(2, "bob", "AAPL", Side.SELL, OrderType.LIMIT, TimeInForce.GTC, 102, 30, System.nanoTime()));
// Aggressive buy at 102 — crosses; matches against the sell
engine.submit(new Order(3, "carol", "AAPL", Side.BUY, OrderType.LIMIT, TimeInForce.GTC, 102, 20, System.nanoTime()));
// → Trade: 20 @ 102 between order 3 (buyer) and order 2 (seller). Order 2 has 10 remaining.
```

---

## Concurrency strategy

### What's contested?
Per symbol — nothing, by design. Each symbol's matching is sequential.

### Model
- Per-symbol bounded queue (`ArrayBlockingQueue`).
- Single worker thread per symbol; processes one command at a time.
- No locks inside the order book.
- No conflicts between symbols; they run in parallel.

### Why not multi-threaded matching per symbol?
- Determinism is non-negotiable: replay must produce the same trades.
- Locks in the order book would slow the hot path far more than they'd help concurrency.
- The book's data structures (TreeMap, Deque) are not thread-safe; making them so adds cost.

### Throughput per symbol
- ~1M orders/sec with the simple design above (with some optimization).
- LMAX Disruptor demonstrated tens of millions/sec by replacing the `BlockingQueue` with a lock-free ring buffer.

### Cross-symbol scaling
- Sharded by symbol — each shard runs N symbol workers.
- Aggregate throughput scales linearly with shards.

### Listener invocation
- Inside the worker thread (synchronously) — guarantees order of events.
- Listeners must be fast. Heavy listeners (audit, market data fan-out) push to their own queues.

### Backpressure
- Bounded queue. On full, `submit` throws.
- Producers (gateway) implement gateway-side backpressure: reject orders, throttle clients.

---

## Edge cases / gotchas

| Case | Handling |
|---|---|
| **Cross at multiple price levels (sweep the book)** | Loop iterates levels until incoming order is filled or no more crosses. |
| **Self-trade** | Resting cancelled (most common policy). Mention alternatives. |
| **Order qty 0** | Reject at validation. |
| **Order limit price 0 (limit order)** | Reject (use MARKET if you want any price). |
| **Cancel after fully filled** | `ordersById.get` returns null → `cancelOrder` returns false. |
| **FOK with partial liquidity** | Pre-check; reject before any matching. |
| **IOC with no match at all** | Cancel; no trades. |
| **Price ticks (e.g., 0.01)** | Stored as integer cents. Validation at gateway. |
| **Negative qty (modify-by-delta)** | Out of scope; modify = cancel + new in this design. |
| **Modify (replace) order** | Cancel + new. Loses time priority. Some exchanges support modify-in-place that preserves priority for qty decrease only. |
| **Halt / circuit breakers** | Out of scope; would freeze the worker for the symbol. |
| **Stale order at book restart** | Book should be reconstructible from the order log. Daily startup reads pending orders. |

---

## Extensions / interview follow-ups

### "Add market orders."
- Already in the design. `crosses` always true for `OrderType.MARKET`.
- Doesn't rest; remainder cancelled.
- Risk: market sells at any price could hit a thin book and execute terribly. Production exchanges have "limit-up / limit-down" guards.

### "Add stop / stop-limit orders."
- A stop order activates as a market order when the last trade price crosses the stop price.
- A stop-limit becomes a limit order on activation.
- Need a separate "stop book" of inactive orders; trade events trigger activation checks.

### "Add iceberg orders (display only part of total quantity)."
- Order has `displayQty` and `totalQty`.
- Only `displayQty` visible to other traders; when filled, replenish from `totalQty`.
- Loses time priority each time displayed qty replenishes.

### "How would you replay the entire trading day?"
- Persist every order accepted, every cancel, every trade in order.
- Replay reads commands sequentially; same sequence → same trades. Deterministic.
- This is why the matcher is single-threaded per symbol.

### "What if a single symbol's volume is too high for one core?"
- Per-symbol → per-instrument; cannot scale a single symbol.
- Vertical scaling (faster CPU, better data structures, lock-free queues) is the answer.
- Real exchanges run hot symbols on dedicated hardware.

### "How do you fan out market data to thousands of subscribers?"
- Order book emits `BookSnapshot` and `Trade` events to a fast topic.
- Multicast at the network layer (UDP for ITCH-style feeds).
- Or via Kafka / event bus for slower consumers.
- Snapshots compress across multiple updates; deltas sent in between.

### "Why TreeMap over PriorityQueue?"
- PriorityQueue doesn't support efficient `remove(specific element)` — only `poll()`.
- For cancellation, you need to find a specific order at a specific price level — TreeMap + Deque allows it in O(log n) + O(level size).
- Also, snapshotting top-N levels requires sorted iteration, which TreeMap provides.

### "How do you ensure determinism across replays?"
- Single-writer per symbol → no race conditions in matching.
- Timestamps assigned at acceptance (gateway-side); used as tie-breakers.
- All inputs (orders, cancels) recorded sequentially in a log.
- Replaying the log re-creates the exact same trades.

### "What's the trade-off of TreeMap vs sorted array?"
- TreeMap: O(log n) insert/remove/lookup. Robust, simple.
- Sorted array: O(1) for top-of-book; O(n) inserts. Better when the book is shallow.
- Real engines often use a hybrid — array of price levels with a hash for direct lookup.

### "Self-trade prevention — what are the policy options?"
- **Cancel oldest** (cancel the resting): same-user resting orders disappear; new order continues to match.
- **Cancel newest** (cancel the incoming): the new order is rejected on the first self-match.
- **Cancel both**: cancel both orders.
- **Decrement and cancel**: reduce both qty by min, cancel both. Complex.
- All are legitimate; specify in interview.

---

## Scaling

### Single-symbol throughput
- Replace `ArrayBlockingQueue` with LMAX Disruptor (lock-free ring buffer): ~10× throughput.
- Use primitive types throughout (`long` for IDs, prices, quantities) to avoid GC pressure.
- Pre-allocate event objects (event sourcing-style; events are reusable slots in the ring).

### Multi-symbol
- Shard symbols across processes / machines.
- Routing layer: incoming order → hash(symbol) → owning shard.
- Each shard handles a non-overlapping subset.

### Audit trail
- Every command (order, cancel) and every trade persisted to a journal.
- Persistence is on the worker thread's hot path → use append-only writes with batching.
- Background flush + replication for durability.

### Market data fan-out
- Snapshot delta scheme: full snapshot every N seconds; deltas in between.
- Multicast for low-latency consumers (HFT).
- Kafka for retail / regulatory consumers.

### Disaster recovery
- Replicated state machine: every input goes to N replicas; primary processes; replicas are warm standbys.
- On primary failure: failover to replica; replicas have identical state because they replayed the same input log.

---

## Persistence tradeoffs

For a real exchange, persistence is the entire game — every order, every trade, every cancel must be reconstructible. Audit and regulatory requirements dictate it.

### What persistent data exists
- **Order log**: every accepted order. Immutable.
- **Cancel log**: every cancel command.
- **Trade log**: every executed trade. Immutable.
- **Order book snapshot**: periodic; for fast restart.
- **Market data feed**: published snapshots and deltas (downstream).

### Option 1 — Append-only file journal (per symbol)
Each symbol has its own journal: a sequence of binary records (OrderEvent, CancelEvent, TradeEvent).
- Worker writes synchronously to the journal before processing (or via batched group commit).
- On restart: read journal → replay → reconstructed book.
- **Pros**: simplest; deterministic replay; minimal overhead.
- **Cons**: long replay on cold restart (mitigated by snapshots).
- **When to pick**: standard for low-latency exchanges. LMAX uses this style.

### Option 2 — Event-sourced + Kafka
- Every command published to Kafka topic per symbol.
- Matcher consumes from Kafka; processes; emits trades to another topic.
- Cold start: re-read from Kafka offset 0.
- **Pros**: durable; replicated; replayable; downstream consumers (analytics, audit, billing) all read the same stream.
- **Cons**: Kafka introduces latency (1ms+) on the hot path; some shops keep matcher in-memory and write to Kafka *after* matching (acceptable trade-off because the matcher is deterministic given the same inputs).
- **When to pick**: cloud-native exchanges; non-HFT use cases.

### Option 3 — Relational DB (PostgreSQL)
- Tables for orders, cancels, trades.
- Sync write per order/trade.
- **Pros**: queryable; familiar; secondary indexes for "all orders by user X."
- **Cons**: too slow for HFT-style throughput; usable for ~1k orders/sec.
- **When to pick**: lower-volume venues; specifically retail brokers internally.

### Option 4 — Hybrid (recommended at scale)
- **Append-only journal** for the hot path (per-symbol disk write or NVMe).
- **Kafka** as the durable broadcast (after matcher emits).
- **PostgreSQL** for end-of-day snapshot, querying, audit reports.
- **In-memory** order book; rebuilt on startup from journal.

### Snapshots
- Every N orders (or periodically), worker writes the current order book to disk.
- Cold start: read latest snapshot + replay journal entries since.
- Reduces replay time from "all of today" to "last few seconds."

### Concurrency under persistence
- The worker writes the journal entry **before** mutating the book. If write fails, reject the order; never have an in-memory state without a persisted record.
- Group commit: accumulate N entries; sync flush to disk. Trades latency for throughput.
- Replication: tail the journal to N replicas synchronously. On failure, fail over.

### Recommendation
- **Default**: per-symbol append-only journal + periodic snapshots + downstream Kafka for audit/analytics.
- **At HFT scale**: same, with the journal on NVMe + LMAX Disruptor + multicast for market data.
- **For interview**: emphasize the determinism + append-only + replayable model. That's the senior signal.

---

## Talking points for the interview

- "Each symbol is its own state machine, processed by a single thread. That's the determinism guarantee — same input sequence, same trades, no matter when you replay."
- "Order book is a TreeMap of price → Deque of orders. Best price at the front; time order at the level. Cancel uses an orders-by-id map for O(1) lookup."
- "Resting price wins on match — gives the taker price improvement. This is the standard."
- "Self-trade prevention: in this design, resting order cancelled. Other policies exist; specify."
- "Concurrency model: per-symbol single-writer + bounded queue. No locks in the matcher itself."
- "FOK requires a pre-pass to check fillability before any state change."
- "Listeners are invoked synchronously inside the worker; for heavy work, push to another queue. Don't block the matcher."
- "Persistence: append-only journal per symbol. Replay = re-execute commands. Snapshots reduce cold-start time."
- "Determinism is the most important property. Any concurrency that breaks it is wrong, no matter the throughput gain."

---

## Summary

Patterns load-bearing: **single-writer per symbol** (concurrency model), **TreeMap + Deque** (order book structure), **command pattern** (orders/cancels are commands), **append-only journal** (persistence + replay).

The mental model:
1. Per-symbol queue + single worker.
2. Order arrives → match against opposite side at best prices, time order within level.
3. Trades fan out to listeners.
4. Remainder rests (GTC), or cancels (IOC), or rejects (FOK).
5. Journal everything; snapshot periodically.

The senior signals:
- Recognizing **single-writer-per-symbol** as the concurrency model (not "lock the book").
- Reaching for **TreeMap of Deques** for the data structure (not a single PriorityQueue).
- Specifying **price-time priority** explicitly.
- Mentioning **determinism / replay** as a design constraint, not an afterthought.
- Self-trade prevention as a policy decision.
- Acknowledging that this is one of those rare problems where vertical scaling (LMAX-style ring buffers, mechanical sympathy with the CPU) is the right answer, not horizontal.
