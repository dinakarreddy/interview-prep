# LLD Interview Prep — Canonical Problems

A working set of 16 classic Low-Level Design interview problems, each written end-to-end as if you were sitting through the interview itself. Every file walks through:

1. The **opener** the interviewer typically gives (one sentence)
2. The **clarification phase** — back-and-forth Q&A you should drive
3. **Refined requirements** (functional + non-functional + out-of-scope)
4. **Domain model** (entities, value objects, enums, with rationale)
5. **Class diagram** (ASCII, with patterns labeled)
6. **Key abstractions** (interfaces and the patterns they realize)
7. **Core code** (the load-bearing classes — Java)
8. **Concurrency strategy** (what's contested, why this lock granularity)
9. **Edge cases / gotchas** (table form)
10. **Extensions / interview follow-ups** (the next questions you'll get)
11. **Scaling** (what changes when one process / one machine isn't enough)
12. **Persistence tradeoffs** (what changes if we swap in-memory for SQL / Redis / DynamoDB / time-series / event sourcing)
13. **Talking points** (lines that earn senior-level credit)

---

## Index

| # | Problem | Headline pattern(s) |
|---|---|---|
| 01 | Parking Lot | Strategy, CAS-per-spot |
| 02 | Vending Machine | State, Strategy |
| 03 | LRU Cache | Doubly-linked list + HashMap, eviction Strategy |
| 04 | Logger | Chain of Responsibility, Strategy, async dispatch |
| 05 | Snake & Ladder | State, turn queue, dice Strategy |
| 06 | Chess | State, Strategy (per piece), Command + Memento |
| 07 | ATM | State, Command, transaction atomicity |
| 08 | Elevator | State, dispatch Strategy, priority queue |
| 09 | Splitwise | Strategy (split type), graph balance |
| 10 | Movie Booking | Repository, optimistic locking, hold-then-confirm |
| 11 | Online Shopping | Many: Strategy, State, Composite, Repository, Saga |
| 12 | Notification System | Adapter, Strategy, Decorator, retry |
| 13 | Rate Limiter | Strategy (algo), atomic token bucket |
| 14 | URL Shortener | Strategy (slug gen), atomic counter |
| 15 | Cab Booking | Spatial index, State, dispatch Strategy |
| 16 | Stock Exchange | Priority queues, single-writer per symbol |

---

## How to use

- **First read**: skim the index, pick a problem, read end-to-end. The format is the same across files so you can build muscle memory.
- **Mock interview**: cover the file below the "Refined requirements" line, write your own design, then compare.
- **Reference**: when you're stuck on a real problem, the patterns table tells you which canonical problem to crib from.

---

## Cross-cutting principles applied throughout

These show up in nearly every problem and earn senior-level credit when mentioned:

### Design
- **Favor composition over inheritance.** Strategy/Decorator/Bridge over deep hierarchies.
- **Depend on abstractions.** Interfaces injected via constructors; no `new` for collaborators in business logic.
- **Single responsibility.** Each class names a thing; method names use the domain language, not implementation jargon.
- **Immutable value objects.** `Money`, `Address`, `Coordinate`, `DateRange` — records / `final` fields, defensive copies.
- **Aggregate boundaries.** References inside an aggregate are direct; across aggregates are by ID.

### Concurrency
- **Identify what's actually contested.** Lock at that granularity, not larger.
- **Prefer atomics and lock-free structures** (`ConcurrentHashMap`, CAS) for hot paths.
- **Never hold a lock across I/O.** Capture state, release lock, do I/O.
- **Document failure modes.** Crash recovery, timeouts, retries with backoff, idempotency keys.

### Data modeling
- **Entities have an ID; value objects don't.**
- **Repositories are domain-shaped**, not generic CRUD. `findActiveByUser` over `findAll`.
- **Versioning** on entities that are concurrently editable (optimistic locking).

### API design
- **Make it hard to misuse.** Required params in constructors; types encode constraints (`PositiveInt`, `EmailAddress`).
- **Return abstractions, not concretions** (`List<T>`, not `ArrayList<T>`).
- **Document the contract** — nullability, threading, exceptions, idempotency.

### Testability
- **Inject `Clock`, `IdGenerator`, `Random`.** Don't call `Instant.now()` directly in domain code.
- **Repositories have in-memory implementations** for unit tests.
- **Strategies are mockable** — every variation point is a seam.

---

## Pattern → problem reverse lookup

When the interviewer hints at a pattern, recognize it:

| Their wording | Pattern they're hinting at |
|---|---|
| "Multiple ways to do X / pluggable algorithm" | Strategy |
| "Determined at runtime by config / input" | Factory |
| "Several variants need to work together" | Abstract Factory |
| "Many optional parameters / fluent setup" | Builder |
| "Only one instance / shared resource" | Singleton (DI'd, not getInstance()) |
| "When X changes, others should react" | Observer |
| "Goes through stages / lifecycle / status" | State |
| "Wrap to add behavior / pipeline" | Decorator |
| "Translate / integrate third-party" | Adapter |
| "Tree / hierarchy / contains itself" | Composite |
| "Undo / redo / queue / log requests" | Command |
| "Pipeline / middleware / filter chain" | Chain of Responsibility |
| "Lazy load / cache / wrap with permission" | Proxy |
| "Add operations to a fixed hierarchy" | Visitor |
| "Workflow with customizable steps" | Template Method |
| "Coordinator between many components" | Mediator |
| "Snapshot and restore" | Memento |
| "Million tiny similar objects" | Flyweight |
| "Two axes of variation" | Bridge |

---

## Recurring shapes of LLD problems

Most problems map onto one or two of these archetypes — recognizing the shape early collapses your search space:

- **State machine with operations** — Vending Machine, ATM, Order, Booking, Document workflow.
- **Registry / dispatcher / matcher** — Notification system, Logger, Vehicle parking, Cab dispatch.
- **Tree / hierarchy** — File system, UI components, Org chart, Menu, Expression tree.
- **Observer / event-driven** — Stock ticker, Order tracking, Game state, Chat.
- **Resource manager / pool** — Connection pool, Parking lot, Movie seats, Elevator.
- **Pricing / pipeline** — Order pricing, Tax calc, Insurance premium, Discount engine.
- **Cache / lookup with eviction** — LRU, DNS, browser history.
- **Game / turn-based** — Chess, Tic-Tac-Toe, Snakes & Ladders.
- **Simulation / scheduler** — Elevator, Cab dispatch, Auction.
- **Editor / undo-redo** — Text editor, Drawing app, Spreadsheet.

---

## First 10 minutes — the universal framework

Regardless of the specific problem, do these in order:

### Minute 1–3: Clarify
- Functional: what operations does it support? Who are the actors? Happy path?
- Non-functional: scale, latency, durability, consistency, multi-user vs single-user, persistent vs in-memory.
- Out-of-scope: pin down what's NOT expected.
- Edge cases: failure? Concurrency? Cancellation?

Write down the **3–5 core operations** the system must support. Almost everything else cascades from these.

### Minute 4–6: Identify entities and value objects
- What are the **nouns** in the requirements?
- For each: entity (has ID, lifecycle) or value object (immutable, equality by content)?
- Spot the **aggregate roots** (which entities own which).

### Minute 7–8: Sketch the class diagram
- Entities and their relationships.
- Pick the *right* arrow for each (composition, aggregation, association, inheritance, realization).
- Add multiplicities.

### Minute 9–10: Identify variation points → patterns
- Where will future change happen?
- Each variation point usually maps to a pattern.

After 10 minutes you have: shared scope, the entity model, and high-level pattern choices. The remaining time fills in details — concurrency, persistence, failure modes.

---

## What to say out loud during the interview

The interviewer hears your reasoning, not just your output. Sprinkle these throughout:

- "I'm going to start with these N core operations. Anything else I should support?"
- "X is an entity here because it has identity that persists; Y is a value object."
- "Z is going to vary — payment methods, pricing rules — so I'll pull it behind a Strategy interface."
- "This is shared mutable state; I'll address concurrency once the structure is clear."
- "If we needed persistence, the `XRepository` interface is already in place — we'd swap in a JDBC implementation."
- "I'm using CAS on the spot status rather than a lot-level lock; that lets throughput scale with the number of spots, not the number of gates."
- "I'll inject `Clock` rather than calling `Instant.now()` so time-dependent logic is testable."

---

## Final note

This pack covers the patterns, primitives, and tradeoffs you'll be tested on in an LLD interview. The code is illustrative, not production-ready — interviewers want to see your reasoning and your design instincts, not perfect Java. Read each problem twice: once for the design, once for the dialogue.
