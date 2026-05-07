# Problem 5 — Snake & Ladder

The "warm-up" turn-based game design — gentle but tests fundamentals: state, queue, polymorphism, and the discipline to keep the rules separate from the data.

## Interview opener

> **Interviewer:** Design a Snake & Ladder game.

---

## Clarification phase

> **You:** Standard rules — N×N board, snakes pull you down, ladders pull you up, first to the last cell wins?
>
> **Interviewer:** Yes. The board size and the placement of snakes/ladders should be configurable.

> **You:** Number of players?
>
> **Interviewer:** Configurable, minimum two.

> **You:** Dice — single six-sided, or do I support variants like two dice or weighted dice?
>
> **Interviewer:** Standard six-sided for the core. Mention how you'd add variants.

> **You:** Special rules — does rolling a 6 give an extra turn? Need an exact roll to land on the final cell?
>
> **Interviewer:** Use these rules: roll a 6 = extra turn. Overshooting the final cell — player stays put.

> **You:** Multiple players on the same cell — both stay, or one bumps the other back to start?
>
> **Interviewer:** Both stay. No bumping in this version.

> **You:** Game ends as soon as one player reaches the final cell, or do all players play until everyone finishes?
>
> **Interviewer:** First to reach wins. Stop the game.

> **You:** Single-process, in-memory? No persistence concerns?
>
> **Interviewer:** Yes for the core. We'll discuss persistence at the end.

> **You:** Multiple games concurrently in the same process — like a server hosting many games?
>
> **Interviewer:** Yes — design should support that. Each game is independent.

OK — the design is a turn-based state machine, dice as Strategy, and the board with snakes/ladders modeled as jumps.

---

## Refined requirements

### Functional
1. Create a game with `(boardSize, snakes, ladders, players, dice)`.
2. Play turns: current player rolls, advances, applies any jump (snake or ladder), turn passes (or repeats on 6).
3. Detect game end (a player reaches the last cell exactly) and report the winner.
4. Get current state at any time (positions, current player, finished).

### Non-functional
- Multiple games can run concurrently in one process.
- Per-game operations are serial (turn-based) — no concurrency within a game.

### Out of scope (mentioned)
- Networking / multiplayer over the wire.
- UI rendering.
- Persistence (covered in the final section).

---

## Domain model

### Entities
- **`Game`** — aggregate root.
- **`Player`** — has identity (name) and current position.
- **`Board`** — owned by Game; immutable.

### Value objects
- **`Jump(from, to)`** — a snake or a ladder; immutable. `from > to` ⇒ snake; `from < to` ⇒ ladder.
- **`DiceRoll(value)`** — the result of a roll.

### Strategy interfaces
- **`Dice`** — pluggable dice (single 6-sided, two dice, weighted).
- **`TurnPolicy`** — what happens after a roll: which player goes next, do they get an extra turn?

### Patterns
- **Strategy** — `Dice`, `TurnPolicy`.
- **State** — Game has states `WAITING`, `IN_PROGRESS`, `FINISHED`.
- **Observer** (extension) — UI / logger / replay subscribes to game events.
- **Command + Memento** (extension) — moves as commands for replay or undo.

---

## Class diagram

```
┌───────────────────────────┐
│           Game            │
├───────────────────────────┤
│ -id : String              │
│ -board : Board            │
│ -dice : Dice              │
│ -turnPolicy : TurnPolicy  │
│ -players : List<Player>   │
│ -turnQueue : Deque<Player>│
│ -state : GameState        │
│ -winner : Player          │
├───────────────────────────┤
│ +playTurn() : TurnResult  │
│ +currentPlayer() : Player │
│ +winner() : Optional<...> │
│ +state() : GameState      │
└───────────────────────────┘
       │ ◆          │ ◆          │ ◆
       ▼            ▼            ▼
┌────────────┐ ┌────────────┐ ┌──────────────┐
│   Board    │ │   Player   │ │    Dice      │
├────────────┤ ├────────────┤ │ <<interface>>│
│ -size      │ │ -id        │ ├──────────────┤
│ -jumps     │ │ -name      │ │ +roll():int  │
├────────────┤ │ -position  │ └──────────────┘
│ +size()    │ ├────────────┤        △
│ +jumpFrom()│ │ +moveTo(p) │  ┌─────┴──────┐
└────────────┘ └────────────┘  │ Standard6  │
                               │  Dice      │
                               └────────────┘

         record Jump(int from, int to)         (value object)
         record DiceRoll(int value)            (value object)
         enum GameState { WAITING, IN_PROGRESS, FINISHED }
```

---

## Key abstractions

```java
public interface Dice {
    int roll();
}

public interface TurnPolicy {
    /** Returns true if the current player should roll again (e.g., rolled a 6). */
    boolean grantsExtraTurn(int diceValue);
}
```

---

## Core code

### Value objects + enums

```java
public record Jump(int from, int to) {
    public Jump {
        if (from == to) throw new IllegalArgumentException("from == to");
    }
    public boolean isSnake()  { return from > to; }
    public boolean isLadder() { return from < to; }
}

public enum GameState { WAITING, IN_PROGRESS, FINISHED }
```

### `Player`

```java
public final class Player {
    private final String id;
    private final String name;
    private int position;

    public Player(String id, String name) { this.id = id; this.name = name; this.position = 0; }
    public String id() { return id; }
    public String name() { return name; }
    public int position() { return position; }
    void moveTo(int p) { this.position = p; }
}
```

### `Board`

```java
public final class Board {
    private final int size;                          // last cell number; e.g., 100 for 10x10
    private final Map<Integer, Integer> jumps;       // from -> to

    public Board(int size, List<Jump> jumps) {
        this.size = size;
        Map<Integer, Integer> map = new HashMap<>();
        for (Jump j : jumps) {
            if (j.from() < 1 || j.from() >= size) throw new IllegalArgumentException("Invalid from: " + j.from());
            if (j.to()   < 1 || j.to()   >= size) throw new IllegalArgumentException("Invalid to: " + j.to());
            if (map.put(j.from(), j.to()) != null) throw new IllegalArgumentException("Duplicate from: " + j.from());
        }
        this.jumps = Map.copyOf(map);
    }

    public int size() { return size; }
    public OptionalInt jumpFrom(int cell) {
        Integer to = jumps.get(cell);
        return to == null ? OptionalInt.empty() : OptionalInt.of(to);
    }
}
```

### Dice (Strategy)

```java
public final class StandardDice implements Dice {
    private final Random rng;
    private final int sides;
    public StandardDice() { this(new Random(), 6); }
    public StandardDice(Random rng, int sides) { this.rng = rng; this.sides = sides; }
    @Override public int roll() { return rng.nextInt(sides) + 1; }
}

public final class WeightedDice implements Dice {
    private final Random rng;
    private final double[] cumulative;        // cumulative probabilities
    // ... weighted construction omitted for brevity
    @Override public int roll() { /* sample by cumulative distribution */ return 0; }
}
```

`Random` injected so tests can use a seeded RNG for determinism.

### Turn policy (Strategy)

```java
public final class StandardTurnPolicy implements TurnPolicy {
    @Override public boolean grantsExtraTurn(int diceValue) { return diceValue == 6; }
}

public final class NoExtraTurnPolicy implements TurnPolicy {
    @Override public boolean grantsExtraTurn(int diceValue) { return false; }
}
```

### `Game`

```java
public record TurnResult(
    Player player, int diceValue, int from, int to,
    OptionalInt jumpedTo, boolean extraTurn, boolean wonGame
) { }

public final class Game {
    private final String id;
    private final Board board;
    private final Dice dice;
    private final TurnPolicy turnPolicy;
    private final Deque<Player> turnQueue;
    private GameState state;
    private Player winner;

    public Game(String id, Board board, Dice dice, TurnPolicy policy, List<Player> players) {
        if (players.size() < 2) throw new IllegalArgumentException("Need at least 2 players");
        this.id = id;
        this.board = board;
        this.dice = dice;
        this.turnPolicy = policy;
        this.turnQueue = new ArrayDeque<>(players);
        this.state = GameState.IN_PROGRESS;
    }

    public synchronized TurnResult playTurn() {
        if (state != GameState.IN_PROGRESS) throw new IllegalStateException("Game " + state);

        Player current = turnQueue.peekFirst();
        int diceValue = dice.roll();
        int from = current.position();
        int target = from + diceValue;
        OptionalInt jumpedTo = OptionalInt.empty();
        int finalPos = from;

        if (target > board.size()) {
            // Overshoot — stay put
            finalPos = from;
        } else {
            finalPos = target;
            OptionalInt jump = board.jumpFrom(finalPos);
            if (jump.isPresent()) {
                jumpedTo = jump;
                finalPos = jump.getAsInt();
            }
        }

        current.moveTo(finalPos);
        boolean wonGame = (finalPos == board.size());
        boolean extraTurn = turnPolicy.grantsExtraTurn(diceValue) && !wonGame;

        if (wonGame) {
            state = GameState.FINISHED;
            winner = current;
        } else if (!extraTurn) {
            // rotate: head to tail
            turnQueue.pollFirst();
            turnQueue.addLast(current);
        }
        // If extraTurn, leave queue unchanged so current player rolls again next call.

        return new TurnResult(current, diceValue, from, target, jumpedTo, extraTurn, wonGame);
    }

    public synchronized Player currentPlayer() {
        return turnQueue.peekFirst();
    }

    public synchronized GameState state() { return state; }
    public synchronized Optional<Player> winner() { return Optional.ofNullable(winner); }
    public String id() { return id; }
}
```

### Putting it together

```java
List<Jump> jumps = List.of(
    new Jump(2, 38),    // ladder
    new Jump(7, 14),    // ladder
    new Jump(43, 17),   // snake
    new Jump(62, 19),   // snake
    new Jump(98, 79)    // snake
);
Board board = new Board(100, jumps);

List<Player> players = List.of(
    new Player("p1", "Alice"),
    new Player("p2", "Bob")
);

Game game = new Game(
    "game-1",
    board,
    new StandardDice(),
    new StandardTurnPolicy(),
    players
);

while (game.state() == GameState.IN_PROGRESS) {
    TurnResult r = game.playTurn();
    System.out.printf("%s rolled %d: %d -> %d%s%s%n",
        r.player().name(), r.diceValue(), r.from(), r.to(),
        r.jumpedTo().isPresent() ? " (jumped to " + r.jumpedTo().getAsInt() + ")" : "",
        r.wonGame() ? " — WINS!" : "");
}
```

---

## Concurrency strategy

Each `Game` is its own state machine. Within a game, turns are serial — `playTurn()` is the only mutating method, and it's `synchronized`. Multiple games run independently in different `Game` instances.

If you wanted to run many games in a thread pool: each game's `playTurn` is serial, so submit each game's turns to the pool but ensure no two turns of the *same* game overlap. A per-game executor (single-thread) accomplishes this.

`Player.position` is mutated only inside the synchronized `Game` method, so no extra synchronization needed — visibility comes from the surrounding monitor.

---

## Edge cases / gotchas

| Case | Handling |
|---|---|
| **Player rolls past the final cell** | Stay put (specified rule). |
| **Player lands exactly on snake/ladder head** | Apply the jump (single hop — don't chain jumps even if `to` is also a jump source, since boards usually don't have chains; we throw on construction if they do). |
| **Multiple players on same cell** | Both stay (specified). |
| **Player lands on the final cell exactly** | Win. |
| **Rolling a 6 wins** | Win first, no extra turn. |
| **Game already finished** | `playTurn` throws. |
| **Less than 2 players at construction** | Throw. |
| **Jump with `from == to`** | Throw at construction. |
| **Cycle in jumps (chains)** | Throw at construction (we forbid by checking that no `to` is also a `from` if you want stricter validation). |
| **Empty board (no jumps)** | Allowed — pure dice game. |

---

## Extensions / interview follow-ups

### "Add multiple dice."
- `Dice` is already an interface. `TwoDice implements Dice` returns sum of two `Random.nextInt(6) + 1`.
- Special rules (e.g., doubles give an extra turn) become a new `TurnPolicy`.

### "Bumping rule (knock other player back to start when you land on their cell)."
- Game checks `for (Player p : players) if (p != current && p.position() == finalPos) p.moveTo(0);`.
- Variant rule: only knock back to a safe cell; or add a separate `CellOccupancyPolicy` Strategy.

### "Tournament mode — best of N games."
- Wrapper `Tournament` that runs N games, tracks wins, declares overall winner.
- Each game is an independent `Game` instance.

### "Replay a game / save game state."
- **Memento**: `GameSnapshot` capturing positions, turn queue, RNG seed.
- **Command**: each `playTurn` is recorded; replay = re-execute commands with same seed.
- Pair both — Memento for fast restore, Commands for human-readable history.

### "Add an Observer for UI / logging."
- `GameListener` interface with `onTurn(TurnResult)`, `onGameEnd(Player winner)`.
- `Game` maintains `List<GameListener>`; calls them after each `playTurn`.
- UI subscribes; rendering happens on observer callback.

### "AI player (computer opponent)."
- `Player` becomes abstract: `HumanPlayer` (waits for input), `AIPlayer` (returns immediately).
- For Snake & Ladder there's no strategy (rolls are random) — this is more relevant for Chess. Mention that.

### "Networked multiplayer."
- Game lives server-side. Each turn becomes a request.
- Server validates turn (was it actually this player's turn? has the game ended?).
- Push updates to all clients via WebSocket / SSE — the **Observer** model applied across the network.

---

## Scaling

For a single game running locally, the design is over-engineered. For a server hosting many games:

### Many concurrent games
- Each `Game` is independent — a `ConcurrentHashMap<String, Game>` indexed by gameId.
- Each turn request looks up its game and calls `playTurn`.
- Per-game synchronization (already present); no global lock.

### Persistent games (resume across server restart)
- Persist game state on each turn (see Persistence section).

### Networked play
- Server-side game; clients submit turns; server broadcasts results.
- WebSocket per client connection; events broadcast on game state change.

### Anti-cheat / fairness
- Server-side dice (clients can't roll their own — would be trivially cheatable).
- Audit log of every roll + state transition.

### Spectator mode
- Read-only subscribers to a game's event stream.

---

## Persistence tradeoffs

For a game that runs for a few minutes and is gone, in-memory is fine. The moment you want any of {pause-and-resume, leaderboards, replays, server upgrades without losing games}, persistence enters.

### What persistent data exists
- **Active games**: state at end of each turn (positions, current player, RNG state, turn count).
- **Completed games**: results (winner, duration, players, final board snapshot).
- **Replays**: ordered list of `TurnResult` per game.
- **Player profiles**: identity, stats, history.

### Option 1 — Relational (PostgreSQL / MySQL)
Schema sketch:
```sql
games(id, board_id, state, current_player_idx, started_at, ended_at, winner_player_id, version)
game_players(game_id, player_id, position, seat_idx, PRIMARY KEY (game_id, seat_idx))
turns(game_id, turn_no, player_id, dice_value, from_cell, to_cell, jumped_to, ts,
       PRIMARY KEY (game_id, turn_no))
boards(id, size, jumps_json)
players(id, name, ...)
```
- **Pros**: ACID — turn write + position update + state update in one transaction. Easy to query "all games Alice played." Replays are a `SELECT * FROM turns WHERE game_id = ? ORDER BY turn_no`.
- **Cons**: schema migration overhead. Read-heavy workloads (leaderboards) need indexes carefully placed. The `version` column gives optimistic locking against concurrent turn writes (shouldn't happen normally, but client retries can cause it).
- **When to pick**: most cases. The data is naturally relational, and ACID matches the "one turn at a time" semantics.

### Option 2 — Key-value (Redis)
- Store game state as a JSON blob keyed by `game:{id}`.
- Each turn: `WATCH game:{id}; GET; modify; MULTI; SET; EXEC` (Redis-native optimistic locking).
- Replays: separate sorted set `turns:{id}`, score = turn number.
- TTL on active games (`EXPIRE game:{id} 3600`) so abandoned games disappear.
- **Pros**: extremely fast; the read-modify-write fits Redis transactions naturally. Leaderboards via `ZADD` / `ZRANGE`.
- **Cons**: durability tradeoff (RDB snapshot vs AOF — AOF is safer). Limited query power (no "find all of Alice's games" without secondary indexes you maintain by hand). Replays grow without natural archival.
- **When to pick**: short-lived games at high concurrency (like an online game lobby). Pair with SQL for long-term storage of completed games.

### Option 3 — Document store (MongoDB / DynamoDB)
- Store the entire game (state + turn history) as one document.
- **Pros**: schema flexibility; one round-trip per turn. Replays embedded with the game.
- **Cons**: document size grows with turn count (Mongo has a 16MB limit). DynamoDB has per-item size limits too. For long games this can hurt — better to split turn history into a separate collection.
- **Consistency**: DynamoDB has strong-read but writes are eventually consistent across replicas; for game turns, use a transactional write or conditional update with a `version` attribute.
- **When to pick**: schema is evolving rapidly, or game data has irregular fields per game variant.

### Option 4 — Event sourcing (append-only log)
- Persist only `TurnResult` events. Game state is the fold over events.
- Storage: Kafka, EventStoreDB, or just an SQL `events` table.
- On startup: replay events to reconstruct in-memory state. For long games, snapshot every N turns.
- **Pros**: complete history for free. Replay is trivial. Time-travel debugging.
- **Cons**: complexity. Reads are slower (must fold or hit a snapshot). Schema migrations are hard (events are immutable; you must support reading old shapes).
- **When to pick**: when audit / replay / time-travel is a hard requirement (more applicable to gambling, finance — overkill for casual games).

### Option 5 — In-memory + periodic checkpoint
- Game lives in memory; snapshot to durable storage every N turns or on graceful shutdown.
- Crash recovery: load latest snapshot, possibly lose a few turns.
- **Pros**: lowest latency on the hot path. Low DB write cost.
- **Cons**: data loss on crash. Reconnect logic for clients in flight.
- **When to pick**: casual games where losing a few turns is acceptable; game state is small enough to checkpoint cheaply.

### Recommendation for Snake & Ladder
- **Hot store**: Redis for active games (state-as-JSON, 1-hour TTL).
- **Cold store**: PostgreSQL for completed games (final state, winner, duration; player stats).
- **Replays**: keep in Redis for ~24 hours, then archive to S3 + a `replays(game_id, blob_key)` row in PostgreSQL.

This split is the canonical "fast hot path + durable cold path" pattern.

### Recommendations across DB choices

| Concern | Best fit |
|---|---|
| **ACID per turn** | SQL |
| **High concurrency, short-lived games** | Redis |
| **Long-running games with growing history** | SQL or MongoDB with separate turns collection |
| **Replay / time-travel** | Event sourcing |
| **Player leaderboards** | Redis sorted sets, or SQL with materialized views |
| **Cross-game analytics** | SQL warehouse / BigQuery |
| **Real-time spectator updates** | Redis pub/sub or Kafka |

### Concurrency under persistence

In-memory had `synchronized` per game. With persistence:
- **SQL**: optimistic locking via `version` column. Each turn does `UPDATE games SET ... WHERE id = ? AND version = ?` and retries on 0-rows-affected.
- **Redis**: `WATCH` / `MULTI` / `EXEC` for atomic compare-and-swap.
- **Document store**: conditional update with a version attribute (DynamoDB `--condition-expression`).

The pattern is the same: **read with version → modify → write conditional on version → retry if lost**.

---

## Talking points for the interview

- "Dice is a Strategy because variants (two dice, weighted) come up; injecting `Random` makes turns deterministic in tests."
- "I model snakes and ladders as a single `Jump` value object — the direction is implicit in `from` vs `to`. Halves the conditional logic."
- "Turn rotation is a `Deque` rather than an index — easier to handle eliminated players or skip turns."
- "Per-game `synchronized` is enough — games are independent. No global lock."
- "I keep the `Game` ignorant of UI; if we need rendering or remote updates, that's an Observer subscribed to game events."
- "If we needed to persist, the `TurnResult` record is already the natural event; an event-sourced design slots in without changing the core."

---

## Summary

Patterns load-bearing: **Strategy** (Dice, TurnPolicy), **State** (game lifecycle).

The mental model: a turn-based loop wrapping a state machine. Roll → move → resolve jump → check win → rotate (or grant extra turn). Each piece is small and pluggable.

Snake & Ladder is the gentlest of the game-design problems — a good warm-up for **Chess**, where the same skeleton scales up dramatically (per-piece move validation, check detection, undo).
