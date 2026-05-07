# Problem 1 — Parking Lot

## Interview opener

> **Interviewer:** Design a parking lot.

That's the entire prompt you'll typically get. Everything else you derive through clarifying questions.

---

## Clarification phase (the dialogue)

This is the part most candidates skip and lose 30% of the interview score on. Drive it.

> **You:** A few clarifying questions before I start. Is this a single parking lot, or a multi-lot system?
>
> **Interviewer:** Single lot for now.

> **You:** What's the physical structure — multiple floors? How many spots per floor roughly?
>
> **Interviewer:** Multiple floors, each with multiple spots. Sizing isn't important — make the design support arbitrary numbers.

> **You:** Are there different vehicle types? And different spot sizes that constrain which vehicles fit?
>
> **Interviewer:** Yes. Motorcycles, cars, vans, trucks. Spot sizes are motorcycle, compact, regular, large. Smaller vehicles can use larger spots, not the other way around.

> **You:** The user flow — vehicle enters, gets a ticket, parks, then on exit pays a fee?
>
> **Interviewer:** Right. Fee depends on duration and vehicle type.

> **You:** Multiple gates? So multiple vehicles could enter or exit simultaneously?
>
> **Interviewer:** Yes — and the design should handle that correctly.

> **You:** Reservations? EV charging spots? Lost ticket flow? Operator/admin actions?
>
> **Interviewer:** Out of scope for the core design, but mention how you'd extend.

> **You:** Persistence? In-memory or do I need to think about a database?
>
> **Interviewer:** In-memory is fine for this. Mention if your design supports swapping in persistence.

> **You:** Display boards showing availability per floor / spot type?
>
> **Interviewer:** Nice-to-have. Mention how you'd add it.

> **You:** Payment methods?
>
> **Interviewer:** Pluggable; you don't need to enumerate.

Now I have enough to start.

---

## Refined requirements

### Functional
1. **Park a vehicle** — given a vehicle, allocate the smallest free spot that can fit it; issue a ticket.
2. **Unpark a vehicle** — given a ticket, calculate the fee and free the spot.
3. **Query availability** — return the count of free spots per size.

### Non-functional
- **Concurrency**: many gates, many simultaneous park/unpark; design should be contention-light. No global lot lock.
- **Latency**: O(1) park and unpark in the common case.
- **Persistence**: in-memory; abstract behind a repository so persistent backends can be swapped in.
- **Extensibility**: fee model, allocation strategy, vehicle/spot types should evolve without core edits.

### Out of scope
- Reservations, EV charging, lost ticket workflow, multi-lot system, payment processing details, persistence implementation.

### Core operations
1. `park(Vehicle) → Ticket`
2. `unpark(ticketId) → Receipt`
3. `availability() → Map<SpotSize, Integer>`

---

## Domain model

### Entities (have ID, lifecycle)
- **`ParkingLot`** — aggregate root for the system.
- **`Floor`** — owned by `ParkingLot` (composition).
- **`ParkingSpot`** — owned by `Floor` (composition). Has mutable status.
- **`Ticket`** — issued at entry, identifies a parking session.

### Value objects (immutable, equality by content)
- **`Vehicle(licensePlate, type)`** — same plate + type means same vehicle conceptually.
- **`Money(amount, currency)`** — never `double` for money.
- **`Receipt(ticketId, vehicle, entry, exit, fee)`** — produced on exit.

### Enums
- `VehicleType` — `MOTORCYCLE`, `CAR`, `VAN`, `TRUCK`.
- `SpotSize` — `MOTORCYCLE`, `COMPACT`, `REGULAR`, `LARGE`.
- `SpotStatus` — `FREE`, `OCCUPIED`, `OUT_OF_SERVICE`.
- `TicketStatus` — `ACTIVE`, `PAID`, `LOST`.

### Why these choices
- `Vehicle` is a value object — no lifecycle to track per session.
- `Spot` is an entity — its status mutates over time.
- `Ticket` is an entity — each parking session is unique by ticket ID.
- Aggregate boundaries: `ParkingLot ◆ Floor ◆ Spot` (composition all the way; destroying the lot destroys the structure). `Ticket` is its own aggregate root referencing `Spot` by ID — across-aggregate references go by ID, not by direct reference.

---

## Class diagram

```
                ┌────────────────────┐
                │    <<interface>>   │
                │   FeeStrategy      │
                ├────────────────────┤
                │ +calculate(t, exit)│
                │      : Money       │
                └────────────────────┘
                          △
                          ┊
       ┌──────────────────┼──────────────────┐
       │                  │                  │
┌──────────────┐ ┌──────────────┐ ┌──────────────────┐
│ HourlyFee    │ │ DailyCapFee  │ │ FreeFirst15Min   │
└──────────────┘ └──────────────┘ └──────────────────┘

                ┌────────────────────┐
                │    <<interface>>   │
                │  SpotAllocator     │
                ├────────────────────┤
                │ +allocate(vehicle) │
                │   : Optional<Spot> │
                └────────────────────┘
                          △
                          ┊
       ┌──────────────────┼──────────────────┐
       │                  │                  │
┌──────────────────┐  ┌────────────────────┐
│ NearestFirstAlloc│  │ EvenDistributionAlloc│
└──────────────────┘  └────────────────────┘

┌────────────────────────┐
│      ParkingLot        │
├────────────────────────┤
│ -id : String           │
│ -floors : List<Floor>  │
│ -allocator : SpotAlloc │ ◇──► SpotAllocator
│ -feeStrategy : FeeStr. │ ◇──► FeeStrategy
│ -ticketRepo : TicketRep│ ◇──► TicketRepository
│ -clock : Clock         │
├────────────────────────┤
│ +park(v) : Ticket      │
│ +unpark(tId) : Receipt │
│ +availability() : Map  │
└────────────────────────┘
         │ ◆ 1..*
         ▼
┌────────────────────┐
│       Floor        │
├────────────────────┤
│ -id : String       │
│ -spots : List<Spot>│
└────────────────────┘
         │ ◆ 1..*
         ▼
┌──────────────────────────┐
│      ParkingSpot         │
├──────────────────────────┤
│ -id : String             │
│ -size : SpotSize         │
│ -status : AtomicRef      │
│ -currentTicketId         │
├──────────────────────────┤
│ +tryReserve(tId) : bool  │
│ +release() : void        │
│ +canFit(VehicleType):bool│
└──────────────────────────┘

┌──────────────────────────┐    ┌──────────────────────┐
│         Ticket           │    │       Receipt        │
├──────────────────────────┤    ├──────────────────────┤
│ -id; -vehicle            │    │ -ticketId            │
│ -spotId; -entryTime      │    │ -vehicle             │
│ -status : TicketStatus   │    │ -entry; -exit        │
└──────────────────────────┘    │ -fee : Money         │
                                └──────────────────────┘

   record Vehicle(plate, type)         (value object)
   record Money(amount, currency)      (value object)
```

Patterns visible:
- **Strategy** — `FeeStrategy`, `SpotAllocator`.
- **Composition** — Lot ◆ Floor ◆ Spot.
- **Repository** — `TicketRepository` interface.
- **Dependency injection** of `Clock`, strategies, and repos.

---

## Key abstractions

```java
// Strategy — fee calculation pluggable per vehicle type / business policy
public interface FeeStrategy {
    Money calculate(Ticket ticket, Instant exitTime);
}

// Strategy — spot allocation policy
public interface SpotAllocator {
    Optional<ParkingSpot> allocate(Vehicle vehicle, List<Floor> floors);
}

// Repository — abstract ticket persistence
public interface TicketRepository {
    void save(Ticket t);
    Optional<Ticket> findById(String id);
    void update(Ticket t);
}
```

---

## Core code

### Value objects + enums

```java
public enum VehicleType  { MOTORCYCLE, CAR, VAN, TRUCK }
public enum SpotSize     { MOTORCYCLE, COMPACT, REGULAR, LARGE }
public enum SpotStatus   { FREE, OCCUPIED, OUT_OF_SERVICE }
public enum TicketStatus { ACTIVE, PAID, LOST }

public record Vehicle(String licensePlate, VehicleType type) {
    public Vehicle {
        Objects.requireNonNull(licensePlate);
        Objects.requireNonNull(type);
    }
}

public record Money(BigDecimal amount, Currency currency) {
    public static Money zero(Currency c) { return new Money(BigDecimal.ZERO, c); }
    public Money plus(Money other) {
        if (!currency.equals(other.currency)) throw new IllegalArgumentException();
        return new Money(amount.add(other.amount), currency);
    }
}
```

### `ParkingSpot` — the concurrency hot spot

```java
public final class ParkingSpot {
    private final String id;
    private final SpotSize size;
    private final AtomicReference<SpotStatus> status;
    private volatile String currentTicketId;

    public ParkingSpot(String id, SpotSize size) {
        this.id = id;
        this.size = size;
        this.status = new AtomicReference<>(SpotStatus.FREE);
    }

    public String id()          { return id; }
    public SpotSize size()      { return size; }
    public SpotStatus status()  { return status.get(); }

    /** Lock-free reservation. Returns true if successfully reserved. */
    public boolean tryReserve(String ticketId) {
        if (!status.compareAndSet(SpotStatus.FREE, SpotStatus.OCCUPIED)) {
            return false;
        }
        this.currentTicketId = ticketId;
        return true;
    }

    public void release() {
        this.currentTicketId = null;
        status.compareAndSet(SpotStatus.OCCUPIED, SpotStatus.FREE);
    }

    public boolean canFit(VehicleType vehicleType) {
        return switch (vehicleType) {
            case MOTORCYCLE -> true;
            case CAR        -> size != SpotSize.MOTORCYCLE;
            case VAN        -> size == SpotSize.REGULAR || size == SpotSize.LARGE;
            case TRUCK      -> size == SpotSize.LARGE;
        };
    }
}
```

The CAS on `status` is the key: two threads can call `tryReserve` simultaneously; only one wins. No locks held across business logic; the spot itself is the synchronization point.

### `Floor`

```java
public final class Floor {
    private final String id;
    private final List<ParkingSpot> spots;

    public Floor(String id, List<ParkingSpot> spots) {
        this.id = id;
        this.spots = List.copyOf(spots);
    }

    public String id() { return id; }
    public List<ParkingSpot> spots() { return spots; }

    public Stream<ParkingSpot> freeFitting(VehicleType type) {
        return spots.stream()
            .filter(s -> s.status() == SpotStatus.FREE && s.canFit(type));
    }
}
```

### `SpotAllocator` — Strategy

```java
public final class NearestFirstAllocator implements SpotAllocator {
    @Override
    public Optional<ParkingSpot> allocate(Vehicle vehicle, List<Floor> floors) {
        // Prefer smallest matching size to maximize utilization.
        for (Floor floor : floors) {
            Optional<ParkingSpot> spot = floor.freeFitting(vehicle.type())
                .min(Comparator.comparing(ParkingSpot::size));
            if (spot.isPresent()) return spot;
        }
        return Optional.empty();
    }
}
```

The allocator returns an *unreserved* candidate. The orchestrator handles the reserve-or-retry.

### `FeeStrategy`

```java
public final class HourlyFeeStrategy implements FeeStrategy {
    private final Map<VehicleType, BigDecimal> hourlyRates;
    private final Currency currency;

    public HourlyFeeStrategy(Map<VehicleType, BigDecimal> rates, Currency currency) {
        this.hourlyRates = Map.copyOf(rates);
        this.currency = currency;
    }

    @Override
    public Money calculate(Ticket ticket, Instant exit) {
        long minutes = Duration.between(ticket.entryTime(), exit).toMinutes();
        long hours = Math.max(1, (minutes + 59) / 60);   // ceil to next hour, min 1
        BigDecimal rate = hourlyRates.get(ticket.vehicle().type());
        return new Money(rate.multiply(BigDecimal.valueOf(hours)), currency);
    }
}
```

### `Ticket`

```java
public final class Ticket {
    private final String id;
    private final Vehicle vehicle;
    private final String spotId;
    private final Instant entryTime;
    private volatile TicketStatus status;
    private volatile Instant exitTime;

    public Ticket(String id, Vehicle vehicle, String spotId, Instant entryTime) {
        this.id = id; this.vehicle = vehicle;
        this.spotId = spotId; this.entryTime = entryTime;
        this.status = TicketStatus.ACTIVE;
    }

    public String id()           { return id; }
    public Vehicle vehicle()     { return vehicle; }
    public String spotId()       { return spotId; }
    public Instant entryTime()   { return entryTime; }
    public TicketStatus status() { return status; }
    public Instant exitTime()    { return exitTime; }

    public synchronized void markPaid(Instant exit) {
        if (status != TicketStatus.ACTIVE)
            throw new IllegalStateException("ticket not active: " + status);
        this.exitTime = exit;
        this.status = TicketStatus.PAID;
    }
}
```

### `ParkingLot` — orchestrator

```java
public final class ParkingLot {
    private final String id;
    private final List<Floor> floors;
    private final SpotAllocator allocator;
    private final FeeStrategy feeStrategy;
    private final TicketRepository ticketRepo;
    private final Map<String, ParkingSpot> spotsById;
    private final Clock clock;
    private final IdGenerator ids;

    public ParkingLot(String id, List<Floor> floors,
                      SpotAllocator allocator, FeeStrategy feeStrategy,
                      TicketRepository ticketRepo, Clock clock, IdGenerator ids) {
        this.id = id;
        this.floors = List.copyOf(floors);
        this.allocator = allocator;
        this.feeStrategy = feeStrategy;
        this.ticketRepo = ticketRepo;
        this.clock = clock;
        this.ids = ids;
        this.spotsById = floors.stream()
            .flatMap(f -> f.spots().stream())
            .collect(Collectors.toUnmodifiableMap(ParkingSpot::id, s -> s));
    }

    public Ticket park(Vehicle vehicle) {
        for (int attempt = 0; attempt < 3; attempt++) {
            Optional<ParkingSpot> candidate = allocator.allocate(vehicle, floors);
            if (candidate.isEmpty()) throw new LotFullException(vehicle.type());

            ParkingSpot spot = candidate.get();
            String ticketId = ids.next();
            if (spot.tryReserve(ticketId)) {
                Ticket t = new Ticket(ticketId, vehicle, spot.id(), clock.instant());
                ticketRepo.save(t);
                return t;
            }
            // CAS lost — retry with a fresh allocation
        }
        throw new LotContentionException("Failed to park after retries");
    }

    public Receipt unpark(String ticketId) {
        Ticket ticket = ticketRepo.findById(ticketId)
            .orElseThrow(() -> new TicketNotFoundException(ticketId));
        if (ticket.status() != TicketStatus.ACTIVE)
            throw new IllegalStateException("ticket already processed: " + ticket.status());

        Instant exit = clock.instant();
        Money fee = feeStrategy.calculate(ticket, exit);

        ticket.markPaid(exit);
        ticketRepo.update(ticket);

        ParkingSpot spot = spotsById.get(ticket.spotId());
        spot.release();

        return new Receipt(ticket.id(), ticket.vehicle(), ticket.entryTime(), exit, fee);
    }

    public Map<SpotSize, Integer> availability() {
        EnumMap<SpotSize, Integer> result = new EnumMap<>(SpotSize.class);
        for (SpotSize s : SpotSize.values()) result.put(s, 0);
        for (Floor f : floors)
            for (ParkingSpot s : f.spots())
                if (s.status() == SpotStatus.FREE) result.merge(s.size(), 1, Integer::sum);
        return result;
    }
}
```

### In-memory `TicketRepository`

```java
public final class InMemoryTicketRepository implements TicketRepository {
    private final ConcurrentHashMap<String, Ticket> store = new ConcurrentHashMap<>();
    public void save(Ticket t)                   { store.put(t.id(), t); }
    public Optional<Ticket> findById(String id)  { return Optional.ofNullable(store.get(id)); }
    public void update(Ticket t)                 { store.put(t.id(), t); }
}
```

---

## Concurrency strategy

### What's contested
- The status of each `ParkingSpot` (FREE → OCCUPIED → FREE).
- The ticket repository (writes from many gates).

### What's not
- The `floors` and `spots` lists themselves — built once at startup, immutable thereafter.
- `Ticket` after creation — mostly immutable; only `status` and `exitTime` mutate, and only at end of life.

### Strategy
1. **Per-spot CAS on `status`** — the load-bearing concurrency primitive. No global lot lock. Throughput scales with the number of spots.
2. **`ConcurrentHashMap` for the ticket store** — many concurrent readers and writers.
3. **Allocator returns candidates without reserving** — the orchestrator handles reserve-or-retry. Allocator stays simple; concurrency lives in one place.
4. **Bounded retry on CAS miss** — if many cars contend on the same spot, alternate candidates exist; backing off and re-allocating is cheap.

### Why not lock the whole lot or the whole floor?
- One global lock = single-threaded throughput.
- One floor lock = floors with hot demand serialize.
- Per-spot CAS = throughput scales with spots.

The cost is the retry path on contention — but that path runs only when there's actual contention. In the steady state, every park is one CAS.

### Crash recovery
If we crash between `markPaid` and `release()`, the spot stays OCCUPIED forever. Mitigation: a background sweeper detects PAID tickets with still-OCCUPIED spots and reconciles. In a real system, both updates would be in the same transaction.

---

## Edge cases / gotchas

| Case | Handling |
|---|---|
| **Lot full** | `park` throws `LotFullException`. |
| **Multiple gates allocate same last spot** | CAS wins one; loser retries with fresh allocation. |
| **Ticket lost** | `markLost(ticketId)` flow → flat penalty fee. |
| **Vehicle never exits** | Sweeper / max-stay policy. |
| **Already-exited ticket re-presented** | Reject — `status != ACTIVE`. |
| **Spot taken out of service mid-occupancy** | Allow current parking to complete; transition on next release. |
| **Crash between paying and releasing spot** | Reconciliation job: tickets PAID + spots OCCUPIED → release. |
| **Money rounding** | `BigDecimal` only; specify rounding mode. |
| **Time changes (DST)** | Use `Instant`, not `LocalDateTime`. |
| **Vehicle type changes between requests** | Use the ticket's stored vehicle, not the vehicle at exit. |

---

## Extensions / interview follow-ups

### "How would you support reservations?"
- Add `Reservation` aggregate: `(spotId, vehicle, fromTime, toTime)`.
- Allocator filters out spots reserved during the requested window.
- Spot status gains `RESERVED`; transition `RESERVED → OCCUPIED` on actual entry.

### "What if the lot has 1 million spots?"
- Per-floor TreeSet of free spots indexed by size for O(log n) allocation.
- Lazy loading of floors into memory; persistent storage per floor.
- Sharded allocator instances, one per zone.

### "How would you scale across multiple lots?"
- `LotRegistry` aggregate; clients pick a lot; per-lot design unchanged.
- Cross-lot search becomes a separate service ("find capacity within 500m of point P").

### "How would you add EV charging?"
- `ParkingSpot` gains `boolean hasCharger`.
- `Vehicle` gains `boolean needsCharger`.
- Allocator filter: needsCharger → spot has charger.

### "How would you handle dynamic pricing (surge)?"
- `FeeStrategy` already pluggable. Add `SurgeFeeStrategy` reading occupancy and multiplying.
- `availability()` feeds the strategy.

### "How would you persist?"
- `TicketRepository` interface already there. Add `JdbcTicketRepository`.
- Spot status also needs persistence — write status changes to a journal; reconcile on startup.

### "How would you display real-time availability?"
- Add **Observer**: `ParkingSpot` notifies on state change.
- Display board subscribes; aggregates per-floor / per-size.

### "How would you implement nearest-spot allocation when 'nearest' actually means physical distance to entry?"
- `Floor` and `ParkingSpot` carry `Coordinate` value objects.
- Allocator becomes a priority queue ordered by distance from entry gate.

---

## Scaling

This design is correct for one parking lot up to ~100k spots. Beyond that, or for a multi-lot operator like ParkPlus / SpotHero, the architecture changes:

### Multi-lot
- **Lot registry service** at the top — clients query "lots near me with capacity" first.
- Each lot is its own service / process, with its own DB.
- Cross-lot queries are eventually consistent; spot-level reservation is per-lot.

### Persistence
- **Spot status journal** — every status change is an append. Reconstruct state on startup by replay.
- **Ticket DB** — relational with indexes on `(lotId, status)`, `(vehicle.plate)`.
- Use **optimistic locking** with a `version` column on the spot for cross-process safety (the in-memory CAS becomes a `WHERE version = ?` clause in SQL).

### High-throughput allocation
- Per-zone in-memory cache of free spots, write-through to DB.
- Sticky routing of allocation requests to the zone owning the spot — avoids lock contention across regions.

### Real-time displays
- Spot status changes publish to a **pub/sub topic** (Kafka, Redis Pub/Sub).
- Display services subscribe and aggregate; eventually consistent is fine for a "13 free regular" sign.

### Reservations + dynamic pricing at scale
- Reservation engine becomes its own service with a time-window data structure (interval tree per spot).
- Pricing service consumes occupancy stream; outputs surge multipliers per zone.

### Failure modes
- **Network partition between gate and central system** — gates need a local cache of recent tickets so exit can complete during a partition; reconciliation on reconnect.
- **Payment service down at exit** — issue an unpaid receipt with grace period; reconcile via offline batch.
- **Sensor misreport** — physical sensors say "occupied" but no ticket exists — flag for operator review; don't double-bill.

---

## Persistence tradeoffs

The in-memory design works for one process. Once you want crash recovery, multi-instance, or long-term reporting, persistence enters. The interesting choices:

### What persistent data exists
- **Spots** — current status (FREE / OCCUPIED / OUT_OF_SERVICE), current ticket reference. Updated frequently.
- **Tickets** — entry/exit timestamps, vehicle, fee. Append-mostly with one update at exit.
- **Lot/Floor structure** — built once, rarely changes.
- **Audit history** — every status transition for compliance / dispute resolution.

### Option 1 — Relational (PostgreSQL / MySQL)
Schema sketch:
```sql
spots(id, floor_id, size, status, current_ticket_id, version)
tickets(id, lot_id, spot_id, license_plate, vehicle_type,
         entry_time, exit_time, status, fee_amount, fee_currency)
spot_status_log(spot_id, status, ticket_id, ts, change_reason)   -- audit
floors(id, lot_id, label)
lots(id, name, address)
```
- **The CAS on spot status becomes** `UPDATE spots SET status='OCCUPIED', current_ticket_id=?, version=version+1 WHERE id=? AND status='FREE' AND version=?`. Affected-rows = 0 means contention; retry.
- **One transaction** wraps "claim spot + insert ticket" so the two can't diverge.
- **Indexes**: `(spot_id, status)` for free-spot scans, `(license_plate)` for finding active tickets without an ID, `(lot_id, status)` on tickets for active counts.
- **Pros**: ACID exactly matches the model. Reports (revenue per day, occupancy) are SQL queries.
- **Cons**: write hotspots if all gates push into the same `spot_status_log` partition — table-partition by `lot_id` or use a write-buffered log.
- **When to pick**: default for any real parking system.

### Option 2 — Redis for hot state, SQL for history
- **Redis** holds spot statuses as keys: `spot:{id}` → JSON `{status, ticketId}`. Atomic transitions via Lua scripts (`if GET = 'FREE' then SET, return ticketId`).
- **SQL** holds tickets, audit log, lot structure. Synchronous on entry/exit; eventual on heartbeat for spot snapshots.
- **Pros**: spot allocation is sub-millisecond. The SQL doesn't see the frequent status thrash.
- **Cons**: two stores to keep consistent. Lua scripts get hairy. Disaster recovery is more complex (Redis must persist or you reconstruct from SQL on boot).
- **When to pick**: very large lots with high entry/exit rates (airports, malls).

### Option 3 — DynamoDB / Cosmos DB
- Single table with PK = `spot:{id}` for spots, PK = `ticket:{id}` for tickets.
- Conditional writes (`condition-expression: status = 'FREE' AND version = :v`) replace the SQL `UPDATE ... WHERE`.
- **Pros**: horizontal scaling, predictable latency, no schema migrations.
- **Cons**: secondary indexes are first-class but expensive at write time. Aggregate reporting requires DynamoDB Streams → analytics store.
- **When to pick**: multi-region, very high scale, willing to pay for managed.

### Option 4 — Event sourcing
- Every status transition is an event: `SpotOccupied{spotId, ticketId, ts}`, `SpotReleased{spotId, ts}`.
- Store events in Kafka or an `events` SQL table.
- Current state is a fold over events; cache the current state in Redis or an in-memory projection.
- **Pros**: complete audit history for free. Time-travel debugging ("what was occupancy at 3pm yesterday").
- **Cons**: read latency unless you maintain a projection. Schema evolution across years of events is hard.
- **When to pick**: regulated environments needing audit (some jurisdictions require parking transaction histories).

### Recommendation for parking lot
- **Default**: PostgreSQL for everything. The data volume per lot (thousands of transactions per day) is well within SQL's comfort zone.
- **At scale**: Add Redis only if profiling shows the spot-status hot path bottlenecks. Don't introduce two stores prematurely.
- **For reporting**: nightly ETL from SQL to a warehouse (BigQuery, Snowflake) for analytics; don't run heavy aggregations on the OLTP DB.

### Concurrency under persistence

The CAS pattern translates directly to the chosen DB:

| In-memory | SQL | Redis | DynamoDB |
|---|---|---|---|
| `AtomicReference.compareAndSet` | `UPDATE ... WHERE version = ?` | `WATCH/MULTI/EXEC` or Lua | `condition-expression` |
| 0 rows affected = retry | 0 rows affected = retry | EXEC fails = retry | `ConditionalCheckFailed` = retry |

The "try-and-retry" loop in `ParkingLot.park` stays the same; only the underlying primitive changes.

### Crash recovery

In-memory: a crash mid-operation can leave a spot OCCUPIED with no ticket. With persistence:
- **SQL transaction** spanning "insert ticket + update spot" — atomic; no half-states.
- **Redis Lua script** — atomic across reads and writes.
- **Two-store design** (Redis + SQL) — needs reconciliation: a sweeper reads SQL tickets and reconciles Redis spot states on startup.

---

## Talking points for the interview

- "I'm using CAS on the spot status rather than a lot-level lock; throughput scales with the number of spots, not the number of gates."
- "Fee calculation is a Strategy because pricing is the most likely thing to change — surge, promotions, daily caps — and I don't want each change to touch the orchestrator."
- "I inject `Clock` so time-based behavior is testable. Same for the ID generator and the repository."
- "Tickets and spots are separate aggregates. The spot is the synchronization point for occupancy; the ticket is the audit trail. Either can fail without corrupting the other if I add idempotency keys to entry."
- "If I needed to persist, the `TicketRepository` interface is already in place. The spot state is trickier — I'd write transitions to a journal and reconcile on startup."
- "I deliberately considered locking the whole lot or the whole floor and rejected both — both serialize gates, and modern lots have many gates."

---

## Summary

Three patterns load-bearing: **Strategy** (fee, allocator), **Repository** (ticket persistence), **CAS-based concurrency** (per-spot atomic status).

The mental model: lot is a tree (lot → floor → spot). Allocator searches the tree for a candidate. Spot's `tryReserve` is a CAS; on success you have a ticket; on failure you retry. Exit is the inverse: find ticket → compute fee → mark paid → release spot.

Adding new behavior — different fee model, different allocation strategy, EV chargers, reservations — is a one-class change against existing interfaces. That's what a clean LLD design buys you.
