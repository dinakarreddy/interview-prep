# Problem 15 — Cab Booking (Uber / Lyft)

The spatial-indexing + dispatch problem. Tests how you handle "drivers near a point" at scale, atomic driver assignment under concurrency, trip lifecycle, and surge pricing.

## Interview opener

> **Interviewer:** Design a cab-booking system like Uber.

---

## Clarification phase

> **You:** Riders request rides; drivers nearby are matched; trip happens; payment?
>
> **Interviewer:** Right. Focus on the matching and trip lifecycle. Payment can be a black box.

> **You:** Geographic scope — single city for now?
>
> **Interviewer:** Single city for the design. Discuss multi-city scaling at the end.

> **You:** Driver matching strategy — nearest, ETA-based, fairness, surge?
>
> **Interviewer:** Nearest by ETA for the core. Strategy-pluggable.

> **You:** Driver locations — how often do they update?
>
> **Interviewer:** Every few seconds while online. Real-time.

> **You:** Multiple ride types — economy, premium, pool?
>
> **Interviewer:** Yes — categories matter for matching.

> **You:** Surge pricing?
>
> **Interviewer:** Yes — pluggable.

> **You:** Concurrent matching — two riders requesting near each other might end up matched to the same driver. How to prevent?
>
> **Interviewer:** That's a key challenge. Ensure exactly-one assignment per driver.

> **You:** Trip lifecycle — request, match, en-route, in-progress, complete, cancel?
>
> **Interviewer:** All of those. State machine.

> **You:** Cancellations — by rider or driver? Penalties?
>
> **Interviewer:** Both. Penalties out of scope.

> **You:** Persistence?
>
> **Interviewer:** Discuss at end.

OK — design needs: spatial index for fast "drivers near a point" lookups, Strategy for matching, atomic driver assignment to prevent double-bookings, State machine for trip lifecycle, surge pricing.

---

## Refined requirements

### Functional
1. Driver: `goOnline(location)`, `updateLocation(location)`, `goOffline()`.
2. Rider: `requestRide(pickup, dropoff, rideType)` → `Trip` (after matching).
3. Driver matching: find nearby drivers, pick the best one, atomically assign.
4. Trip lifecycle: REQUESTED → MATCHED → DRIVER_EN_ROUTE → IN_PROGRESS → COMPLETED / CANCELLED.
5. Real-time location updates during the trip (for both rider and ops).
6. Pricing: pluggable; supports surge multipliers.

### Non-functional
- **Match latency**: < 5s from request to assignment.
- **Spatial query**: O(log N) for "drivers within R km of point P," with N = drivers in city.
- **No double-bookings**: a driver matched to rider A cannot also be matched to rider B.
- **Many concurrent requests in dense areas** (downtown Friday 8pm).

### Out of scope (mentioned)
- Payment integration internals.
- Routing / map rendering.
- Driver onboarding / KYC.
- Anti-fraud.
- Multi-city federation (extension).

### Core operations
1. `Driver.goOnline(coordinate)` / `updateLocation(coordinate)` / `goOffline()`.
2. `RideService.requestRide(rider, pickup, dropoff, rideType)` → `Trip`.
3. `RideService.cancelTrip(tripId, actorId)`.
4. `RideService.updateTripStatus(tripId, newStatus)` — driver / system actions.

---

## Domain model

### Entities (have ID)
- **`Rider`**, **`Driver`** — users.
- **`Vehicle`** — driver's vehicle (type, capacity, license plate).
- **`Trip`** — aggregate root for trip lifecycle.

### Value objects (immutable)
- **`Coordinate(lat, lon)`** — geographic point.
- **`Money(amount, currency)`**.
- **`Eta(distance, durationMinutes)`**.

### Enums
- `DriverStatus` — `OFFLINE`, `AVAILABLE`, `EN_ROUTE`, `IN_TRIP`.
- `TripStatus` — `REQUESTED`, `MATCHED`, `DRIVER_EN_ROUTE`, `IN_PROGRESS`, `COMPLETED`, `CANCELLED_BY_RIDER`, `CANCELLED_BY_DRIVER`.
- `RideType` — `ECONOMY`, `PREMIUM`, `POOL`, `XL`.

### Patterns
- **Spatial index** — Quadtree, Geohash, or H3 for "drivers near a point."
- **Strategy** — driver matching algorithm; pricing.
- **State** — Trip lifecycle.
- **CAS** — atomic driver state transition (AVAILABLE → MATCHED).
- **Observer** — UI / dispatcher subscribes to driver location updates.
- **Repository** — per entity.

---

## Class diagram

```
┌──────────────────────────────────────┐
│         RideService                  │
├──────────────────────────────────────┤
│ -driverIndex : SpatialIndex<Driver>  │
│ -matcher : DriverMatcher             │
│ -pricing : PricingStrategy           │
│ -tripRepo : TripRepository           │
│ -driverRepo : DriverRepository       │
├──────────────────────────────────────┤
│ +requestRide(rider, pickup, drop)    │
│ +updateDriverLocation(driverId, loc) │
│ +cancelTrip(tripId, actor)           │
└──────────────────────────────────────┘
          │           │           │
          ◇           ◇           ◇
          ▼           ▼           ▼
┌──────────────────┐ ┌──────────────────┐ ┌──────────────────┐
│ <<interface>>    │ │ <<interface>>    │ │ <<interface>>    │
│ SpatialIndex<T>  │ │ DriverMatcher    │ │ PricingStrategy  │
├──────────────────┤ ├──────────────────┤ ├──────────────────┤
│ +put(id, loc)    │ │ +match(req,      │ │ +priceFor(req,   │
│ +remove(id)      │ │   nearby)        │ │   matched)       │
│ +nearby(c, r)    │ │   :Optional<Driv>│ │   : Money        │
└──────────────────┘ └──────────────────┘ └──────────────────┘
        △                  △                    △
   ┌────┴────┐         ┌────┴────────┐    ┌────┴─────┐
┌───────────────┐ ┌──────────────────┐ ┌──────────────────┐
│ GeoHashIndex  │ │ NearestEtaMatcher│ │SurgeMultPricing  │
└───────────────┘ └──────────────────┘ └──────────────────┘
                  ┌──────────────────┐
                  │ FairnessMatcher  │
                  └──────────────────┘

┌──────────────────────────────────┐    ┌────────────────────────┐
│            Driver                │    │         Trip           │
├──────────────────────────────────┤    ├────────────────────────┤
│ -id; -vehicle                    │    │ -id; -riderId          │
│ -location : Coordinate           │    │ -driverId (assigned)   │
│ -status : AtomicRef<DriverStatus>│    │ -pickup, -dropoff      │
│ -rating : double                 │    │ -status : TripStatus   │
└──────────────────────────────────┘    │ -fare : Money          │
                                        └────────────────────────┘
```

---

## Key abstractions

```java
public interface SpatialIndex<T> {
    void put(String id, Coordinate location);
    void remove(String id);
    void update(String id, Coordinate location);
    List<String> nearby(Coordinate center, double radiusKm);
}

public interface DriverMatcher {
    /**
     * Pick the best driver for the request from candidates.
     * Returns empty if no driver suitable.
     */
    Optional<Driver> match(RideRequest req, List<Driver> nearby);
}

public interface PricingStrategy {
    Money priceFor(RideRequest req, Driver matched, Coordinate dropoff);
}
```

---

## Core code

### Value objects + enums

```java
public enum DriverStatus { OFFLINE, AVAILABLE, EN_ROUTE, IN_TRIP }
public enum TripStatus   { REQUESTED, MATCHED, DRIVER_EN_ROUTE, IN_PROGRESS,
                            COMPLETED, CANCELLED_BY_RIDER, CANCELLED_BY_DRIVER }
public enum RideType     { ECONOMY, PREMIUM, POOL, XL }

public record Coordinate(double lat, double lon) {
    public double distanceKmTo(Coordinate other) {
        // Haversine formula; returns great-circle distance.
        double R = 6371.0;
        double dLat = Math.toRadians(other.lat - lat);
        double dLon = Math.toRadians(other.lon - lon);
        double a = Math.sin(dLat/2) * Math.sin(dLat/2) +
                   Math.cos(Math.toRadians(lat)) * Math.cos(Math.toRadians(other.lat)) *
                   Math.sin(dLon/2) * Math.sin(dLon/2);
        return 2 * R * Math.atan2(Math.sqrt(a), Math.sqrt(1 - a));
    }
}

public record RideRequest(
    String requestId,
    String riderId,
    Coordinate pickup,
    Coordinate dropoff,
    RideType type,
    Instant requestedAt
) { }
```

### `Driver`

```java
public final class Driver {
    private final String id;
    private final Vehicle vehicle;
    private volatile Coordinate location;
    private final AtomicReference<DriverStatus> status;
    private final double rating;

    public Driver(String id, Vehicle vehicle, double rating) {
        this.id = id; this.vehicle = vehicle;
        this.status = new AtomicReference<>(DriverStatus.OFFLINE);
        this.rating = rating;
    }

    public String id()                  { return id; }
    public Vehicle vehicle()            { return vehicle; }
    public Coordinate location()        { return location; }
    public DriverStatus status()        { return status.get(); }
    public double rating()              { return rating; }

    public void updateLocation(Coordinate c) { this.location = c; }

    /**
     * Atomic state transition. Returns true if the transition succeeded.
     * Used by the matcher: only the thread that flips AVAILABLE → MATCHED
     * actually gets the driver.
     */
    public boolean transition(DriverStatus from, DriverStatus to) {
        return status.compareAndSet(from, to);
    }
}
```

The CAS on `status` is the load-bearing concurrency primitive: only one thread can flip a driver from `AVAILABLE` to `MATCHED`. Two riders requesting at the same time can't both grab the same driver.

### Spatial index — Geohash-based

A geohash maps a 2D coordinate to a string; nearby coordinates share prefixes. Looking up "nearby" is a prefix-based filter.

```java
public final class GeoHashSpatialIndex<T> implements SpatialIndex<T> {
    /** All entries indexed by geohash prefix. */
    private final ConcurrentHashMap<String, Set<String>> byHash = new ConcurrentHashMap<>();
    /** Reverse index: id → geohash so we can move/remove. */
    private final ConcurrentHashMap<String, String> idToHash = new ConcurrentHashMap<>();
    private final int precision;   // hash length

    public GeoHashSpatialIndex(int precision) { this.precision = precision; }

    @Override
    public void put(String id, Coordinate loc) {
        String hash = GeoHash.encode(loc.lat(), loc.lon(), precision);
        idToHash.put(id, hash);
        byHash.computeIfAbsent(hash, k -> ConcurrentHashMap.newKeySet()).add(id);
    }

    @Override
    public void update(String id, Coordinate loc) {
        String newHash = GeoHash.encode(loc.lat(), loc.lon(), precision);
        String oldHash = idToHash.put(id, newHash);
        if (oldHash != null && !oldHash.equals(newHash)) {
            byHash.getOrDefault(oldHash, Set.of()).remove(id);
            byHash.computeIfAbsent(newHash, k -> ConcurrentHashMap.newKeySet()).add(id);
        }
    }

    @Override
    public void remove(String id) {
        String hash = idToHash.remove(id);
        if (hash != null) byHash.getOrDefault(hash, Set.of()).remove(id);
    }

    @Override
    public List<String> nearby(Coordinate center, double radiusKm) {
        // Look up center's hash + 8 neighbors (covers radius for typical precisions).
        String center_hash = GeoHash.encode(center.lat(), center.lon(), precision);
        List<String> hashesToScan = new ArrayList<>();
        hashesToScan.add(center_hash);
        hashesToScan.addAll(GeoHash.neighbors(center_hash));

        var candidates = new ArrayList<String>();
        for (String h : hashesToScan) {
            Set<String> ids = byHash.getOrDefault(h, Set.of());
            candidates.addAll(ids);
        }
        return candidates;
        // Caller filters by exact distance.
    }
}
```

A precision of 6 chars covers ~1.2km × 0.6km. Tune to typical search radius.

Alternative: **H3** (Uber's open-source library) offers hexagonal cells with uniform area and nicer neighbor semantics.

For very small search areas with extreme density: **R-tree** or **Quadtree** offer O(log N) range queries.

### Driver matcher (Strategy)

```java
public final class NearestEtaMatcher implements DriverMatcher {
    private final EtaService etaService;     // typically uses real road network + traffic

    public NearestEtaMatcher(EtaService etaService) { this.etaService = etaService; }

    @Override
    public Optional<Driver> match(RideRequest req, List<Driver> nearby) {
        // Filter by ride type compatibility; sort by ETA to pickup.
        return nearby.stream()
            .filter(d -> d.status() == DriverStatus.AVAILABLE)
            .filter(d -> isCompatible(d.vehicle(), req.type()))
            .map(d -> new DriverWithEta(d, etaService.eta(d.location(), req.pickup())))
            .min(Comparator.comparingInt(de -> de.eta.durationMinutes()))
            .map(de -> de.driver);
    }

    private boolean isCompatible(Vehicle v, RideType type) {
        return switch (type) {
            case ECONOMY  -> true;   // any vehicle
            case PREMIUM  -> v.tier() == VehicleTier.LUXURY;
            case POOL     -> v.allowsPool();
            case XL       -> v.capacity() >= 5;
        };
    }

    record DriverWithEta(Driver driver, Eta eta) { }
}
```

Other matchers worth mentioning:
- **FairnessMatcher**: prefers drivers with fewer trips today (load balancing).
- **EnergyOptimalMatcher**: minimizes total deadhead miles across the city.
- **SurgeAwareMatcher**: in surge zones, prefers drivers willing to drive into surge.

### Pricing (Strategy)

```java
public final class SurgeMultiplierPricing implements PricingStrategy {
    private final BigDecimal baseFare;
    private final BigDecimal perKm;
    private final BigDecimal perMinute;
    private final SurgeService surgeService;

    public SurgeMultiplierPricing(BigDecimal baseFare, BigDecimal perKm, BigDecimal perMinute, SurgeService surge) {
        this.baseFare = baseFare; this.perKm = perKm; this.perMinute = perMinute;
        this.surgeService = surge;
    }

    @Override
    public Money priceFor(RideRequest req, Driver matched, Coordinate dropoff) {
        Eta eta = matched.location().distanceKmTo(req.pickup()) > 0
            ? new Eta(req.pickup().distanceKmTo(dropoff), 0)   // simplified
            : null;

        BigDecimal distance = BigDecimal.valueOf(req.pickup().distanceKmTo(dropoff));
        BigDecimal fareWithoutSurge = baseFare
            .add(distance.multiply(perKm));
            // (skipping per-minute for brevity)

        BigDecimal surge = surgeService.multiplierFor(req.pickup(), Instant.now());
        return new Money(fareWithoutSurge.multiply(surge), Currency.getInstance("USD"));
    }
}

public interface SurgeService {
    /** Returns 1.0 in normal conditions; 1.5+ in surge zones. */
    BigDecimal multiplierFor(Coordinate location, Instant time);
}
```

`SurgeService` reads recent demand/supply ratios per zone; output multiplier feeds into the price.

### `RideService` — orchestrator

```java
public final class RideService {
    private final SpatialIndex<Driver> driverIndex;
    private final DriverMatcher matcher;
    private final PricingStrategy pricing;
    private final TripRepository tripRepo;
    private final DriverRepository driverRepo;
    private final EventBus eventBus;
    private final Clock clock;
    private final IdGenerator ids;
    private final double searchRadiusKm = 5.0;

    public RideService(SpatialIndex<Driver> idx, DriverMatcher m, PricingStrategy p,
                       TripRepository tr, DriverRepository dr, EventBus eb,
                       Clock clock, IdGenerator ids) {
        this.driverIndex = idx; this.matcher = m; this.pricing = p;
        this.tripRepo = tr; this.driverRepo = dr; this.eventBus = eb;
        this.clock = clock; this.ids = ids;
    }

    public Trip requestRide(String riderId, Coordinate pickup, Coordinate dropoff, RideType type) {
        RideRequest req = new RideRequest(ids.next(), riderId, pickup, dropoff, type, clock.instant());

        // Match with retry: if the chosen driver loses CAS, try the next-best.
        for (int attempt = 0; attempt < 5; attempt++) {
            List<String> nearbyIds = driverIndex.nearby(pickup, searchRadiusKm);
            List<Driver> candidates = nearbyIds.stream()
                .map(driverRepo::findById)
                .flatMap(Optional::stream)
                .filter(d -> d.status() == DriverStatus.AVAILABLE)
                .filter(d -> d.location().distanceKmTo(pickup) <= searchRadiusKm)
                .toList();

            Optional<Driver> chosen = matcher.match(req, candidates);
            if (chosen.isEmpty()) throw new NoDriversAvailableException();

            Driver d = chosen.get();
            // Atomic claim
            if (d.transition(DriverStatus.AVAILABLE, DriverStatus.EN_ROUTE)) {
                Money fare = pricing.priceFor(req, d, dropoff);
                Trip trip = new Trip(ids.next(), riderId, d.id(), pickup, dropoff, fare, clock.instant());
                tripRepo.save(trip);
                eventBus.publish(new TripMatchedEvent(trip.id(), d.id()));
                return trip;
            }
            // Lost the CAS — driver was claimed by another rider; retry
        }
        throw new MatchingContentionException();
    }

    public void updateDriverLocation(String driverId, Coordinate loc) {
        Driver d = driverRepo.findById(driverId).orElseThrow();
        d.updateLocation(loc);
        if (d.status() == DriverStatus.AVAILABLE) {
            driverIndex.update(driverId, loc);
        }
        // For drivers in trip: update separate trip-tracking index, not the matching index.
    }

    public void cancelTrip(String tripId, String actorId) {
        Trip trip = tripRepo.findById(tripId).orElseThrow();
        boolean isRider = actorId.equals(trip.riderId());
        boolean isDriver = actorId.equals(trip.driverId());
        if (!isRider && !isDriver) throw new ForbiddenException();

        if (trip.status() == TripStatus.IN_PROGRESS || trip.status() == TripStatus.COMPLETED)
            throw new IllegalStateException("Cannot cancel: " + trip.status());

        trip.cancel(isRider ? TripStatus.CANCELLED_BY_RIDER : TripStatus.CANCELLED_BY_DRIVER);
        tripRepo.save(trip);

        // Free the driver
        Driver d = driverRepo.findById(trip.driverId()).orElseThrow();
        d.transition(DriverStatus.EN_ROUTE, DriverStatus.AVAILABLE);
        driverIndex.put(d.id(), d.location());
    }

    public void driverArrived(String tripId) {
        Trip t = tripRepo.findById(tripId).orElseThrow();
        t.transitionTo(TripStatus.IN_PROGRESS);
        tripRepo.save(t);
        Driver d = driverRepo.findById(t.driverId()).orElseThrow();
        d.transition(DriverStatus.EN_ROUTE, DriverStatus.IN_TRIP);
    }

    public void completeTrip(String tripId, Money finalFare) {
        Trip t = tripRepo.findById(tripId).orElseThrow();
        t.complete(finalFare);
        tripRepo.save(t);
        Driver d = driverRepo.findById(t.driverId()).orElseThrow();
        d.transition(DriverStatus.IN_TRIP, DriverStatus.AVAILABLE);
        driverIndex.put(d.id(), d.location());
    }
}
```

### `Trip`

```java
public final class Trip {
    private final String id;
    private final String riderId;
    private final String driverId;
    private final Coordinate pickup;
    private final Coordinate dropoff;
    private Money fare;
    private TripStatus status;
    private final Instant createdAt;
    private Instant completedAt;

    public Trip(String id, String riderId, String driverId, Coordinate pickup, Coordinate dropoff,
                Money fare, Instant createdAt) {
        this.id = id; this.riderId = riderId; this.driverId = driverId;
        this.pickup = pickup; this.dropoff = dropoff; this.fare = fare;
        this.createdAt = createdAt;
        this.status = TripStatus.MATCHED;
    }

    public synchronized void transitionTo(TripStatus next) {
        if (!isValidTransition(this.status, next))
            throw new IllegalStateException("Cannot " + next + " from " + status);
        this.status = next;
    }

    public synchronized void complete(Money finalFare) {
        transitionTo(TripStatus.COMPLETED);
        this.fare = finalFare;
        this.completedAt = Instant.now();
    }

    public synchronized void cancel(TripStatus cancelledBy) {
        if (status == TripStatus.COMPLETED) throw new IllegalStateException();
        this.status = cancelledBy;
    }

    private boolean isValidTransition(TripStatus from, TripStatus to) {
        return switch (from) {
            case REQUESTED         -> to == TripStatus.MATCHED || to == TripStatus.CANCELLED_BY_RIDER;
            case MATCHED           -> to == TripStatus.DRIVER_EN_ROUTE
                                   || to == TripStatus.CANCELLED_BY_RIDER || to == TripStatus.CANCELLED_BY_DRIVER;
            case DRIVER_EN_ROUTE   -> to == TripStatus.IN_PROGRESS
                                   || to == TripStatus.CANCELLED_BY_RIDER || to == TripStatus.CANCELLED_BY_DRIVER;
            case IN_PROGRESS       -> to == TripStatus.COMPLETED;
            default                -> false;
        };
    }

    public String id() { return id; }
    public String riderId() { return riderId; }
    public String driverId() { return driverId; }
    public TripStatus status() { return status; }
    public Money fare() { return fare; }
}
```

---

## Concurrency strategy

### What's contested
- **Driver state** (`AVAILABLE` → `EN_ROUTE`) — central to "no double-booking."
- **Driver location index** — many concurrent updates; many concurrent reads.
- **Trip state** — driver and rider can both attempt transitions.

### Strategy
- **`AtomicReference<DriverStatus>` with CAS** — only one thread can claim a driver. The matcher's "find best driver" is read-only; the actual claim happens via CAS. On CAS miss → retry with the next-best driver.
- **`ConcurrentHashMap` for spatial index** — concurrent puts/gets/updates per geohash cell.
- **Per-trip `synchronized`** — trip state transitions atomic.

### Why CAS over per-driver locks?
- Drivers are independent — locking per-driver works, but adds memory and context-switching overhead.
- CAS is a single atomic instruction; loser retries with another candidate.
- For a city with 10,000 drivers and 1,000 concurrent matching requests, CAS is dramatically faster than locks.

### "Two riders, same driver" race
- Both riders' matchers look up the same driver as the best candidate.
- Both call `d.transition(AVAILABLE, EN_ROUTE)`.
- CAS allows exactly one to succeed. The other retries with a different driver.
- The retry path does another `nearby` query — driver state may have changed; new candidates considered.

### Location updates during matching
- Matcher reads `nearby` from spatial index → reads driver locations.
- Driver location can change during this read window.
- Acceptable: if a driver moves slightly between the read and the assignment, the match is still good enough. The approximate-but-fast model wins over the exact-but-slow.

### Trip state under concurrent driver/rider actions
- Per-trip `synchronized` ensures a single transition wins.
- e.g., rider cancels at the same time driver marks "arrived": one wins; the other gets `IllegalStateException`.

---

## Edge cases / gotchas

| Case | Handling |
|---|---|
| **No drivers nearby** | Matcher returns empty → service throws `NoDriversAvailable`. UI shows "no cars in area." |
| **Driver goes offline mid-match** | CAS fails (status moved to OFFLINE) → matcher retries with next candidate. |
| **Driver disconnects mid-trip** | Periodic heartbeat; if missed for N seconds, trip flagged for review; rider notified. |
| **Rider cancels right as driver arrives** | Trip state's `synchronized` ensures one transition wins. The loser sees an exception and handles it gracefully (driver UI shows "rider cancelled"). |
| **Multiple riders requesting in surge zone simultaneously** | Each request sees high prices (surge multiplier read at request time); matched independently; supply runs out → `NoDriversAvailable`. |
| **Driver location updates during trip** | Update goes to a separate trip-tracking store, not the matching index. Matching index only contains `AVAILABLE` drivers. |
| **Pricing changes mid-request** | Lock the price at trip-creation time. The trip's `fare` field is the agreed price. |
| **Trip duration much longer than estimated (traffic)** | Driver adjusts; final fare includes per-minute charges in many models. |
| **Rider books trip for someone else (gift ride)** | Out of LLD scope but the trip model has `riderId` separate from `payerId`. |
| **Driver picks up wrong rider** | Verification step (PIN, license plate match) before `IN_PROGRESS`. |

---

## Extensions / interview follow-ups

### "Add Pool / shared rides."
- New `RideType.POOL`.
- A trip can match with another trip in flight: pickup overlap + same general direction.
- Matcher's pool variant looks for in-progress trips first; if no pool match, treat as solo.

### "Add scheduled rides (book for tomorrow at 8am)."
- New entity: `ScheduledTrip`.
- Background scheduler matches near the time (e.g., 30 min before).
- ETA-aware: pre-positions a driver.

### "Add a heat map for drivers."
- Aggregate request density per zone over time.
- Drivers see hot zones; navigate toward demand.
- Engine-side analytics, not LLD.

### "Why a spatial index and not a SQL geo query?"
- PostGIS or MySQL spatial extensions can do "find within radius" too.
- Performance: in-memory geohash is O(1) for the bucket lookup + filter.
- DB-backed works at moderate scale; in-memory wins for hot dispatch.
- Real systems use both: DB for source of truth + in-memory cache for matching.

### "How do you handle drivers across cities?"
- City partitioning: each city has its own dispatch service.
- Cross-city is rare (long trips); handled separately.
- Driver registers to a "home city"; can roam temporarily.

### "What's the trade-off of geohash vs H3 vs quadtree?"
- **Geohash**: prefix-based; rectangular cells; widely used; uneven cell area at high latitudes.
- **H3**: hexagonal; uniform cell area; better neighbor semantics.
- **Quadtree**: dynamic refinement; great for unbalanced distributions.
- For Uber-like systems, **H3** is the modern choice; **geohash** is the classic.

### "How do you prevent the same driver showing up in two riders' results?"
- The driver index returns IDs; matcher reads them; CAS on status.
- The CAS is the serialization point — only one rider can transition the driver out of `AVAILABLE`.

### "What if matching takes >5s under heavy load?"
- Bound matching at 5s; fail fast with "no drivers."
- Async match queue: rider request enters queue; processed in order; user sees "looking for driver" with cancel option.

---

## Scaling

A single dispatch service handles a single city well. For multi-city, multi-region:

### Per-city dispatch
- Each city runs its own dispatch service with its own spatial index.
- Driver and trip data persisted per-city.
- Cross-city is rare → out of the hot path.

### Driver location ingestion at scale
- Drivers update every 5s; with 100k online drivers, that's 20k updates/sec per city.
- WebSocket / gRPC bidirectional stream from app to dispatch.
- Updates fan into the spatial index via a queue (don't block dispatch on update writes).

### Surge calculation
- Per-zone demand (request rate) and supply (driver count) updated every minute.
- Stream processing (Kafka + Flink) computes the multiplier.
- Dispatch reads from a fast cache (Redis with the latest multipliers).

### Trip data warehousing
- Live trip state in PostgreSQL.
- Completed trips ETL'd to a warehouse for analytics.

### Real-time updates to apps
- Driver location → rider's app: WebSocket pub/sub keyed by trip ID.
- Trip status changes broadcast to both.

---

## Persistence tradeoffs

The data shapes here are diverse — hot driver state, append-mostly trip records, time-series locations, real-time spatial index.

### What persistent data exists
- **Drivers** (id, vehicle, profile, current status). Hot.
- **Riders** (id, profile). Mostly read.
- **Driver locations** — high-frequency updates. Live state in memory; archival in a time-series store.
- **Trips** — append-mostly with state transitions during the trip.
- **Trip events** (location updates per trip, status changes) — append-only.

### Option 1 — Polyglot (production answer)
- **PostgreSQL** for users, vehicles, trips, payments. Source of truth for relational data.
- **Redis** for hot driver status + spatial index (Redis has GEO commands: `GEOADD`, `GEORADIUS`).
- **Kafka** for the location update stream. Consumers update Redis index, archive to S3/warehouse.
- **Cassandra / time-series DB** for trip event archive (location-per-second per trip).
- **Elasticsearch** for analytics on past trips.

### Option 2 — Simpler: PostgreSQL + Redis only
- PostgreSQL: users, trips, all OLTP.
- Redis: spatial index + driver state.
- Skip the Kafka / Cassandra tier for moderate scale.

### Option 3 — DynamoDB + DAX
- Cloud-native; partition by city; global tables for multi-region.
- DynamoDB Streams feed downstream (analytics, search).

### Spatial index — Redis GEO?
Redis has native geospatial commands:
- `GEOADD city-drivers lon lat driver-id` — store driver location.
- `GEOSEARCH city-drivers FROMLONLAT lon lat BYRADIUS 5 km` — find drivers within 5km.
- Atomic; well-tested; sub-millisecond.

For most production systems, **Redis GEO** is the right choice for the spatial index — it removes the need to hand-roll geohash-based code. The hand-rolled code in this design is for interview clarity.

### Driver state on Redis vs in-memory
- **In-memory**: faster (ns), but lost on restart. Bootstrap from DB on startup.
- **Redis**: ms latency, but durable across instance restarts. Single source of truth across instances.
- **Hybrid**: Redis as truth; in-memory cache with TTL. Cache invalidated by Redis pub/sub on state changes.

### Trip state under concurrency
- Trip rows have `version`; updates are conditional on version.
- High-frequency updates during a trip (location updates) go to a separate trip-events table.

### Concurrency primitives across stores
| Concern | In-memory | Redis | DynamoDB | PostgreSQL |
|---|---|---|---|---|
| Driver state CAS | `AtomicReference` | Lua script | conditional `UpdateItem` | `UPDATE ... WHERE status = ? AND version = ?` |
| Spatial index | hand-rolled | `GEOADD` / `GEOSEARCH` | (geo via secondary) | PostGIS |
| Trip state | `synchronized` | Lua | conditional update | row version |

### Recommendation
- **Default**: PostgreSQL + Redis (spatial + hot state).
- **At scale**: Add Kafka for ingestion + Cassandra for trip events archive.
- **Cloud-managed**: DynamoDB + ElastiCache + Kinesis.

---

## Talking points for the interview

- "Driver matching is a CAS on driver status — the search-and-match are read-only, the claim is the atomic step. Two riders racing on the same driver: one wins via CAS, the other retries."
- "Spatial index is the load-bearing data structure. In-memory geohash for the interview; Redis GEO for production."
- "Driver location updates during matching are tolerated — slight movement between read and assignment is acceptable; absolute precision isn't needed."
- "Trip state has a state machine — transitions enforce ordering (can't go IN_PROGRESS without first MATCHED + DRIVER_EN_ROUTE)."
- "Surge pricing locks the multiplier at request time — if surge drops between request and trip start, the rider still pays surge. (Different platforms make different choices here.)"
- "City-level partitioning is the natural sharding boundary. Each city's dispatch is independent."
- "For real-time location to riders: WebSocket pub/sub keyed by trip ID. Doesn't go through the matching index."

---

## Summary

Patterns load-bearing: **spatial index** (geohash / H3 / Redis GEO), **CAS on driver state** (atomic assignment), **Strategy** (matcher, pricing), **State** (trip lifecycle).

The mental model:
1. Drivers update their location; spatial index reflects.
2. Rider requests → spatial index returns nearby drivers.
3. Matcher picks the best; CAS claims that driver.
4. CAS miss → retry with the next-best.
5. Trip is a state machine; transitions guarded.

The senior signals here: knowing geohash (or recommending Redis GEO), reaching for CAS on driver status (not a per-driver lock), city-level partitioning, and recognizing that small lookups-vs-claim race windows are fine because the CAS is the serialization point.
