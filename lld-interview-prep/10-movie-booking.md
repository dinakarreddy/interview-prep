# Problem 10 — Movie Ticket Booking (BookMyShow-style)

The canonical hold-then-confirm + optimistic-locking problem. Tests your handling of inventory under contention, two-phase commit-style flows, and time-bounded reservations.

## Interview opener

> **Interviewer:** Design a movie ticket booking system.

---

## Clarification phase

> **You:** Users browse movies, pick a show, pick seats, pay, get tickets?
>
> **Interviewer:** Yes.

> **You:** Multi-city, multi-theater, multi-screen?
>
> **Interviewer:** Yes — assume the standard hierarchy.

> **You:** Seat layout — should I support different layouts (IMAX, recliner, regular)?
>
> **Interviewer:** Yes; some seats cost more. Modeling support, not implementation, is fine.

> **You:** What happens when two users try to book the same seats at the same time?
>
> **Interviewer:** That's the core problem — only one wins. The other gets a clear error.

> **You:** Should a user be able to "hold" seats while completing payment?
>
> **Interviewer:** Yes — exactly. Hold expires after a few minutes if not paid.

> **You:** Pricing — flat per show, or vary by seat type, day, demand?
>
> **Interviewer:** Configurable. At least seat-type-aware; mention surge / weekend pricing.

> **You:** Cancellation / refund policy?
>
> **Interviewer:** Allow cancel until 30 minutes before show; partial refund.

> **You:** Group bookings — atomic across seats?
>
> **Interviewer:** Yes — all-or-nothing for a single booking.

> **You:** Payment processing details?
>
> **Interviewer:** Out of scope; assume a `PaymentService` exists.

> **You:** Persistence?
>
> **Interviewer:** Discuss at end.

OK — design needs: hierarchical browsing, hold-then-confirm with TTL, concurrent seat conflict resolution, pricing Strategy.

---

## Refined requirements

### Functional
1. Browse: cities → theaters → movies → shows.
2. Pick show; see seat availability.
3. **Hold seats** for N minutes — exclusive across users.
4. Pay; confirm booking; ticket issued.
5. Cancel before deadline; refund.
6. Auto-release expired holds.

### Non-functional
- **Concurrency**: two users picking the same seats — exactly one wins.
- **Atomicity**: a booking is all seats or none.
- **Latency**: seat-availability fetch in O(seats) or better.
- **TTL on holds**: 5–10 minutes; cleanly released on expiry.

### Out of scope (mentioned)
- Payment integration internals.
- Recommendation / personalization.
- Reviews / ratings.
- Multi-currency.

### Core operations
1. `searchShows(city, movieId, date)` → `List<Show>`
2. `holdSeats(showId, seatIds, userId)` → `SeatHold` (with `expiresAt`)
3. `confirmBooking(holdId, paymentToken)` → `Booking`
4. `cancelBooking(bookingId, userId)` → `Refund`
5. (Background) sweep expired holds.

---

## Domain model

### Entities (have ID)
- **`Movie`**: title, duration, language, genres.
- **`Theater`**: name, city, address; owns `Screen`s.
- **`Screen`**: physical room; owns seat layout.
- **`Show`**: aggregate root for booking. Movie + Screen + start time + price plan.
- **`Booking`**: confirmed reservation. References show + seats + user.
- **`SeatHold`**: aggregate root for the hold lifecycle. References show + seats + user + expiresAt.

### Value objects (immutable)
- **`Seat(row, col, type)`**: type = REGULAR / PREMIUM / RECLINER.
- **`Money(amount, currency)`**.

### Enums
- `BookingStatus` — `PENDING_HOLD`, `CONFIRMED`, `CANCELLED`, `REFUNDED`.
- `SeatType` — `REGULAR`, `PREMIUM`, `RECLINER`, `IMAX`.
- `HoldStatus` — `ACTIVE`, `CONFIRMED`, `EXPIRED`, `CANCELLED`.

### Why these aggregate boundaries
- **Show** is the unit of contention — a seat is occupied within the context of a specific show.
- **SeatHold** is its own aggregate root because its lifecycle (TTL-bound) is independent of `Booking`'s.
- Cross-aggregate references via ID (`Booking.showId`, not direct reference).

### Patterns
- **Repository** — per entity.
- **Strategy** — pricing rules (per seat type, per show time, per demand).
- **State** — Booking lifecycle.
- **Two-phase commit** (logical) — hold then confirm.
- **Optimistic locking** — version on the show's seat-state to detect concurrent changes.

---

## Class diagram

```
┌──────────────────────┐    ┌──────────────────────┐
│        Movie         │    │       Theater        │
├──────────────────────┤    ├──────────────────────┤
│ -id; -title; -mins   │    │ -id; -name; -city    │
└──────────────────────┘    └──────────────────────┘
                                     │ ◆ 1..*
                                     ▼
                            ┌──────────────────────┐
                            │       Screen         │
                            ├──────────────────────┤
                            │ -id                  │
                            │ -seatLayout: Seat[][]│
                            └──────────────────────┘

┌──────────────────────────────────┐
│            Show                  │
├──────────────────────────────────┤
│ -id : String                     │
│ -movieId; -screenId              │
│ -startTime : Instant             │
│ -basePrice : Money               │
│ -pricingStrategy : PricingStrategy│ ◇──► PricingStrategy
│ -seatStates : Map<Seat, SeatState>│
│ -version : long                  │
└──────────────────────────────────┘

┌──────────────────────────────────┐    ┌──────────────────────────┐
│         SeatHold                 │    │         Booking          │
├──────────────────────────────────┤    ├──────────────────────────┤
│ -id; -showId; -userId            │    │ -id; -showId; -userId    │
│ -seats : List<Seat>              │    │ -seats : List<Seat>      │
│ -expiresAt : Instant             │    │ -totalPrice : Money      │
│ -status : HoldStatus             │    │ -status : BookingStatus  │
└──────────────────────────────────┘    │ -paymentRef : String     │
                                        └──────────────────────────┘

┌─────────────────────────┐
│   <<interface>>          │
│   PricingStrategy        │
├─────────────────────────┤
│ +priceFor(show, seats)   │
│   : Money                │
└─────────────────────────┘
        △
   ┌────┼─────────────────┐
┌─────────────┐ ┌────────────────┐
│SeatTypeBased│ │WeekendSurge    │
│ Pricing     │ │ Pricing        │
└─────────────┘ └────────────────┘

   record Seat(int row, int col, SeatType type)
   enum SeatState { AVAILABLE, HELD, BOOKED }
```

---

## Key abstractions

```java
public interface PricingStrategy {
    Money priceFor(Show show, List<Seat> seats);
}

public interface ShowRepository {
    Optional<Show> findById(String id);
    List<Show> findByCityAndMovie(String city, String movieId, LocalDate date);
    /** Optimistic save — version-aware. */
    boolean save(Show show, long expectedVersion);
}

public interface SeatHoldRepository {
    void save(SeatHold hold);
    Optional<SeatHold> findById(String id);
    List<SeatHold> findExpired(Instant now);
    void delete(String id);
}

public interface BookingRepository {
    void save(Booking b);
    Optional<Booking> findById(String id);
    List<Booking> findByUser(String userId);
}

public interface PaymentService {
    PaymentResult charge(String userId, Money amount, String idempotencyKey);
    void refund(String paymentRef, Money amount, String idempotencyKey);
}
```

---

## Core code

### Value objects + enums

```java
public enum SeatType    { REGULAR, PREMIUM, RECLINER, IMAX }
public enum SeatState   { AVAILABLE, HELD, BOOKED }
public enum HoldStatus  { ACTIVE, CONFIRMED, EXPIRED, CANCELLED }
public enum BookingStatus { CONFIRMED, CANCELLED, REFUNDED }

public record Seat(int row, int col, SeatType type) { }
public record PaymentResult(boolean success, String paymentRef, Money charged) { }
```

### `Show`

```java
public final class Show {
    private final String id;
    private final String movieId;
    private final String screenId;
    private final Instant startTime;
    private final Money basePrice;

    /** Per-seat state. Concurrent reads/writes — see locking discussion. */
    private final Map<Seat, SeatState> seatStates;
    private long version;

    public Show(String id, String movieId, String screenId, Instant startTime,
                Money basePrice, List<Seat> allSeats) {
        this.id = id; this.movieId = movieId; this.screenId = screenId;
        this.startTime = startTime; this.basePrice = basePrice;
        this.seatStates = new ConcurrentHashMap<>();
        for (Seat s : allSeats) seatStates.put(s, SeatState.AVAILABLE);
    }

    public String id()                  { return id; }
    public String movieId()             { return movieId; }
    public Instant startTime()          { return startTime; }
    public Money basePrice()            { return basePrice; }
    public long version()               { return version; }

    /**
     * Atomically reserve seats. Returns true only if ALL seats were AVAILABLE
     * and are now marked HELD; false otherwise (and any partial changes reverted).
     *
     * The CAS pattern is at the per-seat level, then version-bumped.
     */
    public synchronized boolean tryHoldSeats(List<Seat> seats) {
        // Verify all available before mutating
        for (Seat s : seats) {
            SeatState state = seatStates.get(s);
            if (state == null) throw new IllegalArgumentException("No such seat: " + s);
            if (state != SeatState.AVAILABLE) return false;
        }
        // All available — mark held
        for (Seat s : seats) seatStates.put(s, SeatState.HELD);
        version++;
        return true;
    }

    public synchronized void confirmHeldSeats(List<Seat> seats) {
        for (Seat s : seats) {
            if (seatStates.get(s) != SeatState.HELD)
                throw new IllegalStateException("Seat " + s + " not in HELD state");
            seatStates.put(s, SeatState.BOOKED);
        }
        version++;
    }

    public synchronized void releaseHeldSeats(List<Seat> seats) {
        for (Seat s : seats) {
            // Idempotent — only release if currently held
            if (seatStates.get(s) == SeatState.HELD) seatStates.put(s, SeatState.AVAILABLE);
        }
        version++;
    }

    public synchronized void releaseBookedSeats(List<Seat> seats) {
        for (Seat s : seats) {
            if (seatStates.get(s) == SeatState.BOOKED) seatStates.put(s, SeatState.AVAILABLE);
        }
        version++;
    }

    public Map<Seat, SeatState> seatSnapshot() { return Map.copyOf(seatStates); }
}
```

The synchronized methods on `Show` make seat-state transitions atomic *for one show*. Different shows are independent — no global lock.

### `SeatHold`

```java
public final class SeatHold {
    private final String id;
    private final String showId;
    private final String userId;
    private final List<Seat> seats;
    private final Instant createdAt;
    private final Instant expiresAt;
    private volatile HoldStatus status;

    public SeatHold(String id, String showId, String userId, List<Seat> seats,
                    Instant createdAt, Duration ttl) {
        this.id = id; this.showId = showId; this.userId = userId;
        this.seats = List.copyOf(seats);
        this.createdAt = createdAt;
        this.expiresAt = createdAt.plus(ttl);
        this.status = HoldStatus.ACTIVE;
    }

    public String id()                 { return id; }
    public String showId()             { return showId; }
    public String userId()             { return userId; }
    public List<Seat> seats()          { return seats; }
    public Instant expiresAt()         { return expiresAt; }
    public HoldStatus status()         { return status; }

    public boolean isExpired(Instant now) { return now.isAfter(expiresAt); }

    public synchronized void confirm() {
        ensureActive();
        status = HoldStatus.CONFIRMED;
    }

    public synchronized void cancel() {
        ensureActive();
        status = HoldStatus.CANCELLED;
    }

    public synchronized void markExpired() {
        if (status == HoldStatus.ACTIVE) status = HoldStatus.EXPIRED;
    }

    private void ensureActive() {
        if (status != HoldStatus.ACTIVE)
            throw new IllegalStateException("Hold not active: " + status);
    }
}
```

### Pricing — Strategy

```java
public final class SeatTypePricing implements PricingStrategy {
    private final Map<SeatType, BigDecimal> multipliers;

    public SeatTypePricing(Map<SeatType, BigDecimal> multipliers) {
        this.multipliers = Map.copyOf(multipliers);
    }

    @Override
    public Money priceFor(Show show, List<Seat> seats) {
        BigDecimal total = BigDecimal.ZERO;
        for (Seat s : seats) {
            BigDecimal mult = multipliers.getOrDefault(s.type(), BigDecimal.ONE);
            total = total.add(show.basePrice().amount().multiply(mult));
        }
        return new Money(total, show.basePrice().currency());
    }
}

public final class WeekendSurgePricing implements PricingStrategy {
    private final PricingStrategy base;
    private final BigDecimal weekendMultiplier;

    public WeekendSurgePricing(PricingStrategy base, BigDecimal weekendMultiplier) {
        this.base = base; this.weekendMultiplier = weekendMultiplier;
    }

    @Override
    public Money priceFor(Show show, List<Seat> seats) {
        Money basePrice = base.priceFor(show, seats);
        DayOfWeek d = show.startTime().atZone(ZoneOffset.UTC).getDayOfWeek();
        if (d == DayOfWeek.SATURDAY || d == DayOfWeek.SUNDAY) {
            return new Money(basePrice.amount().multiply(weekendMultiplier), basePrice.currency());
        }
        return basePrice;
    }
}
```

`WeekendSurgePricing` is a Decorator — it wraps another `PricingStrategy` and adds weekend surge.

### `BookingService` — orchestrator

```java
public final class BookingService {
    private final ShowRepository showRepo;
    private final SeatHoldRepository holdRepo;
    private final BookingRepository bookingRepo;
    private final PricingStrategy pricing;
    private final PaymentService paymentService;
    private final Clock clock;
    private final IdGenerator ids;
    private final Duration holdTtl;

    public BookingService(ShowRepository sr, SeatHoldRepository hr, BookingRepository br,
                          PricingStrategy pricing, PaymentService ps,
                          Clock clock, IdGenerator ids, Duration holdTtl) {
        this.showRepo = sr; this.holdRepo = hr; this.bookingRepo = br;
        this.pricing = pricing; this.paymentService = ps;
        this.clock = clock; this.ids = ids; this.holdTtl = holdTtl;
    }

    public SeatHold holdSeats(String showId, List<Seat> seats, String userId) {
        Show show = showRepo.findById(showId)
            .orElseThrow(() -> new ShowNotFoundException(showId));

        if (!show.tryHoldSeats(seats)) {
            throw new SeatsUnavailableException(seats);
        }

        SeatHold hold = new SeatHold(ids.next(), showId, userId, seats, clock.instant(), holdTtl);
        try {
            holdRepo.save(hold);
        } catch (Exception e) {
            // Save failed — must release the seats to avoid leaking holds
            show.releaseHeldSeats(seats);
            throw e;
        }
        return hold;
    }

    public Booking confirmBooking(String holdId, String paymentToken) {
        SeatHold hold = holdRepo.findById(holdId)
            .orElseThrow(() -> new HoldNotFoundException(holdId));

        if (hold.isExpired(clock.instant())) {
            // Hold expired — release seats and reject
            Show show = showRepo.findById(hold.showId()).orElseThrow();
            show.releaseHeldSeats(hold.seats());
            hold.markExpired();
            holdRepo.save(hold);
            throw new HoldExpiredException(holdId);
        }

        Show show = showRepo.findById(hold.showId()).orElseThrow();
        Money total = pricing.priceFor(show, hold.seats());

        // Idempotency key: hold id + version → safe to retry on transient failure
        String idempotencyKey = "booking:" + holdId;
        PaymentResult payment = paymentService.charge(hold.userId(), total, idempotencyKey);
        if (!payment.success()) throw new PaymentFailedException();

        try {
            show.confirmHeldSeats(hold.seats());
        } catch (Exception e) {
            // Refund and rethrow
            paymentService.refund(payment.paymentRef(), total, idempotencyKey + ":refund");
            throw e;
        }

        hold.confirm();
        holdRepo.save(hold);

        Booking booking = new Booking(
            ids.next(), hold.showId(), hold.userId(), hold.seats(),
            total, payment.paymentRef(), BookingStatus.CONFIRMED, clock.instant()
        );
        bookingRepo.save(booking);
        return booking;
    }

    public void cancelBooking(String bookingId, String userId) {
        Booking booking = bookingRepo.findById(bookingId)
            .orElseThrow(() -> new BookingNotFoundException(bookingId));
        if (!booking.userId().equals(userId)) throw new ForbiddenException();
        if (booking.status() != BookingStatus.CONFIRMED)
            throw new IllegalStateException("Cannot cancel: " + booking.status());

        Show show = showRepo.findById(booking.showId()).orElseThrow();
        // Refund window check
        if (Duration.between(clock.instant(), show.startTime()).toMinutes() < 30)
            throw new RefundWindowClosedException();

        // Refund first
        Money refund = booking.totalPrice();
        paymentService.refund(booking.paymentRef(), refund, "refund:" + bookingId);

        // Release seats
        show.releaseBookedSeats(booking.seats());
        booking.markCancelled();
        bookingRepo.save(booking);
    }

    /** Background sweeper. Run every minute. */
    public void sweepExpiredHolds() {
        Instant now = clock.instant();
        for (SeatHold hold : holdRepo.findExpired(now)) {
            if (hold.status() != HoldStatus.ACTIVE) continue;
            Show show = showRepo.findById(hold.showId()).orElse(null);
            if (show != null) show.releaseHeldSeats(hold.seats());
            hold.markExpired();
            holdRepo.save(hold);
        }
    }
}
```

The flow: `tryHoldSeats` is atomic for the whole list. If any seat is unavailable, none are mutated, and `holdSeats` throws. The hold is created only after seats are claimed.

### Background sweeper (runs periodically)

```java
public final class HoldExpirySweeper {
    private final BookingService bookingService;
    private final ScheduledExecutorService scheduler = Executors.newSingleThreadScheduledExecutor();

    public HoldExpirySweeper(BookingService bs) { this.bookingService = bs; }

    public void start() {
        scheduler.scheduleAtFixedRate(this::tick, 0, 60, TimeUnit.SECONDS);
    }

    private void tick() {
        try { bookingService.sweepExpiredHolds(); }
        catch (Exception e) { /* log; never let exceptions kill the scheduler */ }
    }
}
```

In production the sweeper would be its own deployment, with a leader-election / single-runner constraint.

---

## Concurrency strategy

### What's contested
- Seat states for a given show (multiple users selecting overlapping seats).
- The hold/booking repositories.

### Strategy
- **Per-show synchronization** for seat-state mutation. Different shows are independent.
- **Atomic seat-list reservation**: `tryHoldSeats` either reserves all or none. No partial state.
- **Pessimistic vs optimistic** within the show: we use the show object's monitor (pessimistic). For a persistent backend, optimistic locking with a `version` column is preferred (`UPDATE shows SET seat_states=..., version=version+1 WHERE id=? AND version=?`).
- **Holds are time-bounded**: even if a buggy or slow client never confirms, the sweeper releases seats.

### Why not row-level locking per seat?
- A booking spans multiple seats; needing a multi-seat atomic reservation makes per-seat locks deadlock-prone (two users grabbing seats {A, B} and {B, A} respectively).
- Per-show locking is simpler: order seat IDs canonically and lock once.
- For sharding by show, this is fine: each show fits on one shard / one node.

### Deadlock resistance
- We hold one lock (the show's monitor) and don't acquire other locks while holding it (no I/O — except `holdRepo.save`, which is a separate transaction).
- The `try/catch` around `holdRepo.save` cleanly releases seats on save failure.

### The "hold expired but user just paid" race
- User holds 2 seats; clicks pay at minute 4:59; payment processes at minute 5:01; hold is expired by sweeper at minute 5:00.
- Resolution: the payment service idempotency key + the hold's `confirm()` method's status check ensure that the sweeper either:
  - Sees `ACTIVE`, marks `EXPIRED`, releases seats. The pay then sees expired, refunds.
  - Or pay completes first, marks `CONFIRMED`, sweeper sees `CONFIRMED`, skips.
- This requires either coordinating via the `holdRepo.save` (sweeper checks status before transitioning), or using a DB-level conditional update.

---

## Edge cases / gotchas

| Case | Handling |
|---|---|
| **Two users hold overlapping seats** | First wins; second gets `SeatsUnavailableException` listing the unavailable seats. |
| **User holds, abandons (closes browser)** | Sweeper releases after TTL. |
| **Hold expires during payment** | Refund the charge (idempotent), release seats, return error. |
| **Payment succeeds but seat-confirm fails** | Refund (with idempotency key); raise alarm; reconcile via audit. |
| **User cancels confirmed booking** | Refund window check; release seats; refund payment. |
| **Cancel after refund window** | Reject. |
| **Show starts during payment** | Hold should expire well before show start; if not, treat as expired. |
| **Seat doesn't exist** | Validate in `holdSeats`; throw. |
| **Empty seat list** | Reject. |
| **Group booking with mixed seat types** | Pricing strategy handles per-seat. |
| **Concurrent cancellation and seat-hold** | The seat is BOOKED → AVAILABLE → HELD; no conflict because `tryHoldSeats` checks current state. |
| **Same seat re-booked while sweeper is mid-release** | `tryHoldSeats` finds AVAILABLE → succeeds; sweeper's `releaseHeldSeats` is idempotent. Both run inside the same monitor so no race. |

---

## Extensions / interview follow-ups

### "Add a queue (waitlist) for sold-out shows."
- New entity: `Waitlist(showId, userId, joinedAt, seatPreferences)`.
- On cancellation, the next waitlister gets a hold automatically and is notified.

### "Add seat preferences (best available)."
- New `SeatRecommender` service: given a layout and N seats, return the best contiguous block.
- Strategy: optimize for centrality, no aisle splits, recliner first.

### "Add multi-show booking (a 'movie marathon' across same evening)."
- A booking can reference multiple shows; payment is one transaction.
- Each show's seats must be held; failure on any rolls back all holds.
- This is a Saga across shows.

### "Add a 'subscriber' tier with discounts."
- User has a tier; `PricingStrategy` chains a `SubscriberDiscountPricing(decorator)`.

### "Why version-counter on Show?"
- For persistent backend: optimistic concurrency. Two services updating seat states need to detect conflicts.
- The in-memory monitor is the in-process equivalent.

### "What if the same user opens two browser tabs, each holds the same seats?"
- First tab succeeds; second fails with `SeatsUnavailableException`.
- Could detect "user already has an active hold for this show" and surface that gracefully.

### "How do you prevent a malicious user from holding all seats forever?"
- TTL is the primary defense.
- Per-user hold limits (maximum N concurrent holds).
- CAPTCHA or rate-limiting at the API layer.

### "What if hold storage and seat storage diverge?"
- Reconcile via the sweeper: scan for "seats in HELD state with no active hold record" and release.
- Or use a DB transaction spanning both.

---

## Scaling

A single show with a few hundred seats is small. Scaling concerns appear with thousands of concurrent shows and millions of users:

### Sharding
- Shard shows by `showId` — each show's contention is local to one node.
- Cross-show queries (search by city/movie) hit a separate read replica or search index.

### Caching
- Show metadata (movie, theater, time) is read-heavy and rarely changes — cache aggressively (Redis with long TTL).
- Seat availability is read-heavy too; cache snapshots with short TTL (or invalidate on every change).

### Event broadcasting
- Seat changes published to a topic; clients (mobile apps) subscribe for real-time updates.
- Helps "Why did the seat I picked turn red while I was scrolling?" UX.

### Hold expiry at scale
- Centralized sweeper doesn't scale. Use:
  - **Per-show timers** — Redis `EXPIRE` on the hold key; cleanup on key-expiration event.
  - **Sharded sweepers** — each instance owns a hash range of shows.
  - **DB-native TTL** (DynamoDB TTL).

### Search and discovery
- "Movies in city X tonight" hits a search service (Elasticsearch).
- Show metadata indexed; query by city + date + filters.

### Payment integrations
- External payment providers; idempotent retries.
- Saga pattern for booking + payment: if payment succeeds but seat confirm fails, refund.

---

## Persistence tradeoffs

Movie booking is a money-handling, high-contention domain — persistence is non-negotiable. The interesting choices are around the seat-state hot spot.

### What persistent data exists
- **Static**: movies, theaters, screens, seat layouts, shows.
- **Hot**: seat states (per show), holds (TTL-bounded), bookings.
- **Audit**: every booking change (created, confirmed, cancelled, refunded).

### Option 1 — Relational (PostgreSQL / MySQL)
```sql
shows(id, movie_id, screen_id, start_time, base_price, version)
seats(show_id, row, col, type, state, current_hold_id, version)
holds(id, show_id, user_id, seats_json, expires_at, status)
bookings(id, show_id, user_id, seats_json, total_amount, payment_ref, status, version)
```
- **Hold seats**: `UPDATE seats SET state='HELD', current_hold_id=?, version=version+1 WHERE show_id=? AND row=? AND col=? AND state='AVAILABLE'` — for each seat in a transaction. If any returns 0 rows affected, ROLLBACK.
- **Pros**: ACID. Versioned seat rows give per-seat optimistic locking.
- **Cons**: hot-row contention if a popular show has thousands of concurrent attempts.
- **When to pick**: standard for most cases; well-tuned PostgreSQL handles thousands of concurrent shows.

### Option 2 — Redis for seat state, SQL for everything else
- Each show's seat states live in a Redis hash: `show:{id}:seats` → `{seat_id: state}`.
- Atomic reservation via Lua script: check all seats AVAILABLE → mark HELD → return success. Lua scripts are atomic in Redis.
- TTL on holds is natural: `EXPIRE hold:{id} 600` and a `SUBSCRIBE __keyevent@0__:expired` to detect expiry.
- Persistent layer (SQL) holds bookings, holds (with status), audit.
- **Pros**: extremely fast hot path; Lua handles multi-seat atomicity.
- **Cons**: two stores; reconciliation logic.
- **When to pick**: very high concurrency on individual shows (block-buster opening night).

### Option 3 — DynamoDB / Cosmos DB
- Conditional writes per seat: `UpdateItem` with `condition-expression: state = 'AVAILABLE'`.
- Multi-seat: DynamoDB transactions (TransactWriteItems supports up to 100 items atomically) — fits typical bookings.
- **Pros**: managed; horizontal scale; strong consistency on conditional writes.
- **Cons**: operationally expensive for write-heavy patterns; query patterns must be planned upfront.
- **When to pick**: serverless / cloud-native deployments with predictable access patterns.

### Option 4 — Event-sourced
- Persist `SeatHeld`, `HoldConfirmed`, `HoldExpired`, `BookingCancelled` events.
- Current seat state derived by fold.
- **Pros**: complete audit trail. Time-travel ("what was seat A1's state at 7:42pm?").
- **Cons**: latency of computing current state; needs snapshots.
- **When to pick**: heavy audit / analytics requirements; reconciliation across systems.

### Option 5 — Hybrid (recommended for production scale)
- **Redis** for hot seat state — sub-millisecond CAS on multi-seat reservations.
- **PostgreSQL** for bookings, holds (with status), audit log.
- **Kafka** for the event stream — feeds analytics, notifications, search.
- Reconcile periodically: SQL holds + Redis seat states must agree.

### TTL handling
The hold TTL is critical — without it, abandoned carts hold seats forever.
- **Redis**: `EXPIRE` natively; subscribe to expire events for cleanup.
- **DynamoDB**: TTL attribute; items auto-deleted; need a separate process for seat-state cleanup.
- **SQL**: a sweeper job querying `WHERE expires_at < now() AND status = 'ACTIVE'`; runs every minute.

### Concurrency under persistence
The pattern: read seat state (with version), compute reservation set, conditional write. All translations preserve the same shape.

| In-memory (synchronized) | SQL | Redis | DynamoDB |
|---|---|---|---|
| `synchronized` block on Show | `SELECT FOR UPDATE` or version check | Lua script | `TransactWriteItems` with conditions |
| Atomic across multiple seats | Single transaction | Lua atomic | Transaction (≤100 items) |

### Recommendation
- **Default**: PostgreSQL with optimistic version locking on seat rows.
- **At scale**: Redis hot path + SQL durable store + Kafka events.
- **Cloud-managed**: DynamoDB transactions + DynamoDB Streams for events.

---

## Talking points for the interview

- "Hold-then-confirm with a TTL is the crux. It separates 'I want these seats' from 'I've paid' and gives the user a window to complete payment without holding seats forever."
- "Atomic multi-seat reservation: either all seats become HELD or none. The contended resource is the show, not individual seats; locking at the show level avoids the multi-seat deadlock you'd get from per-seat locks."
- "Idempotency keys on payment so a network retry doesn't double-charge."
- "Sweeper for expired holds runs out-of-band; it's idempotent (only releases ACTIVE holds) so it's safe even with concurrent confirmations."
- "Pricing is a Strategy because operators want to A/B test (weekend surge, group discounts, member pricing) without redeploying."
- "If we needed cross-region: each region owns specific shows; cross-region search is read-replicated."
- "The two-phase commit pattern here (hold then confirm) is structurally similar to a Saga; the compensating action on payment failure is the seat release."

---

## Summary

Patterns load-bearing: **Strategy** (pricing), **State** (seat lifecycle, booking lifecycle), **Repository** (persistence abstraction), **two-phase reservation** (hold then confirm).

The mental model: seats are a contended inventory item; the hold is a time-bounded reservation; the booking is the final commit. Concurrency is handled at the show level (one show, one monitor); holds expire automatically; payments are idempotent.

The senior signal here is recognizing the two-phase nature of the booking (hold + confirm), correctly handling the TTL, and proposing the right concurrency primitive for the persistent backend (optimistic version locking, or Redis Lua scripts).
