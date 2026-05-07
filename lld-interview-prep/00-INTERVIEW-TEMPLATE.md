# LLD Interview Template — Lightweight Edition

A scannable framework to follow in any LLD interview. Read it before walking in.

## The 7 phases (60-min interview)

| # | Phase | Time | Key question | Output |
|---|---|---|---|---|
| 1 | **Clarify** | 5m | What ops? Scale? In-scope? | 3-line scope note |
| 2 | **Model** | 5m | Entities vs value objects? Aggregates? | List of nouns + types |
| 3 | **Diagram** | 5m | Patterns at variation points? | Class diagram |
| 4 | **Code** | 10m | Load-bearing classes only | Interfaces + orchestrator |
| 5 | **Concurrency** | 5m | What's contested? Granularity? | Lock/CAS choice + reasoning |
| 6 | **Edges** | 5m | What can go wrong? | Edge case table |
| 7 | **Extend** | 10m | Add a feature? Scale? Persist? | Forward-looking discussion |
| — | Buffer | 15m | Follow-up Qs | — |

---

## Phase 1 — Clarify

Ask in order:
1. Core operations + actors
2. Concurrency? Scale? Persistence?
3. Out of scope (volunteer specifics)
4. Failure modes

> Say: "Before I design — core ops are X/Y/Z, I'll assume multi-user with in-memory + repository abstraction, and A/B are out of scope. Sound right?"

---

## Phase 2 — Model

For each noun ask:
- **Entity** (has ID, mutable lifecycle) or **value object** (immutable, equality by content)?
- Which are **aggregate roots**?
- Cross-aggregate refs **by ID**, in-aggregate refs **direct**.

Defaults: `BigDecimal` for money. `Instant` for time. Records for value objects. Inject `Clock`.

---

## Phase 3 — Diagram

Pick the right arrow:
- `◆` composition (owned)  ·  `◇` aggregation (shared)
- `△` inheritance  ·  `△ ┄` realization (interface)
- `─►` association  ·  `┄►` dependency

Tag each pattern explicitly:

| When you see… | Reach for… |
|---|---|
| Multiple algorithms | Strategy |
| Lifecycle / states | State |
| Tree / hierarchy | Composite |
| Pluggable wrappers | Decorator |
| Vendor SDK | Adapter |
| Pipeline / filter chain | Chain of Responsibility |
| One-to-many notify | Observer |
| Request as object | Command |
| Snapshot / undo | Memento |
| Many-to-many coordination | Mediator |
| Many optional fields | Builder |
| Type-driven creation | Factory |

---

## Phase 4 — Code

Write:
- Key **interfaces** (full method signatures)
- The **orchestrator** (composing the parts)
- **One** representative implementation per pattern

Skip: imports, trivial getters, full enum switches, all DTO boilerplate.

Senior markers: constructor injection, `final` fields, `Optional<T>` returns, idempotency keys, repository interfaces.

---

## Phase 5 — Concurrency

The 5-step walk:
1. Name the shared mutable state
2. Identify readers / writers / concurrency
3. Pick the right primitive
4. Justify granularity (why not coarser, why not finer)
5. Failure modes

| Need | Use |
|---|---|
| Single field atomic | `AtomicReference` + CAS |
| Counter | `AtomicLong` |
| Keyed map | `ConcurrentHashMap` |
| Listener list | `CopyOnWriteArrayList` |
| Producer/consumer | `BlockingQueue` |
| Multi-field invariant | `synchronized` on dedicated lock |
| Distributed | Optimistic version + idempotency key |

Rules: **never hold a lock across I/O**. Specify optimistic vs pessimistic.

---

## Phase 6 — Edges

Walk through these categories:
- Invalid input
- Wrong-state operation
- Concurrent same operation
- Resource exhausted
- TTL expiry mid-flow
- Cross-service failure
- Crash mid-flow → reconciliation
- Idempotent retry behavior

Format as a table: case → handling.

---

## Phase 7 — Extend / Scale / Persist

**Extensions**: name the seam. "New X = one new class implementing Y interface."

**Scale**: bottleneck today → first lever (shard / cache / queue) → next bottleneck.

**Persistence options**:

| Pick | When |
|---|---|
| PostgreSQL | Default; ACID; secondary indexes |
| Redis | Hot cache, rate limits, sessions, TTL |
| DynamoDB | Cloud-native, conditional writes, predictable scale |
| Time-series | Telemetry, logs, metrics |
| Event sourcing | Audit-heavy, replay, time-travel |

For each, name the concurrency primitive (`UPDATE ... WHERE version = ?`, Redis Lua, DynamoDB conditional).

---

## Universal must-haves (memorize)

- `BigDecimal` for money
- `Instant` for time (not `LocalDateTime`)
- Inject `Clock`, `IdGenerator`, `Random`
- Constructor injection (no `new` for collaborators)
- Repository interface + in-memory impl
- Idempotency keys on cross-service calls
- Versioning for optimistic concurrency
- Strategy for variation points
- State pattern for lifecycles
- Records for value objects

---

## Top anti-patterns to avoid

- Coding before clarifying
- `double` for money
- `synchronized` everywhere as the concurrency answer
- `if (status == X)` chains (missed State)
- `instanceof` in business logic
- `new SomeService()` inside business logic
- Returning `null` over `Optional`
- One giant aggregate
- No mention of failure modes

---

## Opening line

> "Let me clarify scope before designing — core ops are X/Y/Z, multi-user with in-memory + repository abstraction, A/B out of scope. Anything you'd change?"

## Closing line

> "Patterns: [A] for [variation], [B] for [variation]. Concurrency: [granularity] because [reason]. Persistence behind repo, so swap is one class. Extension points: [list]. Biggest risk: [failure mode], handled by [mitigation]."
