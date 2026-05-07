# Problem 8 — Elevator System

A scheduling and dispatch problem. Tests state-per-car, dispatch Strategy, priority queues, and the discipline to keep individual elevator behavior separate from fleet coordination.

## Interview opener

> **Interviewer:** Design an elevator control system.

---

## Clarification phase

> **You:** Single elevator or multiple elevators in one building?
>
> **Interviewer:** Multiple — assume a typical office building, say 4 elevators across 20 floors.

> **You:** Two kinds of buttons: floor buttons (inside the car, "I want to go to floor X") and hall buttons (on a floor, "I want to go up/down")?
>
> **Interviewer:** Yes.

> **You:** Should I implement a specific scheduling algorithm or design the system to support multiple?
>
> **Interviewer:** Design for multiple — make it pluggable. Implement at least one (nearest-elevator or SCAN/LOOK).

> **You:** Service level — should one car answer one hall request, or can multiple cars converge?
>
> **Interviewer:** Just one car per hall request — pick the best.

> **You:** Door operations — auto-open / auto-close timers, hold buttons?
>
> **Interviewer:** Auto-close after a few seconds. Skip the hold button.

> **You:** Special modes — fire mode, VIP mode (express to lobby), maintenance?
>
> **Interviewer:** Maintenance mode is in scope. Mention fire mode but skip implementation.

> **You:** Capacity / weight limits?
>
> **Interviewer:** Capacity by passenger count. If full, ignore further hall requests but still honor inside requests until empty.

> **You:** Where does the request come from — physical button, app?
>
> **Interviewer:** Doesn't matter — model the request as an input event.

> **You:** Persistence?
>
> **Interviewer:** Discuss at end.

OK — design needs: per-elevator state machine, central dispatcher with pluggable algorithm, separate hall vs car requests, capacity tracking.

---

## Refined requirements

### Functional
1. Press a hall button on floor F with direction D → request `pickup(F, D)`.
2. Inside the car, press floor F → request `goTo(carId, F)`.
3. Dispatcher routes hall requests to the best car.
4. Each car services its requests in an efficient order (SCAN/LOOK).
5. Car stops at a floor: opens doors, waits, closes doors, continues.
6. Maintenance mode toggles per car.

### Non-functional
- **Latency**: dispatcher decision in O(N) where N = cars.
- **Concurrency**: many requests arrive concurrently; per-car state must remain consistent.
- **Fairness**: no request starved indefinitely.

### Out of scope (mentioned)
- Fire / emergency mode.
- Weight sensors.
- Hold-doors button.
- Networked / multi-bank elevator hall calls.

### Core operations
1. `Dispatcher.requestPickup(floor, direction)` — hall call.
2. `Elevator.requestStop(floor)` — car call.
3. `Elevator.tick()` — advance simulation by one time unit (or, in real systems, this is event-driven).
4. `Dispatcher.setMaintenance(carId, boolean)`.

---

## Domain model

### Entities
- **`ElevatorSystem`** (Dispatcher) — aggregate root.
- **`Elevator`** — one per car; has its own state and request set.

### Value objects
- **`HallRequest(floor, direction)`** — immutable.
- **`CarRequest(floor)`** — immutable.

### Enums
- `Direction` — `UP`, `DOWN`, `IDLE`.
- `DoorState` — `OPEN`, `CLOSED`, `OPENING`, `CLOSING`.
- `ElevatorState` — `IDLE`, `MOVING`, `STOPPED`, `MAINTENANCE`.

### Patterns
- **State** — Elevator lifecycle (idle, moving, stopped, maintenance).
- **Strategy** — dispatcher's car-selection algorithm; in-car request ordering.
- **Mediator** — `ElevatorSystem` mediates between hall buttons and cars.
- **Command** — request objects routed through the system.
- **Observer** — UI / floor displays subscribe to state changes.

---

## Class diagram

```
┌─────────────────────────────────────┐
│         ElevatorSystem              │   (Dispatcher / Mediator)
├─────────────────────────────────────┤
│ -elevators : List<Elevator>         │
│ -dispatchStrategy : DispatchStrategy│ ◇──► DispatchStrategy
│ -pendingHallRequests : Set<HallReq> │
├─────────────────────────────────────┤
│ +requestPickup(floor, dir)          │
│ +setMaintenance(carId, on)          │
│ +tick()                             │
└─────────────────────────────────────┘
       │ ◆ 1..*
       ▼
┌──────────────────────────────────────┐
│             Elevator                 │
├──────────────────────────────────────┤
│ -id : String                         │
│ -currentFloor : int                  │
│ -direction : Direction               │
│ -state : ElevatorState               │
│ -doorState : DoorState               │
│ -capacity : int                      │
│ -occupancy : int                     │
│ -upStops : TreeSet<Integer>          │
│ -downStops : TreeSet<Integer>        │
│ -orderingStrategy : RequestOrdering  │ ◇──► RequestOrdering
├──────────────────────────────────────┤
│ +addCarRequest(floor)                │
│ +acceptHallAssignment(req)           │
│ +tick()                              │
│ +nextStop() : OptionalInt            │
└──────────────────────────────────────┘

┌────────────────────────┐    ┌──────────────────────────┐
│ <<interface>>          │    │  <<interface>>           │
│ DispatchStrategy       │    │  RequestOrdering         │
├────────────────────────┤    ├──────────────────────────┤
│ +pick(req, fleet)      │    │ +nextFloor(state):       │
│   : Optional<Elevator> │    │   OptionalInt            │
└────────────────────────┘    └──────────────────────────┘
       △                              △
   ┌───┴────────┐                ┌────┴─────┐
┌───────────────┐ ┌──────────────┐ ┌──────────────┐
│NearestElevator│ │ZoneBased     │ │ ScanLookOrder│
│ Dispatcher    │ │Dispatcher    │ │              │
└───────────────┘ └──────────────┘ └──────────────┘

   record HallRequest(int floor, Direction direction)
```

---

## Key abstractions

```java
public interface DispatchStrategy {
    /** Picks the best elevator for the hall request, or empty if none can serve. */
    Optional<Elevator> pick(HallRequest req, List<Elevator> fleet);
}

public interface RequestOrdering {
    /** Given current floor and direction + pending stops, returns next stop floor. */
    OptionalInt nextFloor(int currentFloor, Direction direction,
                          TreeSet<Integer> upStops, TreeSet<Integer> downStops);
}
```

---

## Core code

### Value objects + enums

```java
public enum Direction { UP, DOWN, IDLE }
public enum DoorState { OPEN, CLOSED, OPENING, CLOSING }
public enum ElevatorState { IDLE, MOVING, STOPPED, MAINTENANCE }

public record HallRequest(int floor, Direction direction) {
    public HallRequest {
        if (direction == Direction.IDLE) throw new IllegalArgumentException();
    }
}
```

### `Elevator`

```java
public final class Elevator {
    private final String id;
    private final int minFloor;
    private final int maxFloor;
    private final int capacity;
    private final RequestOrdering ordering;

    private int currentFloor;
    private Direction direction = Direction.IDLE;
    private ElevatorState state = ElevatorState.IDLE;
    private DoorState doorState = DoorState.CLOSED;
    private int occupancy = 0;

    // Per-direction sorted sets of pending stops
    private final TreeSet<Integer> upStops   = new TreeSet<>();
    private final TreeSet<Integer> downStops = new TreeSet<>(Comparator.reverseOrder());

    public Elevator(String id, int minFloor, int maxFloor, int capacity,
                    RequestOrdering ordering, int initialFloor) {
        this.id = id; this.minFloor = minFloor; this.maxFloor = maxFloor;
        this.capacity = capacity; this.ordering = ordering;
        this.currentFloor = initialFloor;
    }

    public String id()              { return id; }
    public int currentFloor()       { return currentFloor; }
    public Direction direction()    { return direction; }
    public ElevatorState state()    { return state; }
    public int capacity()           { return capacity; }
    public int occupancy()          { return occupancy; }
    public boolean isFull()         { return occupancy >= capacity; }
    public boolean isInService()    { return state != ElevatorState.MAINTENANCE; }

    public synchronized void addCarRequest(int floor) {
        validateFloor(floor);
        addStop(floor);
        if (state == ElevatorState.IDLE) wakeUp();
    }

    public synchronized void acceptHallAssignment(HallRequest req) {
        validateFloor(req.floor());
        addStop(req.floor());
        // Direction is recorded so that when we stop here, we also know
        // which direction-bound passengers will board.
        if (state == ElevatorState.IDLE) wakeUp();
    }

    private void addStop(int floor) {
        if (floor > currentFloor) upStops.add(floor);
        else if (floor < currentFloor) downStops.add(floor);
        else openDoors();    // already here
    }

    private void wakeUp() {
        state = ElevatorState.MOVING;
        if (direction == Direction.IDLE) {
            // Decide initial direction by which set has work
            direction = !upStops.isEmpty() ? Direction.UP
                      : !downStops.isEmpty() ? Direction.DOWN
                      : Direction.IDLE;
        }
    }

    /** One simulation tick — advance the elevator by one floor or open/close doors. */
    public synchronized void tick() {
        if (state == ElevatorState.MAINTENANCE) return;

        OptionalInt next = ordering.nextFloor(currentFloor, direction, upStops, downStops);
        if (next.isEmpty()) {
            state = ElevatorState.IDLE;
            direction = Direction.IDLE;
            return;
        }

        int target = next.getAsInt();
        if (target == currentFloor) {
            // Arrived. Open doors, mark this stop as served.
            openDoors();
            upStops.remove(currentFloor);
            downStops.remove(currentFloor);
            // Re-evaluate direction
            if (direction == Direction.UP   && upStops.isEmpty())   direction = downStops.isEmpty() ? Direction.IDLE : Direction.DOWN;
            if (direction == Direction.DOWN && downStops.isEmpty()) direction = upStops.isEmpty()   ? Direction.IDLE : Direction.UP;
            return;
        }

        // Move one floor toward target
        currentFloor += (target > currentFloor) ? 1 : -1;
    }

    private void openDoors()  { doorState = DoorState.OPEN;   }
    private void closeDoors() { doorState = DoorState.CLOSED; }

    public synchronized void enterPassengers(int n) {
        if (n + occupancy > capacity) throw new IllegalStateException("Over capacity");
        occupancy += n;
    }
    public synchronized void exitPassengers(int n) { occupancy -= n; }

    public synchronized void enterMaintenance() { state = ElevatorState.MAINTENANCE; }
    public synchronized void exitMaintenance()  { state = ElevatorState.IDLE; }

    /** Cost-to-serve estimate for dispatcher — how far is this car from picking up at (floor, dir)? */
    public synchronized int costToServe(HallRequest req) {
        if (!isInService() || isFull()) return Integer.MAX_VALUE;
        // Same direction, request is "on the way"
        if (direction == req.direction()) {
            if (direction == Direction.UP   && req.floor() >= currentFloor) return req.floor() - currentFloor;
            if (direction == Direction.DOWN && req.floor() <= currentFloor) return currentFloor - req.floor();
        }
        if (direction == Direction.IDLE) return Math.abs(currentFloor - req.floor());
        // Otherwise: would need to finish current direction first
        // Compute approximate detour cost
        int extreme = direction == Direction.UP
            ? upStops.isEmpty() ? currentFloor : upStops.last()
            : downStops.isEmpty() ? currentFloor : downStops.last();
        return Math.abs(extreme - currentFloor) + Math.abs(extreme - req.floor());
    }

    private void validateFloor(int f) {
        if (f < minFloor || f > maxFloor) throw new IllegalArgumentException("floor " + f);
    }
}
```

### Request ordering — SCAN/LOOK Strategy

```java
public final class ScanLookOrdering implements RequestOrdering {
    /**
     * SCAN-LOOK: serve all pending stops in the current direction
     * (up to highest pending), then reverse and serve the other direction.
     */
    public OptionalInt nextFloor(int current, Direction dir,
                                 TreeSet<Integer> upStops, TreeSet<Integer> downStops) {
        if (dir == Direction.UP) {
            // Lowest stop above us
            Integer next = upStops.ceiling(current);
            if (next != null) return OptionalInt.of(next);
            // No more up-direction stops; reverse if there are down stops
            if (!downStops.isEmpty()) return OptionalInt.of(downStops.first());
            return OptionalInt.empty();
        }
        if (dir == Direction.DOWN) {
            Integer next = downStops.ceiling(current);   // (reversed comparator: ceiling is "next lower")
            if (next != null) return OptionalInt.of(next);
            if (!upStops.isEmpty()) return OptionalInt.of(upStops.last());
            return OptionalInt.empty();
        }
        // IDLE — pick the closest stop
        Integer up = upStops.ceiling(current);
        Integer down = downStops.ceiling(current);
        if (up == null && down == null) return OptionalInt.empty();
        if (up == null) return OptionalInt.of(down);
        if (down == null) return OptionalInt.of(up);
        return OptionalInt.of(Math.abs(up - current) <= Math.abs(down - current) ? up : down);
    }
}
```

### Dispatch — nearest elevator (Strategy)

```java
public final class NearestElevatorDispatcher implements DispatchStrategy {
    @Override
    public Optional<Elevator> pick(HallRequest req, List<Elevator> fleet) {
        return fleet.stream()
            .filter(Elevator::isInService)
            .filter(e -> !e.isFull())
            .min(Comparator.comparingInt(e -> e.costToServe(req)));
    }
}
```

For zone-based dispatching: partition floors among elevators (e.g., low-rise, mid-rise, high-rise) and route requests by zone. Useful in skyscrapers where dedicated zones reduce average response time.

### `ElevatorSystem` — Dispatcher

```java
public final class ElevatorSystem {
    private final List<Elevator> elevators;
    private final DispatchStrategy dispatcher;
    private final Set<HallRequest> pendingHallRequests = ConcurrentHashMap.newKeySet();

    public ElevatorSystem(List<Elevator> elevators, DispatchStrategy dispatcher) {
        this.elevators = List.copyOf(elevators);
        this.dispatcher = dispatcher;
    }

    public void requestPickup(int floor, Direction direction) {
        HallRequest req = new HallRequest(floor, direction);
        Optional<Elevator> chosen = dispatcher.pick(req, elevators);
        if (chosen.isPresent()) {
            chosen.get().acceptHallAssignment(req);
        } else {
            pendingHallRequests.add(req);   // retry on next tick
        }
    }

    public void setMaintenance(String carId, boolean on) {
        elevators.stream().filter(e -> e.id().equals(carId)).findFirst().ifPresent(e -> {
            if (on) e.enterMaintenance();
            else    e.exitMaintenance();
        });
    }

    /** One time tick — advance all elevators; retry pending hall requests. */
    public void tick() {
        for (Elevator e : elevators) e.tick();

        // Retry pending
        var iter = pendingHallRequests.iterator();
        while (iter.hasNext()) {
            HallRequest req = iter.next();
            Optional<Elevator> chosen = dispatcher.pick(req, elevators);
            if (chosen.isPresent()) {
                chosen.get().acceptHallAssignment(req);
                iter.remove();
            }
        }
    }
}
```

### Putting it together

```java
List<Elevator> fleet = List.of(
    new Elevator("E1", 0, 19, 10, new ScanLookOrdering(), 0),
    new Elevator("E2", 0, 19, 10, new ScanLookOrdering(), 5),
    new Elevator("E3", 0, 19, 10, new ScanLookOrdering(), 12),
    new Elevator("E4", 0, 19, 10, new ScanLookOrdering(), 19)
);
ElevatorSystem system = new ElevatorSystem(fleet, new NearestElevatorDispatcher());

// Hall request from floor 7 going up
system.requestPickup(7, Direction.UP);
// Inside elevator E1, passenger requests floor 14
fleet.get(0).addCarRequest(14);

// Simulation loop (in real system this is event-driven, not tick-based)
for (int t = 0; t < 30; t++) system.tick();
```

---

## Concurrency strategy

### What's contested
- Each elevator's pending stops (mutated by hall assignments + car requests + tick).
- The dispatcher's pending request set.

### Strategy
- **Per-elevator `synchronized`** — every `Elevator` method that touches state is synchronized. Lock granularity is per-car, so different cars don't contend.
- **`ConcurrentHashMap.newKeySet`** for the pending request set — concurrent producers (hall presses) and consumers (the dispatcher's tick).
- **Dispatcher reads elevator state** for cost computation while elevators may be updating — that's acceptable as long as the dispatcher's read is consistent (each call to `costToServe` is itself synchronized inside the elevator).

### What about "dispatcher picks E1, but E1 fills up before the assignment lands"?
- The dispatcher reads `e.costToServe(req)` and `e.isFull()` — both inside the elevator's monitor.
- Then calls `e.acceptHallAssignment(req)` — also synchronized.
- Between those two calls, state can change. If `acceptHallAssignment` checks capacity and rejects, the dispatcher must reassign. Ours doesn't reject — it just enqueues, and the elevator services if it can. Acceptable.

### Tick-based vs event-driven
The `tick()` model is convenient for the interview. In real elevators, every event (button press, door close, floor sensor) drives the state machine — so `tick` becomes a set of event handlers, each operating under the elevator's monitor.

---

## Edge cases / gotchas

| Case | Handling |
|---|---|
| **Hall request with no available elevator** | Enqueue; retry on next tick. |
| **All elevators in maintenance** | Same — pending forever until one returns. |
| **Car at full capacity** | Skip assignment; don't pick up more (but still serve in-car requests until empty). |
| **Hall request floor = current floor** | Open doors; mark served immediately. |
| **Same floor pressed twice** | Idempotent (TreeSet dedupes). |
| **Direction flip mid-trip with no remaining same-direction stops** | Reverse direction; SCAN/LOOK handles this. |
| **Maintenance toggled mid-trip** | Finish current assignment, then enter maintenance? Decision: refuse new requests immediately; complete current motion to a safe stop. |
| **Floor out of bounds** | Throw at request time. |
| **Concurrent request and tick** | Per-car `synchronized` ensures atomicity of "add stop + decide direction." |
| **Dispatcher computes cost based on stale state** | Acceptable; on next tick the state may differ but the algorithm is still self-correcting. |

---

## Extensions / interview follow-ups

### "Add fire mode."
- All elevators return to the ground floor; doors open; ignore further requests until cleared.
- Add `FireModeState` or a global flag on `ElevatorSystem`.

### "Add VIP mode (express to lobby)."
- Mark a car for express; clear all pending stops; go directly to a target floor.
- Doable as a special command: `vipExpress(carId, targetFloor)`.

### "Add destination dispatch (you specify destination at the hall, not inside)."
- Hall calls become `HallRequest(originFloor, destinationFloor)`.
- Dispatcher groups similar destinations into one car (great for skyscrapers).
- Inside the car, no buttons — just a display showing where it's going.

### "Add weight sensors."
- `Elevator` has `currentWeightKg`; capacity becomes weight-based.
- Refuse hall pickups when at weight limit.

### "Add cross-bank coordination (multiple elevator groups)."
- Each bank has its own `ElevatorSystem`.
- A higher-level dispatcher routes hall requests to the nearest bank with capacity.

### "Why per-car locking and not a global system lock?"
- A global lock means N elevators serialize through one mutex; throughput limited.
- Per-car locks mean cars run independently; the dispatcher does cheap cross-car reads but only commits assignments inside one car's lock.

### "Why two TreeSets per car instead of one priority queue?"
- The natural traversal pattern is "go up serving up-stops, reverse, go down serving down-stops." Two sorted sets express this directly.
- A single priority queue would need a custom comparator that flips when direction changes — more error-prone.

### "How do you guarantee fairness — no request waits forever?"
- SCAN/LOOK already provides bounded waiting: the elevator always sweeps to the extremes.
- The dispatcher's pending-request set ensures no request is lost even if no car is currently available.
- For pathological cases (a single hall press at floor 1 while all cars are busy at floor 20), a priority bump (waiting time as a tiebreaker in dispatcher cost) helps.

---

## Scaling

A single building's elevator system is small. "Scaling" appears for fleet operators:

### Building-level
- The design above is correct.
- Latency is negligible (everything in-process).

### Multi-building (operator with hundreds of buildings)
- Each building is its own service.
- Centralized telemetry: car uptime, average wait time, mean time between failures.
- Predictive maintenance (vibration analysis, motor temperature trends).

### Energy optimization
- During off-peak, park cars at strategic floors (lobby for morning rush; mid-floors for off-peak).
- "Mode" Strategy varies by time of day.

### Smart scheduling with ML
- Learn building-specific traffic patterns; pre-position cars.
- Beyond LLD scope but worth mentioning.

---

## Persistence tradeoffs

Elevator state is mostly volatile and reconstructible. Persistence shows up for: telemetry, audit, maintenance records, and configuration.

### What persistent data exists
- **Configuration**: floors, elevator inventory, dispatch strategy, capacity, zones.
- **Operational state**: current floor, direction, maintenance status. Volatile but valuable for monitoring.
- **Telemetry / time-series**: floor visits, door opens, error events, weight readings.
- **Maintenance log**: who serviced what when.
- **Audit log**: incidents, fire-mode activations.

### Option 1 — In-memory + periodic snapshot
- Each `ElevatorSystem` is in-memory.
- Every minute, snapshot to disk: per-car position, direction, occupancy.
- On restart, load snapshot; pending hall requests are lost but new ones come in.
- **Pros**: simple. Latency unaffected.
- **Cons**: requests in flight at crash time are lost (acceptable — physical buttons get pressed again).
- **When to pick**: standalone building controller.

### Option 2 — Embedded SQL on the controller
- SQLite on the controller box.
- Persist on every state change (or batch every N events).
- Retain rolling window (last 24h) for replay.
- **Pros**: queryable; supports operator dashboards.
- **Cons**: more I/O than necessary for a system that's already deterministic.
- **When to pick**: regulated environments needing audit (some jurisdictions require maintenance logs).

### Option 3 — Time-series DB (telemetry)
- Each event (floor-visit, door-open, fault) emitted to a time-series DB (InfluxDB / TimescaleDB / Prometheus).
- Cars push every ~second; central system aggregates.
- **Pros**: fits the natural shape of the data (time-indexed, mostly append).
- **Cons**: not the source of truth — separate DB for configuration and audit.
- **When to pick**: fleet operators monitoring many buildings.

### Option 4 — Event-sourced
- Every hall press, car request, door event, position change is an event.
- Current state derived from event log.
- **Pros**: complete history; replay is trivial; safety-incident forensics natural.
- **Cons**: storage volume; replay cost on cold start.
- **When to pick**: regulated environments where every elevator action must be reconstructible (post-incident audits).

### Option 5 — Hybrid (recommended for production)
- **In-memory** for hot operational state (low-latency dispatch).
- **Time-series DB** (Prometheus / Influx) for telemetry.
- **PostgreSQL** for configuration, maintenance records, audit incidents.
- **Event journal** (Kafka or append-only file) for the event stream — feeds both the time-series store and audit.

### Concurrency under persistence
The in-memory `synchronized` model continues to work; persistence happens **outside** the critical section:
1. Inside the lock: mutate state, append event to a thread-local buffer.
2. Outside the lock: flush buffer to durable store.
- This avoids holding the elevator's monitor across I/O.

For audit-critical events (fire mode triggered, override commands), use synchronous write — accept the latency for correctness.

### Recovery semantics
- **Cold start after crash**: load configuration from SQL. Trust the physical sensors for actual elevator position (cars know where they are; software just reflects).
- **Pending requests**: lost. Users re-press buttons. Acceptable in practice.
- **Maintenance state**: persisted (a car in maintenance must stay in maintenance across restarts for safety).

---

## Talking points for the interview

- "Per-elevator state machine with per-elevator locks; fleet coordination is at the dispatcher, lockless across cars except when committing an assignment."
- "Two TreeSets per car (up-bound, down-bound) make SCAN/LOOK natural — the algorithm is just `next floor in current direction's set, else flip`."
- "Dispatch is a Strategy because building shapes change everything — nearest-elevator vs zone-based vs destination-dispatch — and we should be able to swap without touching cars."
- "Cost-to-serve is approximate; that's fine because the elevator re-evaluates on every tick and can absorb dispatcher imprecision."
- "Pending hall requests are buffered when no car can take them — re-tried each tick. Bounds the worst-case wait."
- "Telemetry is async via a write buffer; the hot dispatch path doesn't pay for I/O."

---

## Summary

Patterns load-bearing: **State** (per-elevator), **Strategy** (dispatch + ordering), **Mediator** (dispatcher coordinates between hall buttons and cars).

The mental model: each elevator is a self-driving sweeping machine; the dispatcher just decides which sweep covers a new request. The elevators are independent state machines; the dispatcher is the only place they touch.

The hardest concept here is the SCAN/LOOK ordering — it's classic disk-arm scheduling repurposed for elevators. Visualizing the floor list as two sorted sets (up-bound, down-bound) makes it natural in code.
