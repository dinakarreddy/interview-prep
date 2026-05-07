# 03 — Design Uber / DoorDash Dispatch (real-time matching, geo-indexing, surge, trip lifecycle)

> The canonical "real-time spatial matching" Staff problem. The interviewer cares less about whether you reach the "right" answer and more about whether you correctly identify that **geo-indexing**, **the matching algorithm**, and **per-market cellular isolation** are the only things that matter — and whether you spend your minutes there instead of on the trip state machine.

---

## 0. Problem statement (as the interviewer would drop it)

> **Interviewer:** "Design the dispatch system for a service like Uber or DoorDash. A rider opens the app, requests a ride, and the system should match them to the nearest available driver in real time. The driver picks them up, drops them off, and the trip completes. Walk me through how you'd build it. If you want to substitute DoorDash — order to courier — that's fine too, the spine is the same."

That's all you get. Your job: scope it, not solve it whole.

---

## 1. Clarifying questions

Pick 5–6. Don't fire them as a checklist — interleave with the interviewer's responses.

> **Candidate:** "Before I design, let me scope this. A few questions:"
>
> 1. **"Are we doing rides (Uber) or deliveries (DoorDash)? They share most of the spine but the leg structure differs — single-leg pickup→dropoff vs two-leg restaurant→customer with prep time."** — *I'll plan for rides as the default and call out where DoorDash diverges. Critical because the second leg + prep-time variance changes the dispatch math.*
> 2. **"What's the rough scale — DAU, concurrent active drivers, trips/day?"** — *Expect: 100M DAU, 5M concurrent active drivers globally, 30M trips/day at peak.*
> 3. **"How fresh do driver locations need to be? Every second? Every five?"** — *Drives the location ingest pipeline. Industry norm: ~4s per driver. At 5M drivers that's ~750K updates/sec.*
> 4. **"Latency target for the rider-side match — from 'request' to 'driver assigned'?"** — *Expect: p50 < 2s, p99 < 5s. Below 2s users start retrying; above 5s they cancel.*
> 5. **"Are surge pricing and ETA in scope, or are those black-box services I just call?"** — *I'll treat both as service boundaries with clear interfaces — model surge as a streaming aggregator, ETA as an ML-served service. Interesting enough to design but I won't go deep on the ML.*
> 6. **"Is this multi-market (multiple cities) from day one?"** — *Yes — drives the cellular architecture. A driver in NYC will never serve a rider in SF, so why share infra?*
>
> **Out of scope (state explicitly):**
> - Payments / payouts (mention as integration; trip-complete event triggers it)
> - Ratings, reviews, fraud detection
> - Driver onboarding, background checks, insurance
> - Maps, routing, traffic data ingestion (I'll assume an internal Maps/Routing Service exists)
> - Authentication
> - Customer support tooling
> - Pool / shared rides (mention briefly as a sibling design)
>
> **Candidate:** "I'm going to design the **rider-request → driver-match → trip lifecycle** path, with location ingestion, geo-indexing, and the matching algorithm as the deep dives. Surge and ETA I'll architect as separate services and describe their interfaces. Sound right?"

> **Interviewer:** "Sounds good. Go."

---

## 2. Functional requirements

| Priority | Requirement |
|---|---|
| P0 | Driver app emits location every ~4s while online |
| P0 | Rider can request a ride (origin, destination, vehicle class) |
| P0 | System matches request to a nearby available driver within seconds |
| P0 | Trip lifecycle: REQUESTED → ASSIGNED → ARRIVED → STARTED → COMPLETED (durable) |
| P0 | Real-time trip tracking (rider sees driver location; driver sees route) |
| P0 | Surge multiplier applied at request time, per market / per cell |
| P0 | ETA shown at request time and updated during pickup |
| P1 | Driver can accept / decline; auto-reassign on decline or timeout |
| P1 | Cancel by rider or driver mid-flow; clean state transitions |
| P1 | Driver goes offline mid-trip → preserve trip state, attempt reassignment |
| P1 | Multi-leg dispatch (DoorDash): restaurant prep + courier pickup + customer dropoff |
| P2 | Batched dispatch (one courier carries 2–3 nearby orders) |
| P2 | Scheduled / pre-booked rides |
| Out | Pool / shared rides, payments, ratings, fraud, maps ingest |

---

## 3. Non-functional requirements

| Dimension | Target | Justification |
|---|---|---|
| Match latency | p50 < 2s, p99 < 5s (request → driver assigned) | Below this, riders retry/cancel; above this, abandonment skyrockets |
| Location ingest freshness | Driver location ≤ 4s stale at match time | Below 4s costs more (network, battery); above 4s causes "ghost driver" matches |
| Trip-state availability | 99.99% (~52 min/year) per cell | Trip-state outage is a "money on the line" event — drivers and riders are both blocked |
| Match-rate availability | 99.95% per cell | A 2-minute match outage in NYC is recoverable; a daily one is not |
| Trip-state consistency | Strong (linearizable) on the trip row | Two services must never both think a trip is assigned to different drivers |
| Location-data consistency | Eventual; staleness ≤ 4s | We're already accepting 4s lag from the device; doesn't need stronger |
| Durability | No completed trip ever lost | Tax/legal/payment audit trail |
| Scale | 100M DAU, 5M concurrent drivers, 30M trips/day, ~750K location updates/sec | Industry-public Uber-class scale |
| Geography | Multi-region active-active **per market**; cells don't share traffic | Cell = city; cross-cell traffic is the long tail (airports, inter-city) |

---

## 4. Capacity estimation

> **Candidate:** "Let me estimate aggressively — round numbers. Stop me if you want me to dig deeper."

### Users / drivers
- 100M DAU
- 5M **concurrently active** drivers globally (peak — far smaller than total registered drivers)
- 30M trips/day at peak

### Trips
- 30M trips/day → 30M / 86,400 ≈ **350 trips/sec** average
- Peak (3×): **~1,000 trip requests/sec**
- Each trip request triggers: 1 dispatch match + N candidate-driver scans + 1 trip-state-row write at ASSIGNED + ~5 state transitions over the trip's lifecycle

### Location updates (this is the punchline)
- 5M drivers × 1 update / 4s = **1.25M updates/sec at saturation**
- Realistic load (drivers idle/offline mid-shift, smoothing): **~750K updates/sec**
- Peak (commute, Friday night): **~1M updates/sec**
- **This is by far the dominant write workload — orders of magnitude bigger than trip writes.**

### Storage
- Per location update: `{driver_id, lat, lng, heading, speed, timestamp}` ≈ 80 bytes serialized
- Per second: 750K × 80B = **60 MB/s** ingest
- Per day: ~5 TB/day raw
- **Decision driver:** we don't store every location update durably. Hot path (current location) is in memory / Redis, expires in seconds. Tail (telemetry / route reconstruction) is sampled (every Nth point or on state transitions) and lands in object storage / data lake.

### Trip storage
- 30M trips/day × ~1 KB per trip row (ids, state transitions, ts) = **30 GB/day** for trip records
- Per year: ~11 TB. Plus replication and history. Trivial compared to location stream.

### Bandwidth
- Driver app upstream: 80 B / 4s × 5M drivers = ~100 MB/s upstream globally — small.
- Driver app downstream (route, instructions): tens of KB at trip start, then tiny pings.
- Rider app: poll-or-push trip updates ~1Hz during active trip, 1–2 KB/update.

### Match work
- 1,000 requests/sec × ~50 candidate drivers screened/request = **50K candidate evaluations/sec**
- Plus surge eval, ETA eval per candidate. Bounded; small compared to the location stream.

### What the numbers tell me
1. **The location pipeline is the dominant cost driver, not the matcher.** ($/driver/day for telemetry > $/trip for compute.)
2. **Match QPS is small.** 1K/sec is unremarkable. We don't need exotic matching infra — we need a *correct* matcher.
3. **Geo-index size matters more than QPS.** 5M drivers across thousands of H3 cells with 4s update freshness means each cell has tens-to-hundreds of drivers and updates several times/sec.
4. **Per-cell isolation is natural.** A driver only ever appears in one or two adjacent H3 cells per second; the entire workload is geographically partitionable.

---

## 5. API design

> **Candidate:** "I'll keep this minimal — the public REST/gRPC surface for rider, driver, and the internal dispatch surface."

### Driver-side
```
POST /v1/driver/online
  body: { vehicle_class, current_location }
→ 200 { session_token }

POST /v1/driver/location   (high-volume; ideally over WebSocket / gRPC stream)
  body: { lat, lng, heading, speed, accuracy, ts }
→ 204

POST /v1/driver/offline
→ 204

POST /v1/driver/trips/{trip_id}/accept
POST /v1/driver/trips/{trip_id}/decline
POST /v1/driver/trips/{trip_id}/arrive   (at pickup)
POST /v1/driver/trips/{trip_id}/start    (after rider in vehicle)
POST /v1/driver/trips/{trip_id}/complete
```

### Rider-side
```
POST /v1/rider/quote
  body: { origin, destination, vehicle_class }
→ 200 { quote_id, fare_estimate, eta_seconds, surge_multiplier, expires_at }

POST /v1/rider/trips
  body: { quote_id }
→ 202 { trip_id, status: "REQUESTED" }
   (long-lived state; client polls or subscribes via WebSocket)

GET /v1/rider/trips/{trip_id}
→ 200 { trip_id, status, driver{...}, eta_seconds, driver_location, ... }

POST /v1/rider/trips/{trip_id}/cancel
→ 200 { status: "CANCELLED", cancellation_fee }
```

### Internal (between dispatch and downstream)
```
# Match Service → Notification Service (driver app push)
POST /v1/notify/driver/{driver_id}
  body: { type: "trip_offer", trip_id, expires_at, ... }

# Pricing Service interface (called by Quote)
GET /internal/pricing/surge?cell={h3_index}&vehicle_class=...
→ 200 { multiplier, computed_at }

# ETA Service
GET /internal/eta?from={lat,lng}&to={lat,lng}&driver_heading=...
→ 200 { eta_seconds, route_polyline, confidence }
```

### Anti-patterns to avoid
- Don't return raw lat/lng in the rider-side trip endpoint without coalescing — emit a smoothed `driver_location` updated at most once per second.
- Don't make the driver location endpoint synchronous-per-update over HTTP — that's 750K HTTP requests/sec. Use WebSocket / gRPC stream / MQTT.
- Don't merge `quote` and `trips`. The quote is cheap, idempotent, frequent; the trip is expensive and stateful. Separate.

---

## 6. Data model

### Tables / collections

#### `drivers` (sharded by driver_id; per-cell)
| Column | Type | |
|---|---|---|
| driver_id | bigint (Snowflake) | PK |
| current_state | enum {OFFLINE, AVAILABLE, ON_TRIP, BREAK} | |
| vehicle_class | enum | |
| home_market_id | string | foreign cell ID |
| last_location | (lat, lng, ts) | not authoritative — denormalized cache |
| last_seen_at | timestamp | |

This table is durable but not on the hot path for matching. The hot path uses the in-memory geo index (below).

#### `trips` — the durable trip-state row (Spanner / CockroachDB, **strong consistency**)
| Column | Type | |
|---|---|---|
| trip_id | bigint (Snowflake) | PK |
| rider_id | bigint | secondary index |
| driver_id | bigint (nullable until ASSIGNED) | secondary index |
| state | enum {REQUESTED, ASSIGNED, ARRIVED, STARTED, COMPLETED, CANCELLED} | |
| origin | (lat, lng, h3_cell) | |
| destination | (lat, lng, h3_cell) | |
| market_id | string | partition key for residency / cell isolation |
| vehicle_class | enum | |
| fare_estimate | decimal | |
| surge_multiplier | decimal | |
| state_transitions | array<{state, ts, actor_id}> | append-only audit |
| created_at | timestamp | |
| updated_at | timestamp | |

**Why Spanner / CockroachDB:** the trip row is **the source of truth** that two services (rider-side and driver-side) read and update concurrently. Two services must never both believe the same driver was assigned two different trips. We need linearizable single-row updates with conditional writes (compare-and-set on `state`) and we need it across regions for failover. Spanner / CockroachDB / YugabyteDB give us this. PostgreSQL with multi-region replication can do single-region strong consistency but cross-region failover is harder. DynamoDB transactions are an option but costly at this latency target.

#### `driver_location_index` — the in-memory geo index (NOT a table; it's an in-process structure)
- Per-cell process holds: `Map<h3_cell, Map<driver_id, {lat, lng, heading, speed, ts}>>`
- Or: Redis Geo (`GEOADD`, `GEORADIUS`) per cell as the simple variant
- Updated by Kafka consumer every ~4s per driver
- Eviction: any driver whose `ts` is more than 30s old is removed (treated as offline)

**Why in-memory / Redis Geo, not a database:** the read pattern is "all drivers in this H3 cell and adjacent cells, sorted by distance to (lat, lng)". A traditional database with a spatial index works but at 1M reads/sec across millions of cells with sub-100ms tail latency, in-memory wins. We accept that crashing a cell's in-memory index loses the index — but Kafka replays it back in seconds (every driver re-emits within 4s).

#### `surge_metrics` — per-cell rolling supply/demand (Flink stream state, materialized to Redis)
| Column | Type | |
|---|---|---|
| cell_id | string (h3) | PK |
| supply | int (drivers AVAILABLE in cell) | |
| demand | int (open requests in cell, last 60s) | |
| multiplier | decimal | computed |
| computed_at | timestamp | |

#### `historical_trips` (Cassandra / data lake)
- Long-tail trip data for analytics, ML training, audit
- Sharded by `market_id, completion_date`
- Backfilled via CDC from the `trips` table

#### `location_telemetry` (object storage — sampled)
- Sampled location pings per trip (every 30s during a trip, plus state transitions)
- For ETA ML training, route replay, fraud detection
- Cold storage; not on any hot path

### Why these choices

- **Spanner-class store for `trips`**: I need linearizable writes on a row that two services touch. Anything weaker risks double-assignment.
- **In-memory + Redis Geo for the live driver index**: the read pattern is hyper-local and the data is ephemeral.
- **Cassandra for historical**: write-heavy append, no transactions, time-partitioned queries.
- **Flink for surge**: it's fundamentally a streaming aggregation problem with windowed state per cell; Flink owns this idiom.

---

## 7. High-level architecture

```
                  ┌───────────────────────────────────────────────┐
                  │              Driver / Rider Apps              │
                  └──────────────┬──────────────┬─────────────────┘
                                 │              │
                          location stream    request/trip RPC
                          (gRPC/WS, 4s)      (HTTPS / WS)
                                 │              │
                                 ▼              ▼
                  ┌───────────────────────────────────────────────┐
                  │           Edge / API Gateway                  │
                  │   (auth, rate limit, mTLS, geo-routing)       │
                  └──────────┬───────────────────────┬────────────┘
                             │                       │
              ┌──────────────┘                       └─────────────┐
              ▼                                                    ▼
     ┌──────────────────┐                                 ┌─────────────────┐
     │ Location Ingest  │                                 │ Dispatch API    │
     │ (gRPC/WS server) │                                 │ (rider entry)   │
     └────────┬─────────┘                                 └──────┬──────────┘
              │ writes to Kafka                                  │
              │ (partitioned by H3 cell)                         │
              ▼                                                  ▼
     ┌──────────────────────┐                          ┌────────────────────┐
     │ Kafka:               │                          │ Quote Service      │
     │ driver.locations.{cell}                         │ (calls Pricing,    │
     │ partitioned by H3    │                          │  ETA; idempotent)  │
     └────────┬─────────────┘                          └─────┬──────────────┘
              │                                              │
              ▼                                              ▼
     ┌──────────────────────────────┐               ┌────────────────────────┐
     │ Cell Process (one per cell)  │               │ Trip Service           │
     │  - in-memory geo index       │ ◄────────────►│ (state machine,        │
     │  - per-cell Redis Geo mirror │   query       │  Spanner-backed,       │
     │  - subscribes to its         │   nearest     │  trip row CAS)         │
     │    Kafka partitions          │   drivers     └──┬─────────────────┬──┘
     └────────┬────────┬────────────┘                  │                 │
              │        ▲                               │                 │
              │        │ candidate request             │                 │
              ▼        │                               ▼                 ▼
     ┌──────────────────────────┐              ┌──────────────┐   ┌──────────────┐
     │ Matching Service         │              │ Notification │   │ Pricing /    │
     │ (windowed batched        │ ────────────►│ Service      │   │ Surge Stream │
     │  dispatch, every 1–2s)   │   trip_offer │ (push to     │   │ (Flink)      │
     │ - top-K nearest          │              │  driver app) │   └──────┬───────┘
     │ - ETA-weighted score     │              └──────┬───────┘          │
     │ - acceptance prediction  │                     │ accept/decline   │
     └──────────┬───────────────┘                     ▼                  │
                │ on match                  ┌─────────────────┐          │
                ▼                           │ Trip Service    │ ◄────────┘
        ┌─────────────────┐                 │ updates trip    │   surge multiplier
        │ Trip Service    │                 │ row state       │
        │ assigns driver  │                 └─────────────────┘
        └─────────────────┘
                │
                │ writes everywhere downstream
                ▼
     ┌────────────────────────────────────────────────────────┐
     │   Spanner trips table   │   Cassandra historical       │
     │   (live trip state)     │   (post-completion archive)  │
     └────────────────────────────────────────────────────────┘
```

### Request walk-through

**Location ingest (continuous):**
1. Driver app sends `{lat, lng, heading, speed, ts}` every ~4s over a persistent gRPC/WS connection.
2. Edge terminates the connection, forwards to Location Ingest service.
3. Location Ingest writes to Kafka, **partitioned by the H3 cell** computed from (lat, lng).
4. Cell Process (one logical process per "live" cell, scaled out per market) consumes its partition, updates in-memory index and per-cell Redis Geo.
5. Stale entries (>30s) are evicted; driver is treated as offline.

**Quote (rider opens app, sees ETA + price):**
1. Rider app → API Gateway → Quote Service.
2. Quote Service calls:
   - Pricing/Surge Service for `surge_multiplier` at origin's H3 cell.
   - ETA Service for `eta_to_pickup` (how long to find a driver) and `eta_origin_to_destination` (trip duration).
   - Cell Process for `nearest_drivers_count` (sanity check supply).
3. Returns a quote with `expires_at` (~1–2 min). The quote is idempotent and can be re-issued.

**Trip request → match (the hot path):**
1. Rider confirms quote → `POST /v1/rider/trips` → Dispatch API → Trip Service.
2. Trip Service inserts a `trips` row in **REQUESTED** state (Spanner, single-row write, strong consistency).
3. Trip Service emits `trip.requested` to Kafka topic for that cell.
4. Matching Service (running per cell) ingests the request, **batches with other open requests in this cell over a 1–2s window**, queries the Cell Process for top-K candidate drivers per request, scores each `(request, driver)` pair, picks the optimal assignment for the batch (Hungarian-style or greedy depending on batch size).
5. For each chosen pair, Matching Service does a **conditional update** on the trip row: `UPDATE trips SET state='ASSIGNED', driver_id=X WHERE trip_id=Y AND state='REQUESTED'`. CAS prevents double-assignment.
6. On success, Matching Service publishes `trip.offered` → Notification Service pushes to driver app.
7. Driver accepts within N seconds (default 15s) → driver app POSTs `accept` → Trip Service.
8. On decline / timeout → Matching Service re-enters the matching pool for that trip with the previously-offered driver excluded.

**Trip lifecycle (state machine):**
- REQUESTED → ASSIGNED (on match)
- ASSIGNED → ARRIVED (driver pings "at pickup")
- ARRIVED → STARTED (rider in vehicle)
- STARTED → COMPLETED (trip ends)
- Any state → CANCELLED (with rules: free if pre-arrival, fee otherwise)

Each transition is a CAS on the trip row with the expected prior state. Unexpected transitions are logged and rejected.

**Real-time tracking:**
- Driver app continues sending location at 4s.
- Rider app subscribes (WebSocket) to `trip.{trip_id}.location`, gets coalesced 1Hz updates of the assigned driver's position.
- Server side is a small subscription fanout: trip-id → set of viewers (rider, ops dashboards). Per-trip channel.

### Why this shape

- **Cell-process per H3 cell** for the live driver index is the single most important architectural commitment. It exploits the fact that all matching is geographically local: 99% of matches are within one H3 cell + 6 neighbors. By making the cell the unit of work, we get linear scaling, natural sharding, and natural fault isolation.
- **Kafka partitioned by H3 cell** means a hot region (NYC at rush hour) has many partitions; an empty region has one. We can repartition by adjusting H3 resolution for the partition key.
- **Windowed batched matching** beats greedy 1:1 matching. The interview literature still describes greedy nearest-driver. The right answer is window-based: collect 1–2s of requests, solve the assignment problem across all of them. We'll deep-dive this.
- **Trip state in Spanner** is non-negotiable for the consistency guarantees we listed.

---

## 8. Deep dives

### Deep dive 1: Geo-indexing — why H3 over S2 / Geohash / quadtree

> **Interviewer:** "You mentioned H3. Why? Walk me through the geo-index."

**Candidate:**

There are four serious options:

| System | Shape | Key property | Trouble |
|---|---|---|---|
| **Geohash** | Rectangular cells from string-prefix encoding | Easy to use; lexicographic prefix = containment | Cell aspect ratio depends on latitude (squished at poles); seams at longitude/latitude boundaries make "nearby" cells non-obvious |
| **S2 (Google)** | Hierarchical sphere covering | Mathematically rigorous; great for global apps; no projection distortion | Library complexity; cells are quadrilaterals, neighbors of edge cells aren't uniform |
| **H3 (Uber)** | Hex grid, multiple resolutions | **Uniform 6-neighbor adjacency**; consistent cell area within a resolution; built precisely for dispatch | Edge cases at the 12 pentagons (near poles); not as expressive for arbitrary polygons |
| **Quadtree** | Recursive subdivision | Adapts to density; great for non-uniform data | Mutable index; coordination on splits/merges; hard to partition Kafka topics by |

**My pick: H3.**

Reasons:
1. **Uniform adjacency.** A hex has exactly 6 neighbors, all the same distance. For a "find drivers within X km" query, the candidate set is *the cell + the 6 neighbors* (k-ring of size 1) — a clean, predictable expansion. With Geohash you have to handle 8 neighbors of varying sizes.
2. **Multi-resolution out of the box.** H3 has 16 resolutions. We use **resolution 8 (~1 km edge)** for matching — large enough that any city block has a useful candidate set, small enough to avoid scanning the whole city. We use **resolution 10 (~100 m edge)** for ETA snapping and arrival detection. We use **resolution 6 (~5 km edge)** for surge.
3. **Predictable partitioning.** Kafka partitions keyed by `h3.toString(res=8)` give us cells of similar driver volume. We can subdivide hot cells (Manhattan) by going to resolution 9 just for those keys.
4. **Battle-tested for this exact problem** at Uber.

**Cell-size choice:**
- Resolution 8 (~1 km edge, ~0.7 km² area) for the matching index. A typical urban cell has 5–50 drivers AVAILABLE at any moment. Small enough for fast scans, large enough that the candidate set isn't empty most of the time.
- Resolution 10 (~100 m) for "did the driver arrive at pickup" checks (geofencing).
- Resolution 6 (~5 km) for surge metrics (smaller cells make the supply/demand signal too noisy).

**The query pattern:**
```python
def candidate_drivers(rider_lat, rider_lng, vehicle_class):
    cell = h3.geo_to_h3(rider_lat, rider_lng, resolution=8)
    candidates = []
    for c in [cell] + h3.k_ring(cell, k=1):           # cell + 6 neighbors
        candidates += driver_index[c].available(vehicle_class)
    if len(candidates) < MIN_K:
        for c in h3.k_ring(cell, k=2):                 # expand if sparse
            candidates += driver_index[c].available(vehicle_class)
    # filter by distance, sort, return top-K
    return top_k_by_distance(candidates, k=20)
```

**Why I would NOT use a database spatial index (PostGIS, MongoDB 2dsphere) for the live index:**
- 1M location updates/sec into a database with a maintained spatial index is brutal on the write path.
- Read latency at this rate is hard to keep below 10ms.
- The live index is *ephemeral* — we don't need durability. Lose it and Kafka replays in 4 seconds.
- In-memory per-cell wins on every dimension.

I'd use PostGIS for the *static* geometry (city polygons, market boundaries, geofences) but not for the live driver positions.

### Deep dive 2: Location ingestion at 750K updates/sec

> **Interviewer:** "Walk me through the ingestion pipeline. What does a single location update touch?"

**Candidate:**

The pipeline:

```
Driver app
  │  (gRPC bi-di stream, persistent — saves the TCP/TLS handshake per update)
  ▼
Edge gateway (terminate TLS, route to nearest region's Location Ingest)
  │
  ▼
Location Ingest service
  │  - validate (range, schema, monotonic ts per driver)
  │  - compute h3_cell at resolutions 6, 8, 10
  │  - publish to Kafka with key = h3_cell_res8
  ▼
Kafka topic: driver.locations.{market_id}
  partitions: ~256 per medium market, scaled by cell density
  retention: 60s (we don't need long retention for live state)
  ▼
Cell Process (one logical owner per cell, multiple processes per market)
  │  - consume own partitions (assigned by Kafka group + sticky partitioning)
  │  - update in-memory geo index
  │  - mirror to Redis Geo (for cross-process reads from Matching Service)
  │  - emit derived events (driver_arrived_in_geofence, etc.)
  ▼
Sampled telemetry → object storage (every 30s during a trip, or N% baseline)
```

**Why gRPC bi-di stream and not HTTP-per-update:**
- Saves the TCP/TLS connect cost on each update. At 4s cadence the connection is reused.
- Lets us push back to the driver in the same channel (e.g., "you're offline, rejoin").
- Compresses better over a long-lived connection.
- Backpressure friendly.

Alternative: **MQTT**. Cheaper on battery, smaller header, designed exactly for this. Equally valid pick — I'd choose based on whether we already have MQTT brokers in our infra. For greenfield I lean gRPC because the rest of the platform is gRPC.

**Why Kafka and not a raw stream into Redis:**
- We need durability for "the last 30s of all driver locations" — if the Cell Process crashes, we replay.
- We need fan-out: the same location update feeds the live index, the surge stream, and the telemetry archive.
- Kafka's partition model maps cleanly to H3 cells.

**Why partition by H3 cell, not by driver_id:**
- Locality: all updates for the same cell go to the same partition → the same Cell Process. The hot read path (matching) doesn't have to fan-out across processes.
- Driver_id partitioning would scatter cell members across processes; matching would have to scatter-gather.
- The cost: when a driver crosses a cell boundary, they switch partitions. We accept this — at 4s freshness, a driver is in their new cell within one update.

**Hot-cell handling:**
- Times Square cell at noon Saturday is way denser than a quiet residential cell. We *don't* let a single Kafka partition handle it.
- Solution: at provisioning time, identify hot cells and **subdivide them at H3 resolution 9** for the partition key. The Cell Process then unions the sub-cells back when serving matches.
- Alternative: re-key by `h3_cell || driver_id_mod_K` and round-robin. We lose strict locality but spread load. I'd start with subdivision and go to mod-K only if the hottest cell exceeds partition throughput.

**Cost of this pipeline:**
- This is the **single most expensive component** in the system. Estimating: ~$0.0005 per driver per active hour for ingestion alone (Kafka + compute). At 5M concurrent drivers × 8 active hours/day × 365 days × $0.0005 ≈ **$7M/year just for location ingest.** It's worth a deep dive on cost levers:
  - **Adaptive cadence** — drivers stationary at a red light don't need 4s updates. Send 4s when moving, 15s when stopped. ~30% reduction.
  - **Edge aggregation** — coalesce multiple drivers' updates at the edge before publishing to Kafka. Saves Kafka request overhead, not bandwidth.
  - **Cell-local processing** — keep computation in the same datacenter as the cell. Cross-region location streams are wasted bandwidth.
  - **Sampling for telemetry** — full-rate updates feed the live index; archive captures every 30s only.

### Deep dive 3: The matching algorithm — greedy vs windowed batch

> **Interviewer:** "Now the actual match. Take a single rider request — how do you pick a driver?"

**Candidate:**

Two paradigms:

**Greedy nearest-driver (the obvious / wrong answer):**
- Request comes in. Query cell index. Pick the nearest available driver. Offer.
- Pros: trivial, low latency, no batching delay.
- Cons: globally suboptimal. Imagine two requests A and B in the same cell, and two drivers D1, D2. D1 is slightly closer to both. Greedy gives A→D1 (best for A) and then forces B→D2 (worse for B). The correct assignment might be A→D2 and B→D1, minimizing total ETA.
- At scale, "globally suboptimal" translates to ~5–15% worse driver utilization and rider ETA.

**Windowed batched dispatch (what real Uber/Lyft/DoorDash do):**
- Collect open requests in a cell over a 1–2s window.
- At the end of the window, run an assignment algorithm over (open_requests × candidate_drivers).
- Optimization: minimize sum of ETAs, weighted by surge / supply-demand pressure.

**The math:**
- Hungarian algorithm: O(n³) for n requests against n drivers. For n=20 (typical batch in a cell), n³ = 8000 ops — sub-millisecond. We can afford it.
- For larger batches (busy cells), bipartite matching with greedy fallback or auction algorithms. Real systems use customized solvers tuned for the dispatch objective.

**The objective function** is more than distance:
```
score(request, driver) =
    α * eta_to_pickup_seconds          # primary: how fast can they get there
  + β * (1 / driver_acceptance_rate)   # weight in drivers more likely to accept
  - γ * driver_idle_minutes            # fairness: give the longer-waiting driver
  + δ * heading_alignment_penalty      # avoid 180° turns
  + ε * supply_demand_imbalance        # if cell is undersupplied, pull from neighbors
```

Coefficients are learned per market.

**Why a 1–2 second window:**
- Below 1s: too few requests batched, devolves to greedy.
- Above 2s: rider perceives the delay; competitors win.
- 1–2s is the sweet spot empirically. We treat it as a tunable per-market knob (slower in supply-rich, low-pressure markets).

**Decline / timeout handling:**
- Driver has 15s to accept. On decline or timeout, the trip re-enters the next batch with the previous driver excluded.
- After 3 cycles without acceptance, escalate: notify rider "still searching", widen the cell radius, possibly raise surge.
- After 5 cycles, fail the request and let the rider retry / pick another option.

**ETA prediction in the score:**
- The ETA Service is queried per (request, candidate) pair. With batches of 20×20 = 400 pairs, that's a lot of queries.
- Optimization: pre-compute ETA from each driver to a coarse grid of pickup points (resolution 9 H3 cells). Match-time lookup is cell-coarse and adjusts for the residual.
- We'll deep-dive ETA separately.

**Failure mode: window-stuck cell.**
- If the Matching Service for cell X is paused (GC, deploy), the next window doesn't fire. Requests pile up.
- Mitigation: another cell's Matching Service can adopt the orphaned cell within seconds (Kafka consumer group rebalancing). Trips' state in Spanner is the truth, the in-memory match queue is regenerable.

> **Interviewer:** "What about cross-cell matches? An airport is on a cell boundary."

**Candidate:**

Three sub-cases:
1. **Pickup near a boundary, candidates in adjacent cell.** k-ring(1) covers this. Standard.
2. **Long-distance pickup (no driver in the rider's cell or k-ring).** Expand to k-ring(2), then 3. After expansion, if still empty, return "no drivers" to the rider. Tail latency suffers for these requests but they're rare.
3. **Inter-city / cross-market pickup (airport is in market A, rider's home is market B; or driver licensed in A, rider in A's airport but driver going to B).** Rare but real. We have a special **airport cell** designation: airport cells are members of multiple markets. Drivers can opt into it. The matching service for the airport cell can reach into both markets' driver indices. Compliance/insurance teams own the policy.

> **Interviewer:** "OK and what does this look like for DoorDash?"

**Candidate:**

DoorDash diverges in three ways:

1. **Two legs.** The trip has restaurant→courier→customer, not just rider→destination. The dispatch problem is finding a courier to do *both legs* well, not just the closer leg.
2. **Restaurant prep time variability.** The order isn't ready when placed. We can't assign a courier too early — they'll wait 15 min — or too late — order gets cold. The dispatcher needs an estimated `food_ready_at` time, which itself is an ML prediction.
3. **Batched dispatch (multi-pickup).** A courier can carry 2–3 orders if they're nearby and pickup-time-compatible. The matching becomes assignment of *baskets of orders* to couriers, not 1:1.

The architecture is the same — H3-indexed couriers, windowed batching — but:
- The Matching Service objective changes to include both leg ETAs and prep-time alignment.
- The batch optimizer must also decide *how to bundle* orders into courier baskets, not just match singletons.
- We add a `food_ready_at` predictor (ML on restaurant + dish + time-of-day) feeding the matcher.
- Cancellations are more expensive — a courier already at the restaurant for a cancelled order is a sunk cost.

### Deep dive 4: Surge pricing as a streaming aggregator

> **Interviewer:** "How does surge work? Is it called per request?"

**Candidate:**

Surge is a **read-time lookup** but the *computation* is a continuous streaming aggregation. Two services collaborate:

**The Pricing/Surge Stream Job (Flink):**
- Consumes the Kafka location stream (driver state changes) and the trip-request stream.
- Maintains windowed state per H3 res-6 cell:
  - `available_drivers_count` (last-known supply per cell)
  - `requests_count` (rolling 60s)
  - `unmatched_request_count` (rolling 60s)
  - `cancellation_rate` (rolling 5min)
- Every 60 seconds, computes a surge multiplier per cell using a model: `multiplier = f(supply, demand, unmatched_rate, time_of_day, weather, ...)`.
- Writes the result to a Redis hash: `surge:{market}:{cell_res6} → multiplier`.

**The Pricing Service (read-time):**
- Stateless service that, given (origin_lat, origin_lng, vehicle_class), looks up the surge multiplier in Redis and applies pricing rules.
- p99 < 20ms. Trivial on the hot path.

**Why a stream, not a request-time computation:**
- Per-request computation would re-scan supply/demand every quote. At 1K requests/sec × cells × drivers, that's wasted compute.
- Stream model means the multiplier is "always available, slightly stale". Surge that's 60s stale is fine — surge is a slow signal anyway.

**Why per-cell at res 6 (~5 km):**
- Smaller cells (res 8) have noisy signals — one driver going offline shouldn't double the price for a city block.
- Larger cells (res 4) lose locality — surge for downtown shouldn't apply to the suburbs.

**Surge fairness as a metric:**
- "Surge fairness" is a hard problem. We need to be able to explain to a regulator and to riders why surge is what it is.
- We log every surge computation with inputs, version of model, and result. Auditable.
- We also expose surge as a multiplier *with caps* (e.g., 5× max in most markets) and decay rules (multiplier never increases by more than 20% per minute, to avoid shock).

**Per-market surge models:**
- Each market has its own elasticity. NYC: small price changes don't move supply much. A new market: highly elastic.
- We train one model per major market, falling back to a "shared regional" model for small markets.
- This is one of the natural per-cell isolations: surge is *per market by definition*.

### Deep dive 5: The trip state machine

> **Interviewer:** "Walk me through the trip state machine. What if a driver goes offline mid-trip?"

**Candidate:**

States and transitions:

```
                    REQUESTED
                       │
                       │ matched
                       ▼
                    ASSIGNED ────────────► CANCELLED (rider/driver, free if no fee window)
                       │
                       │ driver geofence-arrival
                       ▼
                    ARRIVED
                       │
                       │ driver pings "start"
                       ▼
                    STARTED
                       │
                       │ driver pings "complete" (at destination geofence)
                       ▼
                    COMPLETED
                       │
                       │ async finalize
                       ▼
                  (payment, receipt, ratings)
```

**Transition rule:** every state change is a CAS on the trip row in Spanner: `WHERE trip_id=? AND state=expected_prev_state`. If the CAS fails, the requesting actor learns the trip moved on without them and reconciles.

**Why Spanner / CockroachDB and not just MySQL:**
- Trip lifecycle is 5–30 minutes. During that time, two services (driver app, rider app) are mutating it concurrently. We need linearizable single-row operations.
- Cross-region: a trip might span markets (rare, but airport-to-suburb of next state). Spanner gives us this transparently.
- The cost — Spanner write latency p99 ~50ms — is small relative to the trip length. We pay it.

**Failure: driver goes offline mid-trip.**

This is an interview classic. The textbook flow:

1. Driver's app stops sending location pings. Detection: location-ingest's "no update in 30s" heartbeat alarm fires for that driver.
2. Trip Service's reconciler (a periodic job per cell) examines trips in `ASSIGNED` / `STARTED` whose driver is now offline.
3. Two cases:
   - **Pre-pickup (`ASSIGNED`)**: cancel the offer, re-enter the trip into the matching pool with that driver excluded. Rider sees "finding new driver" message. The trip row state moves back to `REQUESTED` (or a new `REASSIGNING` substate).
   - **Mid-trip (`STARTED`)**: the rider is in the car. We can't reassign — there's a human in transit. We notify ops, start a customer service flow, possibly surface "driver having issues" to the rider, and after a longer grace period (5 min) treat it as a trip incident.
4. The trip's `state_transitions` array logs the offline event, so postmortem and customer-support tooling have an audit trail.

**Failure: dispatcher dies between offering and confirming.**
- The `trip.offered` event was published. The driver app may have shown "incoming trip" UI.
- On dispatcher recovery (or another instance picking it up), the Trip Service sees the trip in REQUESTED, the driver in some "offered" cache state. The driver-side accept flow does a CAS — if the trip was already re-offered to someone else, the CAS fails and the driver app gets "trip taken".
- Net: at-most-once user-visible assignment, even with dispatcher failures.

**Failure: trip row write succeeds, downstream notification fails.**
- This is the canonical dual-write problem. The trip is `ASSIGNED` in Spanner but the driver was never told.
- Mitigation: outbox pattern. The Trip Service writes the trip row and a notification record in the same Spanner transaction. A separate publisher reads the outbox and publishes notifications, with at-least-once delivery.
- Alternatively, make notification idempotent and re-emittable from the trip's audit log.

### Deep dive 6: Real-time ETA

> **Interviewer:** "How is ETA computed?"

**Candidate:**

ETA is its own service, with three sources of signal:

1. **Map / routing** — base travel time from origin → destination via the road network. We own this internally (Google Maps / Mapbox cost is huge at scale; at our scale, in-house wins).
2. **Live traffic** — derived from our own driver fleet's recent speeds on each road segment. Self-fulfilling: more drivers = better traffic data.
3. **ML adjustment** — model that learns the difference between routing-predicted time and actual time, conditioned on time-of-day, weather, day-of-week, segment.

The ETA Service is queried in two regimes:
- **At quote time** — "how long will the trip take?" Best-effort, can take 100ms.
- **At match time** — "ETA from this driver to this rider's pickup." Called O(K) times per match. Latency budget per call: 5–10ms.

To meet the match-time budget, ETA is heavily cached by (origin_cell_res9, dest_cell_res9), with the live-traffic adjustment applied at lookup. This gets us ~95% cache hit and sub-5ms p99.

**ETA accuracy as a metric:**
- We measure |predicted - actual| at trip completion.
- Goal: <90s MAE for trips under 30 min. Tighter for the pickup leg (<60s MAE) because riders compare it to the wall clock.

> **Interviewer:** "What if the ETA service is down?"

**Candidate:**
- Match-time fallback: use driver-to-rider straight-line distance × a regional average speed. Loses optimization quality but matches still happen.
- Quote-time fallback: show last known ETA for the route (from a cache) or display a wider range ("5–10 min").
- Pricing fallback: don't apply distance-based surge components.
- The system degrades to "rough estimates" but never blocks.

---

## 9. Tradeoffs & alternatives

### Decision: H3 over S2 / Geohash / quadtree

| Option | Pros | Cons | When to pick |
|---|---|---|---|
| **Geohash** | Trivially simple, lex-prefix = containment | Cells distorted by latitude; non-uniform neighbors; seam issues | Tiny system, English-keyed analytics |
| **S2** | Mathematically rigorous, global covering | Quadrilateral cells, library complexity | Global catalog of geo-tagged objects (where uniform area matters less than rigor) |
| **✅ H3** | Uniform 6-neighbor adjacency, multi-resolution, made for dispatch | Pentagons at 12 points (rare), less expressive for arbitrary polygons | **Any geo-dispatch problem at scale** |
| **Quadtree** | Adapts to density | Mutable index, hard to partition Kafka by | Static / low-write spatial data |

### Decision: Spanner / CockroachDB for trip state

| Option | Pros | Cons | When to pick |
|---|---|---|---|
| **PostgreSQL (single primary)** | Familiar, ACID, mature | No cross-region strong consistency without multi-master complexity | Single-region, smaller scale |
| **MySQL with Group Replication** | Cheap, common | Same single-region limit; tooling weaker | Same |
| **DynamoDB transactions** | Managed, scales | Vendor lock-in, transaction quotas, costly for our access pattern | AWS-native, light transactions |
| **✅ Spanner / CockroachDB / YugabyteDB** | Linearizable cross-region single-row + transactions; strong CAS | Operational complexity, cost, latency overhead (~30–50ms per write) | **Stateful workflows that span regions and demand strong consistency** |
| **Cassandra LWT (lightweight transactions)** | Cheap, in our stack | Paxos overhead per op, LWT slow under contention; eventual the rest of the time | Smaller blast radius use cases |

I'd defend Spanner with: "the trip row is the canonical example of a CRDT-resistant workflow — concurrent state changes from two actors that must serialize. Spanner gives us linearizable CAS in a managed package. PostgreSQL would force me into single-region or hand-roll multi-master."

### Decision: In-memory / Redis Geo for live driver index

| Option | Pros | Cons | When to pick |
|---|---|---|---|
| **PostGIS** | Mature, queryable | Write throughput tops out far below 1M/sec; index maintenance cost | Static geometry / lower write rate |
| **Elasticsearch geo_point** | Searchable, integrates with existing logging stack | Soft real-time, write amplification | Search-heavy workloads with geo as a filter |
| **MongoDB 2dsphere** | Easy schema | Same throughput problems as PostGIS at our scale | Smaller scale |
| **✅ In-memory per-cell process + Redis Geo mirror** | Sub-millisecond reads, scales linearly per cell, recoverable from Kafka | We own the recovery story; not a "database" | **Dispatch-class real-time spatial queries** |

### Decision: Kafka partitioned by H3 cell for location stream

| Option | Pros | Cons | When to pick |
|---|---|---|---|
| **Kinesis** | Managed | AWS lock-in; throughput per shard tighter than Kafka | AWS-only, smaller scale |
| **Pulsar** | Multi-tenant, geo-replication built-in | Newer; smaller community | If geo-replication is a primary requirement |
| **Pub/Sub (GCP)** | Managed, scales | At-least-once, no strict ordering per key | GCP-only |
| **✅ Kafka, partitioned by H3 cell** | Persistent, partition-ordered, replayable, per-cell affinity | Ops overhead | **Real-time data pipelines with locality semantics** |

### Decision: Cell-process per H3 cell vs single Matching Service

| Option | Pros | Cons | When to pick |
|---|---|---|---|
| Single global Matching Service | Simple | Doesn't scale; cross-cell traffic dominates; single point | Toy / proof of concept |
| **✅ Cell process per cell, scaled per market** | Linear scale; locality; fault isolation | Cross-cell coordination needed for boundaries | **Any real dispatch system** |
| Cell process per *city*, not per H3 | Coarser, fewer processes | Whole-city outages; hot intra-city cells overload one process | Small markets only |

### Decision: Windowed batched dispatch over greedy

| Option | Pros | Cons | When to pick |
|---|---|---|---|
| **Greedy nearest-driver** | No batching delay, simple | Globally suboptimal, ~10% worse utilization | Bootstrapping / very low density (one driver per cell) |
| **✅ Windowed batched (1–2s)** | Globally near-optimal, much better utilization | Adds 1–2s latency floor to match | **Any market with multiple drivers per cell at peak** |
| Continuous optimization | Even better quality | Hard to bound latency; harder to reason about | Research / high-density mature markets |

### Decision: Eventual consistency for location, strong for trip state

| Scenario | Consistency model | Mechanism |
|---|---|---|
| Driver location used to find candidates | Eventual (≤4s) | Kafka stream → in-memory; a slightly-stale candidate is fine |
| Trip state transitions | Linearizable on the trip row | Spanner CAS |
| Surge multiplier at quote time | Eventual (≤60s) | Flink-computed, Redis-cached |
| Driver "I'm online" status | Read-your-writes per driver session | Sticky-routed driver session on a cell process |

---

## 10. Defending architectural choices — interviewer attacks

> **Interviewer:** "Why not just use PostGIS with read replicas? You're reinventing a database."

**Candidate:** PostGIS is the right tool for *static* spatial data — city polygons, geofences, market boundaries — and we use it for that. For 1M location updates/sec, the index-maintenance cost dominates and the read tail blows past our 5ms budget. Even with read replicas, the *write* primary tops out. We'd end up sharding PostGIS by region and hand-rolling our own partition map — at which point we've reinvented the cell-process model with a slower data plane underneath. The simpler answer is to put the live index in process memory, accept that it's ephemeral, and rely on Kafka replay for recovery. PostGIS still owns the static geometry catalog.

> **Interviewer:** "Why not just use one giant Cassandra cluster for the trip state?"

**Candidate:** Cassandra's lightweight transactions are paxos-per-write, and they degrade badly under contention. The trip row is *contended* — driver and rider both touch it. Spanner / CockroachDB give us single-row linearizable CAS as a first-class operation, with predictable latency. Cassandra LWT works at low contention but our hot trips (peak hour, dense city) are exactly where contention spikes. So I pay ~30ms per Spanner write to avoid the long-tail risk on the most operationally important table in the system. For the *historical* archive — append-only, read-mostly — Cassandra is fine and that's where we land completed trips.

> **Interviewer:** "Why per-cell processes rather than per-driver actors? Wouldn't actors give you better isolation?"

**Candidate:** Per-driver actors over-isolate. The dominant query is "find me drivers in this region", which crosses thousands of actors and forces a scatter-gather. Per-cell aggregates exactly the right unit of locality: the things that ever appear together (a rider and nearby drivers) are in the same process. Per-driver actors would make sense if drivers had complex private state we wanted to evolve independently — but our driver state is small (last location, availability) and the matcher needs *aggregate* queries. The cell *is* the actor.

> **Interviewer:** "Your surge is 60s stale. Doesn't that miss flash demand?"

**Candidate:** It does — and we accept it. A surge multiplier that updates every second oscillates and produces price flicker, which users hate worse than slight staleness. The 60s window is a smoothing decision, not a tech limit. For genuinely flash events (concert lets out, weather hits), we have a separate "burst supply" trigger — a stream rule that publishes a step-function bump immediately to the surge cache, then the rolling window catches up. So 60s for steady-state, sub-second for genuine bursts. Two regimes, one cache.

> **Interviewer:** "What happens in a Kafka outage?"

**Candidate:** Three layers of impact:
- **Live driver index** stops getting updates. After ~30s, every driver looks "offline" and matching collapses.
- **Trip state** is in Spanner, untouched. In-flight trips continue.
- **Surge** keeps serving the last computed multiplier (60s old → 5 min old as the outage drags).

Mitigations:
- Multi-AZ Kafka with majority-quorum replicas → most "Kafka outages" are partial.
- For genuine cluster-wide outage: secondary Kafka cluster in a different region. Producers dual-write at low rate; on primary failure, full traffic flips. RTO ~60s.
- Driver app has local buffering (last 60s of updates) and re-emits on reconnection.
- The system gracefully degrades: matches stop in regions cut off from Kafka; trips already in flight are unaffected; the user-visible failure is "no drivers nearby" rather than data loss.

> **Interviewer:** "How do you prevent a driver from being assigned two trips simultaneously?"

**Candidate:** Two-layer defense:
1. **At the driver record:** the driver's `current_state` field is `AVAILABLE` or `ON_TRIP`. Matching only considers `AVAILABLE`. Transition to `ON_TRIP` is a CAS on the driver row, executed atomically with the trip's `state=ASSIGNED` update via a Spanner transaction.
2. **At the in-memory index:** when Matching Service makes an offer, it marks the driver as `OFFERED` in its local index for the offer-window duration. Even before Spanner confirms, no other matcher in the same cell can re-offer.

If a Spanner transaction fails (e.g., conflict with a concurrent driver state change), the matcher releases the in-memory hold and re-queues the request. The driver's truth lives in Spanner; the in-memory state is a hint.

> **Interviewer:** "What if two cells both want to offer the same boundary driver?"

**Candidate:** This is the cross-cell race. Cell A and Cell B both see a driver near their boundary. Both make offers.

Defense: the driver row's `assigned_cell` is updated as part of the same Spanner transaction that flips state to `OFFERED`. Whichever cell wins the CAS owns the driver for the offer window. The losing cell's offer fails on commit and re-queues the request.

This is a small race that resolves in milliseconds. The pathological failure mode (offer ping-ponging) is rare; observability can catch it via a metric "cross-cell offer collision rate".

---

## 11. Failure modes & mitigations

| Failure | Detection | Blast radius | Mitigation | Recovery |
|---|---|---|---|---|
| Location ingest lag | Kafka producer lag, end-to-end staleness metric | Drivers look "stale", matches degrade in affected cells | Auto-scale ingest pods; circuit break drivers from far regions; degrade match window | Resolves as ingest catches up |
| Cell process crash | Liveness probe, Kafka consumer rebalance event | One cell — matching pauses for that cell for ~5–15s | Kafka rebalance; another instance adopts the cell; in-memory index re-populates from the topic in 4s | Auto-recover; logged as routine |
| Trip state Spanner unavailable | DB error rate, write latency spike | All trip state writes blocked in affected region | Read-only mode for in-flight trips (rider still sees driver location), reject new requests, multi-region failover | Spanner manages most cases automatically; on full region failure, traffic shifts to next region |
| Matching Service falls behind (window backlog) | Match latency p99, queue depth | Riders see "still searching" → cancel | Auto-scale; widen H3 search radius; reduce optimization complexity (greedy fallback) | Backlog drains; tune worker count |
| Surge model staleness | Flink consumer lag | Surge frozen at last value; pricing slightly off | Acceptable for minutes; alert at >5 min lag | Restart Flink job from checkpoint; surge resumes |
| Driver app loses connectivity mid-trip | Heartbeat timeout per driver | The trip — rider's location stream stops updating, but trip continues in DB | Reconciler reassigns if pre-pickup; ops escalation if post-pickup | Driver reconnects, resumes |
| Notification Service down | Driver app push delivery failure rate | Drivers don't see offers | Fall back to in-app polling; longer offer expiry | Restore service; drained offer queue replays |
| ETA Service down | 5xx rate / fallback rate metric | Match quality degrades but matches happen | Use straight-line distance × regional speed for match scoring | Restore; cache rebuilds |
| Hot cell (concert lets out) | Per-cell QPS, match latency | One cell overwhelmed | Subdivide partitions; pull from neighbors more aggressively; pre-position drivers via incentive nudges | Surge naturally rebalances over minutes |
| Cross-region network partition | DNS / inter-region connectivity probes | Cells in cut-off region operate independently | Cells are designed to operate self-contained per market; trip state in regional Spanner replica continues with local consistency | On heal, Spanner reconciles |
| DDoS on rider request endpoint | Edge WAF metrics | Could exhaust quote / trip create capacity | Rate limit per IP / device at edge; bot detection; CAPTCHA escalation | Throttle bad actors; capacity holds |
| Bad code push to dispatch | Canary error rate | Per-market — only the cell where canary is deployed | Per-market canary, auto-rollback on SLO breach | Roll back; investigate |
| Driver double-booking (two trips assigned) | Trip-row CAS-failure metric, per-driver concurrent-trip alarm | Both riders impacted; reputational | Two-layer defense (driver state CAS + in-memory hold); alarm should be ~zero rate | If detected: cancel one offer, take engineering postmortem |
| Trip-state row corruption | Audit checks, CDC consumer drift | One trip | Spanner snapshots; replay from state_transitions audit array | Manual repair from audit; root-cause |

---

## 12. Productionization

### Pre-launch

- [ ] **Simulator.** Replay 6 months of historical trip-request and driver-location traces against the new dispatcher in shadow mode. Compare match outcomes (which driver, ETA, surge) against ground truth. Block launch if regression on key metrics > 2%.
- [ ] **Load test** to 5× projected peak ingest (5M updates/sec). Verify in-memory index doesn't degrade, Spanner write p99 < 100ms, match p99 < 5s.
- [ ] **Chaos test.** Kill random cell processes during load; kill a Kafka broker; saturate the network between regions. Verify no trip data loss, recovery within SLO.
- [ ] **Game day** with ops + customer support: simulate a regional Spanner outage. Walk through customer-support implications, runbook, and external comms.

### Rollout

- [ ] **Per-market rollout.** New dispatcher launches in one Tier-3 city first (low traffic, low blast radius). Bake 2 weeks. Observe match rate, time-to-match, cancellation rate.
- [ ] **Tier-3 → Tier-2 → Tier-1.** Last to upgrade: NYC, SF, Chicago, London, where the cost of a bad day is highest.
- [ ] **Per-cell A/B within a market.** New matching algorithm? A/B between adjacent cells (with care — boundary effects). Compare driver utilization, rider ETA, cancellation.
- [ ] **Kill switch.** A dynamic config can fall back the entire matcher to "greedy nearest" for any market within seconds. This is non-negotiable.
- [ ] **Auto-rollback** on: match latency p99 > 8s for 5 min, or cancellation rate up > 10% rolling, or 5xx > 1%.

### Capacity

- [ ] Reserve 2× headroom on the location ingest pipeline. Drivers don't pause for our deploy.
- [ ] Auto-scale cell processes on (in-memory size, kafka lag, match latency).
- [ ] Spanner: provision for 2× current write rate; growth requires planning (Spanner scale isn't instant).
- [ ] Kafka: 5× partition headroom on hot markets; repartition is painful.

### Migration (if replacing existing dispatch)

- [ ] **Dual dispatch shadow.** New dispatcher runs alongside old, makes its own match decisions, but the old system's decisions are what's actually executed. Compare match-by-match for 4 weeks per market.
- [ ] **Cutover per cell.** A single H3 cell can flip to the new dispatcher; rest of the market stays on the old. This is the smallest unit of rollback.
- [ ] **Old system retained for 60 days** as fallback after full cutover.

### Cost

- Estimated $/trip: dominated by location pipeline + map/routing API. At our scale we own internal mapping (Mapbox/Google would cost ~$1B/year if we paid retail — at scale we negotiate or self-host).
- Cost levers:
  - Adaptive driver location cadence (15s when stationary, 4s when moving). Saves ~30% of ingest cost.
  - Per-region location compute (don't replicate driver streams across regions).
  - In-house mapping & routing (huge save vs paying per query).
  - Spot capacity for batch / non-trip-critical compute.
  - Per-market sizing — Tier-3 markets share infrastructure where they can.

### Compliance

- Driver / rider PII encrypted at rest; access logged.
- Per-market data residency — EU drivers and riders' data stays in EU; no cross-region replication for protected fields.
- Trip records retained 7 years for tax/audit; archived to cold storage after 90 days.
- Right-to-erasure: tombstone in `trips`; PII fields scrubbed on request; aggregate reporting unaffected.

---

## 13. Monitoring & metrics

### Business KPIs
- Trips/day per market
- Cancellation rate (rider cancel, driver cancel, dispatcher fail)
- Driver utilization (% of online time on a trip)
- Time to match (rider perceived)
- Surge fairness (% of trips with surge >2×; complaints)
- ETA accuracy (|predicted - actual| at completion)

### Service-level (SLIs)
- **Match flow** (the front door): time-to-match p50/p95/p99/p99.9, per market.
- **Trip transition write** latency, error rate.
- **Location ingest** lag p99 (now − last-event-ts).
- **SLO**: 99.95% of match attempts succeed within 5s p99; 99.99% trip-state availability per market.

### Component metrics

#### Location ingest
- Updates/sec ingested per region
- Per-driver update cadence histogram (detect bad clients)
- Validation rejection rate
- Kafka publish latency
- Per-cell update rate (detect hot cells)

#### Cell process
- In-memory index size (drivers per cell)
- Read query latency (matcher → cell process)
- Kafka consumer lag for owned partitions
- Memory utilization, GC pauses
- Eviction rate (drivers expired)

#### Matching Service
- Match window depth (requests per batch)
- Time per match window (compute time)
- Match success rate per cell
- Acceptance rate from drivers
- Decline / timeout rate
- Cycles per match (1st offer accepted vs 5th)
- Cross-cell offer collision rate

#### Trip Service / Spanner
- Write latency p99
- CAS failure rate (state-transition contention)
- Per-region replication lag
- Active trip count per market
- Trip-completion rate

#### Surge / Pricing
- Flink consumer lag
- Time since last surge update per cell
- Distribution of multipliers (detect oscillation)
- Pricing-service p99

#### ETA Service
- p99 inference latency
- Cache hit rate
- Predicted vs actual MAE
- Fallback-to-distance-only rate

### Alert routing
- **Page** (24/7): match-rate SLO breach in any market, location ingest lag > 30s, Spanner regional outage, per-driver double-assignment rate > 0
- **Ticket**: cell process restart frequency, cache hit rate degraded, ETA accuracy regression
- **Dashboard-only**: cost dashboards, capacity headroom, model A/B results

### Dashboards
1. **Dispatch health** per market — match latency, success rate, error budget
2. **Location pipeline** — ingest lag, throughput, hot cells
3. **Trip lifecycle** — state distribution, transition timings
4. **Surge** — multipliers per market, oscillation, fairness
5. **Driver utilization** — per market, per shift
6. **Per-cell heatmap** — supply vs demand vs surge by H3 cell on a city map

---

## 14. Security & privacy

- **Driver and rider PII** (real names, phone, payment) is held in separate stores from the trip dispatch path. Dispatch only sees opaque IDs.
- **Location data** is highly sensitive. Driver location during a trip is shared with the rider; idle driver location is not. Compliance tooling enforces this.
- **mTLS** between all internal services; external endpoints behind OAuth/OIDC.
- **Per-driver / per-rider rate limits** at edge; anomaly detection on request velocity, geographic implausibility, multi-account device.
- **Anti-fraud integration:** trip / driver / rider have hooks into a fraud service. The dispatcher itself doesn't decide; it asks "is this trip OK to accept?" via an interface.
- **GDPR / CCPA / India DPDP / China PIPL:** per-market data residency. Right-to-erasure scrubs PII; aggregate trip data retained for legal / tax purposes.
- **Audit logs:** every trip state transition, every match decision, every surge computation logged. Immutable, queryable for legal / customer-support.
- **Driver-app side-channels:** location pings are signed by a driver session token to prevent location spoofing. Detected spoofing → flag for review, exclude from matching.

---

## 15. Cost analysis

Rough order-of-magnitude:

| Component | Rough cost driver | Optimization lever |
|---|---|---|
| Location ingest pipeline | Per-driver-hour ingest cost (dominant) | Adaptive cadence (4s active / 15s stationary); edge aggregation; per-region compute |
| Maps / routing API | Per-routing-query (huge if external) | Own internal map and routing service (pays for itself >100M trips/yr) |
| Spanner / CockroachDB | Per-write × replication factor × multi-region | Co-locate trip writes in home region; archive completed trips to Cassandra after N days |
| Cassandra (history) | Storage × retention | Tiered storage; compress; TTL very old data |
| Redis (geo + surge) | RAM × cluster count | Right-size per market; aggressive TTL |
| Kafka | Disk × retention × replication | Retention 60s for live location stream is enough |
| Cell-process compute | CPU × per-cell instance count | Per-market sizing — Tier-3 markets share processes |
| ML inference (ETA, surge, acceptance) | GPU / specialized infra | Model distillation; cache predictions aggressively |

Largest cost: location ingest + maps. At 5M concurrent drivers × 8 hours × 365 days that's ~15 trillion location updates per year. Even at $0.0000001 per update (favorable), that's $1.5M/year just for ingest, not counting downstream compute. Engineering time spent on this lever pays back orders of magnitude.

The **dispatch compute itself** (matchers, cell processes) is small compared to ingest — typically <10% of the dispatch infra bill.

---

## 16. Open questions / what I'd validate with PM

These are real Staff-level instincts — surfacing what you don't know.

1. **Multi-market driver licensing.** Are drivers strictly per-market, or do some operate across (e.g., NJ↔NYC)? Affects cell membership rules.
2. **Pool / shared rides** — separate problem or layered on top? Pool changes the matching problem materially (multi-rider routing).
3. **Pre-booked / scheduled rides** — different dispatch flow (we *plan* assignment hours ahead) or layered on real-time? Drives a separate optimization service.
4. **Driver acceptance rate as part of match score** — does the business want "fair" rotation (treat all drivers equally) or pure utility (match to whoever is most likely to accept)? Has revenue and equity implications.
5. **DoorDash batched delivery** policy — when two orders are nearby, do we batch automatically or only with explicit promo? Customer expectation matters.
6. **Real-time data sharing** with regulators (NYC TLC, EU) — affects data export pipeline design and PII tagging.
7. **Multi-modal expansion** (bikes, scooters, public transit handoff). Does the cell model extend? Generally yes; we'd add vehicle-class filters.

---

## 17. Staff-level scorecard for this problem

| Signal | Did the candidate... |
|---|---|
| ✅ Identified geo-indexing + matching algorithm as the central deep-dives | Within first 10 min, not as an afterthought |
| ✅ Quantified the location pipeline as the dominant cost | "750K updates/sec dominates everything" calculation made explicit |
| ✅ Picked H3 with reasoning (uniform adjacency, multi-resolution) | Not "I'd use a geohash because it's simple" |
| ✅ Defended Spanner / CockroachDB for trip state | Not handwaving "we'd use a SQL DB" |
| ✅ Volunteered windowed batched matching over greedy | Most candidates default to greedy and never mention batching |
| ✅ Treated surge as a streaming aggregator, not a per-request compute | Recognized it as a data-pipeline problem |
| ✅ Discussed cellular per-market isolation | Volunteered, didn't wait for the prompt |
| ✅ Volunteered failure modes (driver offline mid-trip, dispatcher crash, Kafka outage) | All before being asked |
| ✅ Discussed productionization unprompted | Per-market rollout, kill switch, simulator |
| ✅ Talked about cost in concrete terms | Identified location ingest + maps as cost dominators |
| ✅ Asked the interviewer at decision points | "Should I deep-dive matching algorithm or location pipeline?" |
| ✅ Closed with a wrap-up speech | Restated dominant constraints, choices, what to do next |

### What separates Staff from Senior on *this* problem

A Senior candidate will:
- Correctly draw the cell-process + Kafka + matcher diagram
- Pick H3 (or at least a reasonable geo-index)
- Mention sharding by region

A Staff candidate will additionally:
- Quantify the **location ingest cost** and propose adaptive-cadence as a lever
- Pick **windowed batching** over greedy and explain why
- Treat the **trip row** as the hardest consistency problem and pick Spanner-class storage with reasoning
- Identify **per-market cellular** as the natural isolation, not as an afterthought
- Discuss **cross-cell** scenarios (airport, boundary races) explicitly
- Volunteer the **dispatcher kill switch** and per-market canary as productionization staples
- Distinguish **surge as a stream** from the request-time pricing call
- Bring a **simulator** to pre-launch validation — replay-against-history is a Staff move

---

## 18. Wrap-up speech (template)

> "To wrap up: the dominant constraints here are the location-ingest write rate (~750K updates/sec, the cost dominator) and the consistency requirement on the trip row (linearizable, two services concurrently mutating a 30-minute-long workflow). The architecture answers both: H3-cell-keyed Kafka pipelines feeding per-cell in-memory geo indices for the dispatch hot path, and Spanner-backed trip state with conditional-write transitions for the lifecycle. The matcher is windowed batched (1–2s) running per cell, scoring (request, driver) pairs on ETA + acceptance probability + supply-demand pressure. Surge is a Flink streaming aggregator publishing per-cell multipliers every 60s with a fast-path bump for genuine flash demand. The two pieces I'd worry about in production are (1) location-ingest lag during a Kafka or network event — mitigated by multi-AZ Kafka and graceful degradation that treats stale drivers as offline, and (2) hot-cell saturation during real-world peaks (concert, weather) — mitigated by partition subdivision and aggressive cross-cell pulling. To productionize: per-market rollout starting Tier-3, simulator-validated against 6 months of historical traces, dynamic kill switch falling back to greedy matching, SLO-alerted on time-to-match p99 and trip-state availability. If we had another 30 minutes, I'd deep-dive (a) the ML-served acceptance-rate predictor and how it feeds the matcher's objective function, or (b) the DoorDash multi-leg batched-courier dispatch problem, which adds prep-time prediction and order bundling on top of this same spine."

---

## 19. Market-based isolation (cellular architecture) — the natural fit

> Bring this up confidently: dispatch is the textbook example of a system that's *naturally* cellular. Unlike news feed (where cells are imposed for compliance), dispatch *is* per-market by physics. Frame it as: *"This system isn't just compatible with cellular — it's already cellular at the domain level. The question is how explicitly we model it in the architecture."*

### What & why

A **market** is a city or metro area: NYC, SF, London, São Paulo, Mumbai. A **cell** is the entire dispatch stack for one market: its own location-ingest, Kafka, cell processes, matcher, trip-state Spanner replica, surge model, ETA model.

Drivers (in priority order at FAANG/Uber scale):

1. **Physical reality.** A driver in NYC will not serve a rider in SF. Ever. The two markets have *zero* shared dispatch traffic. Sharing infrastructure between them buys nothing and adds risk. This isn't a compliance argument — it's a "the workload is naturally partitioned" argument.
2. **Blast radius reduction.** A bad config push to NYC's matcher should not stop SF's dispatch. Per-cell deploy = per-cell blast radius.
3. **Per-market product variation.** NYC has a yellow-cab integration that London doesn't. India has cash payments. Mumbai has auto-rickshaws as a vehicle class. Per-market deploys let teams ship per-market features without coupling.
4. **Per-market ML models.** Surge elasticity in NYC ≠ Mumbai. ETA models trained on NYC's grid are useless in São Paulo. Each market trains its own.
5. **Compliance.** EU markets must keep PII in-EU. Same for India, China, Brazil. Cellular-by-market is the only design that makes this clean.
6. **Cost attribution.** PMs and finance want to know what serving NYC costs vs what serving Mumbai costs. Cellular makes per-market P&L trivial.
7. **Operational independence.** NYC outage runbook is run by NYC oncall. They don't wake up the global team. Mumbai's launch issues don't block London's release.

### Architecture

```
        ┌─────────────────────── Global Edge ───────────────────────┐
        │  GeoDNS / Anycast routes user → home market               │
        └────────┬─────────────────┬──────────────────┬─────────────┘
                 │                 │                  │
       ┌─────────▼────────┐ ┌─────▼────────┐ ┌──────▼────────┐
       │   NYC Cell       │ │  London Cell │ │  Mumbai Cell  │
       │ ┌──────────────┐ │ │ ┌──────────┐ │ │ ┌──────────┐  │
       │ │ Loc Ingest   │ │ │ │Loc Ingest│ │ │ │Loc Ingest│  │
       │ │ Kafka        │ │ │ │Kafka     │ │ │ │Kafka     │  │
       │ │ Cell procs   │ │ │ │Cell procs│ │ │ │Cell procs│  │
       │ │ Matcher      │ │ │ │Matcher   │ │ │ │Matcher   │  │
       │ │ Trip Spanner │ │ │ │Trip Span │ │ │ │Trip Span │  │
       │ │ Surge (Flink)│ │ │ │Surge     │ │ │ │Surge     │  │
       │ │ ETA model    │ │ │ │ETA model │ │ │ │ETA model │  │
       │ └──────────────┘ │ │ └──────────┘ │ │ └──────────┘  │
       └─────────┬────────┘ └─────┬────────┘ └──────┬────────┘
                 │                │                  │
                 └────────────────┴──────────────────┘
                              │
                  ┌───────────▼──────────────┐
                  │  Cross-Cell Bridge       │
                  │  - User → home market    │
                  │    directory             │
                  │  - Airport / inter-city  │
                  │    handoff               │
                  │  - Cross-market trip     │
                  │    settlement            │
                  └──────────────────────────┘
```

### Routing

- Each user (rider and driver) has a **home market**, set at signup based on country / city.
- A small global **User Directory** (replicated KV) maps `user_id → home_market`. Cached aggressively at the edge.
- A request with a (lat, lng) from a rider in NYC's polygon hits the NYC cell. We use the *current location* (not the home market) to route, with home market as the fallback if location is unavailable.
- A driver only operates in their licensed markets; their cell membership is enforced at the edge.

### The hard part: cross-cell scenarios

There are three real cross-market scenarios:

1. **Airport pickups.** SFO is geographically in San Francisco. A driver based at SFO might pick up a rider going to San Jose. SFO is on a **cell boundary** (San Francisco market vs San Mateo / Peninsula market). Solution: airports are *multi-market cells*, members of both markets. Drivers can opt into airport pickup. Matcher for the airport cell can pull from either market's driver pool.
2. **Inter-city trips** (rider in NYC books trip to Boston). Rare but real. Solved by:
   - The trip is dispatched in NYC (rider's market).
   - The driver chosen is one licensed for inter-city.
   - On crossing into Boston market, the trip remains under NYC cell's control (single owning cell per trip simplifies state).
   - At Boston dropoff, no handoff is needed — the driver returns home empty (or accepts a Boston offer if licensed).
3. **Cross-market driver presence** (driver based in Newark drives into NYC). Newark and NYC are different markets even though geographically close. Solution: enforce licensing at edge — driver cannot go online in NYC unless licensed. If they want to, they update their licensing through onboarding.

The Cross-Cell Bridge is small — it owns these specific cross-market operations and the global User Directory. **Most traffic — >95% — never crosses cells.**

### Per-cell vs single global

| Dimension | Single global | Cellular per-market | Net |
|---|---|---|---|
| Total infra cost | Lower (no per-market duplication of compute) | Higher: per-market deploys, separate Spanner clusters, separate Kafka | +20–40% but easily justified |
| Engineering complexity | Lower up front | Per-cell awareness in code, deploys, oncall | Real tax — pays back at >5 markets |
| Compliance | Hard | Easier (data residency by construction) | Cellular wins |
| Blast radius | 100% on global outage | 1 market at a time | Cellular wins |
| Latency for in-market reads | Higher (wider replication, cross-region) | Lower (everything in-region) | Cellular wins for ≥99% of traffic |
| Cross-market consistency | N/A (everything is in one place) | New problem to solve (User Directory, Bridge) | Cellular pays a tax — but small (95% of traffic doesn't cross) |
| ML personalization | One model, less data per region | Per-market models, more relevant | Cellular wins for product quality |

### What changes in the design

- **User Directory** — global, replicated, sub-ms reads at edge.
- **Cross-Cell Bridge** — owns inter-cell handoff and cross-market reporting.
- **Per-cell Kafka, per-cell Spanner replicas, per-cell cell-processes** — no shared dispatch state across cells.
- **Per-cell ML models** — per-market surge model, per-market ETA model, per-market acceptance-rate predictor.
- **Per-cell rollouts** — feature flags now have a `market` dimension. Roll a feature out to Mumbai first, observe, then NYC.
- **Per-cell oncall** — Tier-1 markets (NYC, SF, London, Mumbai, São Paulo) have dedicated oncall; Tier-3 markets share regional oncall.
- **Per-cell capacity sizing** — NYC's peak is 100× a Tier-3 city's peak. We size per cell, not per company.
- **Failover** — if NYC cell is degraded: do we route NYC users to NJ cell? Generally **no** — it's a different market with different licensing. The cell IS the failure domain; intra-cell HA (multi-AZ within NYC) is what protects users.

### Tier-1 vs Tier-3 markets

Not every market gets the same investment.

- **Tier-1** (NYC, SF, London, Mumbai, São Paulo, Tokyo): full dedicated stack, dedicated oncall, custom ML models, dedicated capacity.
- **Tier-2** (Boston, Seattle, Berlin, Bengaluru): full dedicated stack, shared oncall with neighboring markets.
- **Tier-3** (smaller US cities, smaller European cities): shared compute infrastructure across multiple markets in the same region; per-market trip-state data; shared ML models with per-market fine-tuning.

The cell IS the unit; how richly we resource it varies by market revenue and complexity.

### When NOT to do cellular

- Single-city startup. Pre-50K trips/day. Just run one stack.
- Two adjacent markets that *do* share traffic (rare, but conceivable in a small country).
- A team too small to operate N parallel deploys.

### What "Staff signal" sounds like here

> "Dispatch is the textbook example where cellular isn't a choice — it's a recognition of physics. The workload partitions naturally by city; sharing infrastructure between NYC and Mumbai costs more (in latency, blast radius, compliance) than it saves. So I'd build cellular per-market from day one *if* we're launching in multiple markets, with the Cross-Cell Bridge handling the airport-and-inter-city long tail. Tier-1 markets get rich resourcing; Tier-3 markets share infrastructure where the workload allows. The hard part isn't the cells — it's the User Directory and the airport edge cases. Designing for cells is a small upfront tax that pays back the first time a bad config push hits NYC and SF keeps running."
