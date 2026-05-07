# Problem 2 — Vending Machine

The canonical State pattern interview problem. The State pattern is what makes the design clean; everything else is supporting cast.

## Interview opener

> **Interviewer:** Design a vending machine.

---

## Clarification phase

> **You:** Quick scoping questions. The user inserts coins, picks an item, gets the item plus change?
>
> **Interviewer:** Right.

> **You:** Single user at a time, or could it be a connected machine taking remote orders simultaneously?
>
> **Interviewer:** Single user — it's a physical machine.

> **You:** What payment methods? Just cash, or card / mobile?
>
> **Interviewer:** Cash for the core; mention how you'd add card.

> **You:** Should the machine support a refund button — return inserted money before completing a purchase?
>
> **Interviewer:** Yes.

> **You:** What happens if the machine can't make exact change?
>
> **Interviewer:** Notify the user; they should be able to refund or pick a different item.

> **You:** Operator/maintenance flow — restock, collect cash?
>
> **Interviewer:** Yes — the machine should have a maintenance mode.

> **You:** Multiple slots with different items at different prices? Stock count per slot?
>
> **Interviewer:** Yes, items by code (A1, B2, etc.), each with a price and stock count.

> **You:** Single currency?
>
> **Interviewer:** Yes.

> **You:** Power-off persistence? Should state survive a power cycle?
>
> **Interviewer:** Mention how you'd add it but don't implement it.

Got it. The state machine is going to be central — IDLE, money inserted, dispensing, maintenance — and operations have different semantics in each state.

---

## Refined requirements

### Functional
1. `insertMoney(int amount)` — increments inserted balance.
2. `selectItem(String code)` — attempts purchase.
3. `dispense()` — completes the purchase: release item + change.
4. `refund()` — returns inserted money.
5. Operator: `restock`, `replenishCoins`, `enterMaintenance`/`exitMaintenance`.

### Non-functional
- **Latency**: O(1) or O(D) where D = number of denominations.
- **Concurrency**: physical machine = single-threaded; treat as state machine guarded by one mutex.
- **Persistence**: design supports adding it (extension).

### Out of scope
- Card / mobile payments (extension).
- Multi-currency.
- Power-off recovery implementation (mention only).

---

## Domain model

### Entities
- **`VendingMachine`** — aggregate root.
- **`Inventory`** — owned by the machine; holds item slots and stock counts.
- **`CoinReserve`** — owned by the machine; tracks coin counts per denomination.

### Value objects
- **`Item(code, name, price)`** — immutable.
- **`Receipt(item, amountPaid, change)`**.

### Enums
- `Denomination` — `COIN_1`, `COIN_5`, `COIN_10`, `NOTE_20`, `NOTE_50`, `NOTE_100` (each with `int value`).

### State hierarchy
- `VendingState` (interface)
  - `IdleState` — no money inserted, ready.
  - `HasMoneyState` — money inserted, awaiting selection or refund.
  - `DispensingState` — selection made, releasing item + change.
  - `MaintenanceState` — operator has the machine.

The State pattern's whole point: each state owns its rules for every operation.

---

## Class diagram

```
                ┌──────────────────────────────────┐
                │        <<interface>>             │
                │        VendingState              │
                ├──────────────────────────────────┤
                │ +insertMoney(m, amount)          │
                │ +selectItem(m, code)             │
                │ +dispense(m)                     │
                │ +refund(m)                       │
                │ +name() : String                 │
                └──────────────────────────────────┘
                              △
              ┌───────────────┼───────────────┬───────────────┐
              │               │               │               │
       ┌───────────┐   ┌───────────────┐ ┌────────────────┐ ┌─────────────┐
       │ IdleState │   │ HasMoneyState │ │ DispensingState│ │MaintenanceSt│
       └───────────┘   └───────────────┘ └────────────────┘ └─────────────┘

┌──────────────────────────────────┐
│        VendingMachine            │
├──────────────────────────────────┤
│ -state : VendingState            │
│ -inventory : Inventory           │
│ -coinReserve : CoinReserve       │
│ -insertedAmount : int            │
│ -selectedItem : Item (nullable)  │
│ -changeMaker : ChangeMaker       │
├──────────────────────────────────┤
│ +insertMoney / selectItem / ...  │
│ +setState(VendingState) [pkg]    │
└──────────────────────────────────┘
       │           │              │
       ◆ 1         ◆ 1            ◆ 1
       ▼           ▼              ▼
  ┌──────────┐ ┌─────────────┐ ┌───────────────┐
  │Inventory │ │ CoinReserve │ │ ChangeMaker   │
  │          │ │             │ │ <<interface>> │
  └──────────┘ └─────────────┘ └───────────────┘
                                       △
                                       ┊
                          ┌────────────┼────────────┐
                  ┌───────────────┐         ┌────────────────┐
                  │ GreedyChange  │         │ DPChangeMaker  │
                  └───────────────┘         └────────────────┘
```

Patterns visible:
- **State** — `VendingState` hierarchy.
- **Strategy** — `ChangeMaker` (greedy vs DP).
- **Composition** — machine owns Inventory, CoinReserve, ChangeMaker.

---

## Key abstractions

```java
public interface VendingState {
    void insertMoney(VendingMachine machine, int amount);
    void selectItem(VendingMachine machine, String code);
    void dispense(VendingMachine machine);
    void refund(VendingMachine machine);
    String name();
}

public interface ChangeMaker {
    /** Optional.empty() if exact change cannot be made. */
    Optional<Map<Denomination, Integer>> make(int amount, Map<Denomination, Integer> available);
}
```

---

## Core code

### Value objects + enums

```java
public enum Denomination {
    COIN_1(1), COIN_5(5), COIN_10(10), NOTE_20(20), NOTE_50(50), NOTE_100(100);
    public final int value;
    Denomination(int v) { this.value = v; }
}

public record Item(String code, String name, int price) { }
public record Receipt(Item item, int amountPaid, Map<Denomination, Integer> change) { }
```

### `Inventory`

```java
public final class Inventory {
    private final Map<String, Item> items;
    private final Map<String, Integer> stock;

    public Inventory(Map<String, Item> items, Map<String, Integer> initialStock) {
        this.items = new HashMap<>(items);
        this.stock = new HashMap<>(initialStock);
    }

    public Item lookup(String code) {
        Item i = items.get(code);
        if (i == null) throw new ItemNotFoundException(code);
        return i;
    }

    public boolean isInStock(String code)   { return stock.getOrDefault(code, 0) > 0; }
    public void take(String code) {
        if (!isInStock(code)) throw new OutOfStockException(code);
        stock.merge(code, -1, Integer::sum);
    }
    public void restock(String code, int qty) { stock.merge(code, qty, Integer::sum); }
}
```

### `CoinReserve` and `GreedyChangeMaker`

```java
public final class CoinReserve {
    private final EnumMap<Denomination, Integer> counts = new EnumMap<>(Denomination.class);

    public CoinReserve(Map<Denomination, Integer> initial) {
        for (Denomination d : Denomination.values()) counts.put(d, 0);
        initial.forEach((d, n) -> counts.merge(d, n, Integer::sum));
    }

    public void addCoin(Denomination d, int n) { counts.merge(d, n, Integer::sum); }

    public void take(Map<Denomination, Integer> change) {
        for (var e : change.entrySet()) {
            int have = counts.getOrDefault(e.getKey(), 0);
            if (have < e.getValue()) throw new IllegalStateException("Insufficient " + e.getKey());
        }
        change.forEach((d, n) -> counts.merge(d, -n, Integer::sum));
    }

    public Map<Denomination, Integer> snapshot() { return new EnumMap<>(counts); }
}

public final class GreedyChangeMaker implements ChangeMaker {
    @Override
    public Optional<Map<Denomination, Integer>> make(int amount, Map<Denomination, Integer> available) {
        var result = new EnumMap<Denomination, Integer>(Denomination.class);
        var remaining = amount;
        var sortedDenoms = Stream.of(Denomination.values())
            .sorted(Comparator.comparingInt((Denomination d) -> d.value).reversed())
            .toList();
        for (Denomination d : sortedDenoms) {
            int canUse = Math.min(remaining / d.value, available.getOrDefault(d, 0));
            if (canUse > 0) {
                result.put(d, canUse);
                remaining -= canUse * d.value;
            }
        }
        if (remaining > 0) return Optional.empty();
        return Optional.of(result);
    }
}
```

For Indian/USD-like denomination sets, greedy is optimal. For pathological denominations it isn't — mention you'd swap in a DP-based maker.

### State implementations

```java
public final class IdleState implements VendingState {
    public void insertMoney(VendingMachine m, int amount) {
        m.addInsertedAmount(amount);
        m.setState(new HasMoneyState());
    }
    public void selectItem(VendingMachine m, String code) {
        throw new IllegalStateException("Insert money first");
    }
    public void dispense(VendingMachine m)  { throw new IllegalStateException("Insert money first"); }
    public void refund(VendingMachine m)    { /* idempotent — nothing inserted */ }
    public String name()                    { return "IDLE"; }
}

public final class HasMoneyState implements VendingState {
    public void insertMoney(VendingMachine m, int amount) {
        m.addInsertedAmount(amount);   // stay in this state
    }

    public void selectItem(VendingMachine m, String code) {
        Item item = m.inventory().lookup(code);
        if (!m.inventory().isInStock(code))         throw new OutOfStockException(code);
        if (m.insertedAmount() < item.price())      throw new InsufficientFundsException(item.price(), m.insertedAmount());

        int changeNeeded = m.insertedAmount() - item.price();
        Optional<Map<Denomination, Integer>> change =
            m.changeMaker().make(changeNeeded, m.coinReserve().snapshot());
        if (change.isEmpty()) throw new CannotMakeChangeException();   // user can refund

        m.setSelectedItem(item);
        m.setPendingChange(change.get());
        m.setState(new DispensingState());
    }

    public void dispense(VendingMachine m) { throw new IllegalStateException("Select an item first"); }

    public void refund(VendingMachine m) {
        Map<Denomination, Integer> refund = m.changeMaker()
            .make(m.insertedAmount(), m.coinReserve().snapshot())
            .orElseThrow(CannotMakeChangeException::new);
        m.coinReserve().take(refund);
        m.physicallyReturnCoins(refund);
        m.resetTransaction();
        m.setState(new IdleState());
    }

    public String name() { return "HAS_MONEY"; }
}

public final class DispensingState implements VendingState {
    public void insertMoney(VendingMachine m, int amount) {
        m.physicallyReturnCoinsForAmount(amount);   // reject mid-dispense
    }
    public void selectItem(VendingMachine m, String code) {
        throw new IllegalStateException("Already dispensing");
    }

    public void dispense(VendingMachine m) {
        Item item = m.selectedItem();
        Map<Denomination, Integer> change = m.pendingChange();

        m.inventory().take(item.code());
        m.coinReserve().take(change);
        m.physicallyReleaseItem(item);
        m.physicallyReturnCoins(change);

        m.resetTransaction();
        m.setState(new IdleState());
    }

    public void refund(VendingMachine m) {
        throw new IllegalStateException("Item already in process");
    }

    public String name() { return "DISPENSING"; }
}

public final class MaintenanceState implements VendingState {
    public void insertMoney(VendingMachine m, int a)    { throw new MachineUnavailableException(); }
    public void selectItem(VendingMachine m, String c)  { throw new MachineUnavailableException(); }
    public void dispense(VendingMachine m)              { throw new MachineUnavailableException(); }
    public void refund(VendingMachine m)                { throw new MachineUnavailableException(); }
    public String name()                                { return "MAINTENANCE"; }
}
```

### `VendingMachine` — Context

```java
public final class VendingMachine {
    private VendingState state;
    private final Inventory inventory;
    private final CoinReserve coinReserve;
    private final ChangeMaker changeMaker;
    private int insertedAmount = 0;
    private Item selectedItem;
    private Map<Denomination, Integer> pendingChange;

    public VendingMachine(Inventory inv, CoinReserve coins, ChangeMaker changeMaker) {
        this.inventory = inv;
        this.coinReserve = coins;
        this.changeMaker = changeMaker;
        this.state = new IdleState();
    }

    public synchronized void insertMoney(int a)        { state.insertMoney(this, a); }
    public synchronized void selectItem(String code)   { state.selectItem(this, code); }
    public synchronized void dispense()                { state.dispense(this); }
    public synchronized void refund()                  { state.refund(this); }
    public String currentState()                       { return state.name(); }

    // package-private — only states call these
    void setState(VendingState s)                  { this.state = s; }
    void addInsertedAmount(int a)                  { this.insertedAmount += a; }
    int  insertedAmount()                          { return insertedAmount; }
    void setSelectedItem(Item i)                   { this.selectedItem = i; }
    Item selectedItem()                            { return selectedItem; }
    void setPendingChange(Map<Denomination, Integer> c) { this.pendingChange = c; }
    Map<Denomination, Integer> pendingChange()     { return pendingChange; }
    Inventory inventory()                          { return inventory; }
    CoinReserve coinReserve()                      { return coinReserve; }
    ChangeMaker changeMaker()                      { return changeMaker; }
    void resetTransaction()                        { insertedAmount = 0; selectedItem = null; pendingChange = null; }

    // Hardware-bound — would call out to physical actuators
    void physicallyReleaseItem(Item i)             { /* ... */ }
    void physicallyReturnCoins(Map<Denomination, Integer> c) { /* ... */ }
    void physicallyReturnCoinsForAmount(int a)     { /* ... */ }

    public synchronized void enterMaintenance() {
        if (insertedAmount > 0) throw new IllegalStateException("Refund pending transaction first");
        setState(new MaintenanceState());
    }
    public synchronized void exitMaintenance()                 { setState(new IdleState()); }
    public synchronized void restock(String code, int qty)     { inventory.restock(code, qty); }
    public synchronized void replenishCoins(Denomination d, int n) { coinReserve.addCoin(d, n); }
}
```

---

## Concurrency strategy

A physical vending machine has one user at a time, but the public API may be called from multiple input sources (button presses, sensors, web interfaces if connected). Treat the machine as a state-machine guarded by **whole-machine synchronization**:

- All public methods on `VendingMachine` are `synchronized`.
- Operations are short, exclusive, and must respect state invariants.
- Cost of contention is irrelevant — there's only ever one user.

For a **connected vending machine**, the same lock applies — but you'd add a queueing layer in front (incoming requests serialize through the machine).

The `state` field is read/written under the lock, so no `volatile` needed. If we wanted lock-free reads of `currentState()` for monitoring, mark it `volatile`.

---

## Edge cases / gotchas

| Case | Handling |
|---|---|
| **Insufficient funds** | `selectItem` throws; user can insert more or refund. |
| **Out of stock** | Throw; user picks another. |
| **Cannot make exact change** | Throw; user picks cheaper item or refunds. |
| **Coin rejected mid-transaction** | Treat as never inserted (sensor concern). |
| **Refund mid-dispense** | Forbidden — DispensingState throws. |
| **Dispense fails physically (jammed)** | Hardware concern; need failed-dispensing state and operator alert. |
| **Power loss** | Persist state + insertedAmount + selectedItem to non-volatile storage; restore on boot. |
| **Operator maintenance with pending transaction** | Forbid — `enterMaintenance` throws if money inserted. |
| **Counterfeit coin detection** | Coin sensor — fail-fast at insert. |

---

## Extensions / interview follow-ups

### "Add card / mobile payments."
- Introduce `PaymentMethod` Strategy with `authorize` / `capture` / `void`.
- States become payment-aware: `HasMoneyState` becomes `PaymentAcceptedState` for card.
- For card, `selectItem` triggers an authorize hold; `dispense` captures.

### "Make the change algorithm optimal for arbitrary denominations."
- Replace `GreedyChangeMaker` with `DPChangeMaker` (DP for minimum coins).
- Strategy interface unchanged — the swap is one line at composition root.

### "Persist state across power cycles."
- Add `MachineStateSnapshot { stateName, insertedAmount, selectedItemCode, pendingChange }`.
- Persist on every state transition.
- On boot: load snapshot; map name → state; resume.
- Use a write-ahead log if you need atomicity guarantees.

### "Multiple users simultaneously (connected vending)."
- Each remote order becomes a queued command (Command pattern).
- Worker takes commands one at a time, drives the state machine.
- Customer-facing API returns a `commandId` and an async `Future<Receipt>`.

### "Add a display screen with prompts."
- Add **Observer**: states fire events on entry/exit (`onStateChanged(old, new)`).
- `Display` subscribes; renders prompts.

### "What if dispense physically fails?"
- Add `FailedDispensingState` with operator alert.
- Money refunded if possible; otherwise held until operator clears.

### "Add daily reports."
- Each `dispense` emits a transaction event (Observer).
- In-memory accumulator; persisted at midnight by a scheduler.

### "Why State pattern instead of `if (status == X)`?"
- 4 states × 4 operations = 16 cells. Spread across 4 methods, each becomes a 4-way switch.
- Adding a new state means editing every method; easy to miss one.
- State pattern: 1 new class, the compiler tells you which methods to implement.
- Each state's rules colocate — easier to read, audit, test.

---

## Scaling

Vending machines aren't a "scaling" problem in the system-design sense — there's just one of them per location. But operators of fleets (Coinstar, vending companies with thousands of machines) face system-design problems around the fleet:

### Fleet management
- Each machine reports telemetry (stock levels, sales, errors, cash on hand) over MQTT/HTTP to a central service.
- The central service is a regular distributed system — ingestion → storage → dashboards, alerts on low stock or jams.

### Predictive restocking
- ML on historical sales predicts when each machine will run out; dispatches operators efficiently.

### Remote operations
- Operator can remotely put a machine in maintenance, refund a stuck transaction, push a price change.
- Commands queued per machine; the machine consumes them on its next "heartbeat."

### Cashless and digital
- QR-code purchase: user scans, pays via app, machine receives a one-time auth token that completes the dispense.
- Removes the cash-handling subsystem entirely — software-only state machine.

### Connected → multi-tenant
- A "vending machine" might serve as an interface to a third-party order system (Amazon Locker, Wolt smart cabinets). The state machine's "items" become slots indexed by reservation IDs; "dispense" releases the reserved slot.

The LLD design above is invariant under most of these changes — `Inventory` becomes an interface (could be backed by a remote service), `PaymentMethod` is already pluggable, the state machine is unchanged.

---

## Persistence tradeoffs

A vending machine is an embedded system; persistence here has two flavors: **on-machine** (survive power cycle) and **fleet-side** (operator dashboards, sales analytics).

### What persistent data exists
- **Machine state**: current state name, inserted amount, selected item, pending change. Tiny.
- **Inventory**: per-slot stock count. Updated on every dispense + restock.
- **Coin reserve**: count per denomination. Updated on every transaction.
- **Transaction log**: append-only history of every dispense/refund/restock.
- **Health/telemetry**: errors, jams, sensor readings.

### Option 1 — Embedded SQL (SQLite) on the machine
Single file DB on the machine itself. Tables for state snapshot, inventory, coin reserve, transaction log.
- **Pros**: ACID, queryable, well-understood. Commit on every state transition. Power loss recovery: open the DB, read state, resume.
- **Cons**: SQLite has limited concurrency (one writer); not a problem here since the machine has one user.
- **When to pick**: standard for connected vending machines and similar embedded devices.

### Option 2 — Embedded key-value (LevelDB / RocksDB / LMDB) on the machine
Append-only log of state mutations; periodic snapshot.
- **Pros**: extremely fast writes; small footprint.
- **Cons**: less queryable. You'd want a separate journal-and-snapshot scheme for state recovery.
- **When to pick**: when transaction volume is so high that SQLite's WAL becomes a bottleneck (rare for vending).

### Option 3 — Server-side via heartbeat
Machine holds state in memory only; pushes telemetry every N seconds to a fleet management service. On crash, the server has the last-known-state but recent transactions might be lost.
- **Pros**: cheap on the machine; rich analytics on the server side.
- **Cons**: data loss on crash. Network outage cuts you off.
- **When to pick**: combined with on-machine persistence as a buffer — local-first, then sync.

### The fleet (server) side
For an operator running 10,000 vending machines, the server-side is a regular distributed system:
- **PostgreSQL or DynamoDB** for machine inventory, sales, configuration.
- **Time-series DB (InfluxDB / TimescaleDB / Prometheus)** for telemetry: stock level over time, dispense rate per slot, error counts.
- **Object storage (S3)** for periodic full snapshots of each machine's state.
- **Kafka** for real-time event stream from machines to backend.

### Power-loss recovery flow
1. Machine boots; reads latest snapshot from local DB.
2. Replays any post-snapshot events (if event-sourced).
3. Restores state machine to that point.
4. Publishes "I restarted at state X" to the fleet service for visibility.

If a transaction was mid-flight (money inserted, item not yet dispensed), the policy choice is: refund automatically, or leave in `HasMoneyState` and let the user complete or refund.

### Recommendation for vending machine
- **On-machine**: SQLite, with WAL mode. State persisted on every transition; full snapshot on graceful shutdown.
- **Server-side**: PostgreSQL for OLTP (inventory, sales), Prometheus/Grafana for telemetry, Kafka for the event stream.

### Concurrency under persistence
On a single machine, there's one user — no concurrency. The DB writes are serial under the existing `synchronized` lock. The trade-off is between **commit on every transition** (slower, safest) and **batch commits at end of transaction** (faster, possible data loss on crash).

For audit-critical events (purchase completed), commit synchronously. For idle state changes, batching is fine.

---

## Talking points for the interview

- "I drew the state diagram first because that's the spec — the code follows from it directly."
- "Every operation in every state has explicit semantics — including idempotent ones (refund in IDLE), illegal ones (dispense in IDLE → throw), and stay-in-state ones (insertMoney in HAS_MONEY)."
- "Transaction state (`insertedAmount`, `selectedItem`, `pendingChange`) is reset together by `resetTransaction()` — single helper to avoid forgetting one field."
- "Change-making is its own Strategy because for non-canonical denominations greedy is wrong; the interface lets us swap in DP without touching the state machine."
- "I'm using `synchronized` on the public API rather than fine-grained locks because the whole machine is a single state machine."
- "If we needed power-off recovery, this design persists cleanly: state name + a few fields. We'd write a snapshot after each transition."

---

## Summary

**State** is the load-bearing pattern; **Strategy** sits underneath for ChangeMaker; **Observer** comes in if we add a display.

The mental model: states own rules; the machine owns state; transitions happen in states by calling `m.setState(...)`. No `if (state == X)` anywhere. The state diagram on the whiteboard maps 1:1 to classes in code — the interviewer can audit transitions visually.
