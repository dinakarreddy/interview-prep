# Problem 6 — Chess

The "everything-in-one" LLD problem. Tests polymorphism (per-piece rules), state machines (turn, check, checkmate), command pattern (move history, undo), and your taste for keeping rules out of data.

## Interview opener

> **Interviewer:** Design a chess game.

---

## Clarification phase

> **You:** Standard 8×8 chess, two players, full rules?
>
> **Interviewer:** Yes. Don't worry about rendering or UI — just the rules engine and game state.

> **You:** Should I implement all the special moves? Castling, en passant, pawn promotion?
>
> **Interviewer:** Yes — at least castling and pawn promotion. Mention en passant; you can stub it if time runs out.

> **You:** Check, checkmate, stalemate detection?
>
> **Interviewer:** All three.

> **You:** Draw conditions — fifty-move rule, threefold repetition, insufficient material?
>
> **Interviewer:** Skip those for the core. Mention how you'd add them.

> **You:** Move history — should the game support undo / redo?
>
> **Interviewer:** Yes — at least undo. That's the natural fit for the Command pattern, and chess is a good test of it.

> **You:** Move input format — algebraic notation ("e4"), or `(from, to)` coordinate pairs?
>
> **Interviewer:** Coordinate pairs are fine. Don't waste time on parser.

> **You:** Two players locally, or networked, or AI?
>
> **Interviewer:** Two players locally for the rules engine. Mention what changes for networked play.

> **You:** Time controls (chess clocks)?
>
> **Interviewer:** Out of scope.

> **You:** Persistence — save/load games?
>
> **Interviewer:** Discuss it at the end.

OK, design needs: 8×8 board, six piece types with their move rules, two players, turn management, check/checkmate/stalemate, special moves (castling, promotion, en passant), move history with undo.

---

## Refined requirements

### Functional
1. Initialize a fresh game with the standard starting position.
2. Make a move: `move(from, to)` (with optional `promotionPiece` for pawn promotion).
3. Validate moves — only legal moves accepted, including: piece-specific rules, blocked paths, can't put own king in check, special moves' preconditions.
4. Detect check, checkmate, stalemate after each move.
5. Track move history; support `undo()` to restore previous state.
6. Query state: whose turn, current board, last move, status.

### Non-functional
- Turn-based; no concurrency *within* a game.
- Multiple games supported (separate `Game` instances).
- Validation must be exhaustive — illegal moves rejected with clear errors.

### Out of scope (mentioned)
- Draw by 50-move rule, threefold repetition, insufficient material.
- Time controls.
- Networked play, UI rendering.
- AI / move suggestions.

---

## Domain model

### Entities
- **`Game`** — aggregate root.
- **`Board`** — owned by Game; mutable.
- **`Player`** — has color + identity.

### Value objects
- **`Square(file, rank)`** — coordinate; immutable. (Files a–h, ranks 1–8.)
- **`Move(from, to, piece, captured, promotion?, isCastling?, isEnPassant?, ...)`** — recorded for history.

### Pieces (one entity per type, sharing an interface)
- **`Piece`** — abstract: `color`, has-moved flag (for castling/pawn first move).
- `King`, `Queen`, `Rook`, `Bishop`, `Knight`, `Pawn` — each implements its move logic.

### Enums
- `Color` — `WHITE`, `BLACK`.
- `PieceType` — `KING`, `QUEEN`, `ROOK`, `BISHOP`, `KNIGHT`, `PAWN`.
- `GameState` — `WHITE_TURN`, `BLACK_TURN`, `CHECK`, `CHECKMATE`, `STALEMATE`.

### Patterns
- **Strategy** — each piece's move generation is its own logic; `Piece` is the strategy interface.
- **State** — game lifecycle (turn → check → checkmate / stalemate).
- **Command** — `Move` is a command; **Memento** for undo state capture.
- **Composite** — board contains squares contain pieces (loose composite).
- **Observer** — UI / logger subscribes to game events.

---

## Class diagram

```
┌──────────────────────────────────────┐
│              Game                    │
├──────────────────────────────────────┤
│ -board : Board                       │
│ -players : Map<Color, Player>        │
│ -currentColor : Color                │
│ -state : GameState                   │
│ -history : Deque<Move>               │
│ -mementos : Deque<GameMemento>       │
├──────────────────────────────────────┤
│ +makeMove(from, to, promo?) : Move   │
│ +undo() : void                       │
│ +legalMoves(Square) : List<Move>     │
│ +state() : GameState                 │
└──────────────────────────────────────┘
       │                  │
       ◆ 1                ◆ 1
       ▼                  ▼
┌──────────────┐   ┌─────────────────────┐
│   Player     │   │       Board         │
└──────────────┘   ├─────────────────────┤
                   │ -squares: Piece[8][8]│
                   ├─────────────────────┤
                   │ +get(Square)        │
                   │ +set(Square, Piece) │
                   │ +findKing(Color)    │
                   │ +inBounds(Square)   │
                   │ +copy() : Board     │
                   └─────────────────────┘
                            │ ◇ holds (sparse)
                            ▼
                   ┌──────────────────────┐
                   │  <<abstract>> Piece  │
                   ├──────────────────────┤
                   │ -color : Color       │
                   │ -hasMoved : boolean  │
                   ├──────────────────────┤
                   │ +pseudoLegalMoves    │
                   │   (board, from)      │
                   │   : List<Move>       │
                   │ +type() : PieceType  │
                   └──────────────────────┘
                            △
        ┌────────┬──────────┼──────────┬────────┬────────┐
     ┌──────┐┌──────┐┌──────┐ ┌──────┐┌──────┐┌──────┐
     │ King ││Queen ││ Rook │ │Bishop││Knight││ Pawn │
     └──────┘└──────┘└──────┘ └──────┘└──────┘└──────┘

     record Square(int file, int rank)
     record Move(from, to, piece, captured, promotion, kind)
```

---

## Key abstractions

```java
public abstract class Piece {
    protected final Color color;
    protected boolean hasMoved;

    protected Piece(Color color) { this.color = color; }

    public Color color()              { return color; }
    public boolean hasMoved()         { return hasMoved; }
    public void markMoved()           { this.hasMoved = true; }
    public abstract PieceType type();

    /**
     * Pseudo-legal moves — follow piece rules; do not check whether
     * the resulting position would expose own king to check. Game
     * filters these to true legal moves.
     */
    public abstract List<Move> pseudoLegalMoves(Board board, Square from);
}
```

The "pseudo-legal" / "legal" split is the classic chess-engine separation. Each piece *knows its movement geometry*. The Game *filters out moves that would leave its own king in check.*

---

## Core code

### Value objects + enums

```java
public enum Color  { WHITE, BLACK; public Color opposite() { return this == WHITE ? BLACK : WHITE; } }
public enum PieceType { KING, QUEEN, ROOK, BISHOP, KNIGHT, PAWN }
public enum GameState { WHITE_TURN, BLACK_TURN, CHECK, CHECKMATE, STALEMATE }

public record Square(int file, int rank) {
    public Square {
        if (file < 0 || file > 7 || rank < 0 || rank > 7)
            throw new IllegalArgumentException("Out of bounds: " + file + "," + rank);
    }
    public Square offset(int df, int dr) { return new Square(file + df, rank + dr); }
    public static boolean isInBounds(int f, int r) { return f >= 0 && f <= 7 && r >= 0 && r <= 7; }
}

public enum MoveKind { NORMAL, CASTLE_KINGSIDE, CASTLE_QUEENSIDE, EN_PASSANT, PROMOTION }

public record Move(
    Square from, Square to,
    Piece piece, Piece captured,
    PieceType promotion,        // nullable
    MoveKind kind
) { }
```

### `Board`

```java
public final class Board {
    private final Piece[][] squares = new Piece[8][8];

    public Piece get(Square s)            { return squares[s.file()][s.rank()]; }
    public void set(Square s, Piece p)    { squares[s.file()][s.rank()] = p; }
    public boolean isEmpty(Square s)      { return get(s) == null; }

    public Square findKing(Color color) {
        for (int f = 0; f < 8; f++)
            for (int r = 0; r < 8; r++) {
                Piece p = squares[f][r];
                if (p != null && p.type() == PieceType.KING && p.color() == color)
                    return new Square(f, r);
            }
        throw new IllegalStateException("No king found for " + color);
    }

    public Stream<Square> piecesOf(Color color) {
        Stream.Builder<Square> b = Stream.builder();
        for (int f = 0; f < 8; f++)
            for (int r = 0; r < 8; r++) {
                Piece p = squares[f][r];
                if (p != null && p.color() == color) b.add(new Square(f, r));
            }
        return b.build();
    }
}
```

For undo, `Board.copy()` would deep-copy the array. We'll use `Memento` instead — capture only what changes.

### Pieces (Strategy per type)

```java
public final class King extends Piece {
    public King(Color c) { super(c); }
    public PieceType type() { return PieceType.KING; }
    public List<Move> pseudoLegalMoves(Board board, Square from) {
        var moves = new ArrayList<Move>();
        int[][] deltas = { {1,0},{-1,0},{0,1},{0,-1},{1,1},{1,-1},{-1,1},{-1,-1} };
        for (int[] d : deltas) {
            int f = from.file() + d[0], r = from.rank() + d[1];
            if (!Square.isInBounds(f, r)) continue;
            Square to = new Square(f, r);
            Piece occupant = board.get(to);
            if (occupant == null) moves.add(new Move(from, to, this, null, null, MoveKind.NORMAL));
            else if (occupant.color() != color) moves.add(new Move(from, to, this, occupant, null, MoveKind.NORMAL));
        }
        // Castling — added by Game (needs full game state to validate path/no-check)
        return moves;
    }
}

public final class Knight extends Piece {
    public Knight(Color c) { super(c); }
    public PieceType type() { return PieceType.KNIGHT; }
    public List<Move> pseudoLegalMoves(Board board, Square from) {
        var moves = new ArrayList<Move>();
        int[][] deltas = { {1,2},{2,1},{-1,2},{-2,1},{1,-2},{2,-1},{-1,-2},{-2,-1} };
        for (int[] d : deltas) {
            int f = from.file() + d[0], r = from.rank() + d[1];
            if (!Square.isInBounds(f, r)) continue;
            Square to = new Square(f, r);
            Piece occupant = board.get(to);
            if (occupant == null || occupant.color() != color)
                moves.add(new Move(from, to, this, occupant, null, MoveKind.NORMAL));
        }
        return moves;
    }
}

public final class Rook extends Piece {
    public Rook(Color c) { super(c); }
    public PieceType type() { return PieceType.ROOK; }
    public List<Move> pseudoLegalMoves(Board board, Square from) {
        return slidingMoves(board, from, new int[][] {{1,0},{-1,0},{0,1},{0,-1}});
    }
    private List<Move> slidingMoves(Board board, Square from, int[][] dirs) {
        var moves = new ArrayList<Move>();
        for (int[] d : dirs) {
            int f = from.file(), r = from.rank();
            while (true) {
                f += d[0]; r += d[1];
                if (!Square.isInBounds(f, r)) break;
                Square to = new Square(f, r);
                Piece occ = board.get(to);
                if (occ == null) moves.add(new Move(from, to, this, null, null, MoveKind.NORMAL));
                else { if (occ.color() != color) moves.add(new Move(from, to, this, occ, null, MoveKind.NORMAL)); break; }
            }
        }
        return moves;
    }
}

public final class Bishop extends Piece {
    public Bishop(Color c) { super(c); }
    public PieceType type() { return PieceType.BISHOP; }
    public List<Move> pseudoLegalMoves(Board board, Square from) {
        // Reuse the sliding helper. In real code, factor it into an abstract base.
        return new SlidingHelper().slide(board, from, this, new int[][] {{1,1},{1,-1},{-1,1},{-1,-1}});
    }
}

public final class Queen extends Piece {
    public Queen(Color c) { super(c); }
    public PieceType type() { return PieceType.QUEEN; }
    public List<Move> pseudoLegalMoves(Board board, Square from) {
        return new SlidingHelper().slide(board, from, this, new int[][] {
            {1,0},{-1,0},{0,1},{0,-1},{1,1},{1,-1},{-1,1},{-1,-1}
        });
    }
}

public final class Pawn extends Piece {
    public Pawn(Color c) { super(c); }
    public PieceType type() { return PieceType.PAWN; }
    public List<Move> pseudoLegalMoves(Board board, Square from) {
        var moves = new ArrayList<Move>();
        int dir = color == Color.WHITE ? 1 : -1;
        int promotionRank = color == Color.WHITE ? 7 : 0;
        int startRank     = color == Color.WHITE ? 1 : 6;

        // Single-step
        Square ahead = new Square(from.file(), from.rank() + dir);
        if (Square.isInBounds(ahead.file(), ahead.rank()) && board.isEmpty(ahead)) {
            addPawnMove(moves, from, ahead, null, promotionRank);
            // Double-step from start rank
            if (from.rank() == startRank) {
                Square ahead2 = new Square(from.file(), from.rank() + 2*dir);
                if (board.isEmpty(ahead2)) moves.add(new Move(from, ahead2, this, null, null, MoveKind.NORMAL));
            }
        }
        // Captures
        for (int df : new int[] {-1, 1}) {
            int f = from.file() + df, r = from.rank() + dir;
            if (!Square.isInBounds(f, r)) continue;
            Square to = new Square(f, r);
            Piece occ = board.get(to);
            if (occ != null && occ.color() != color) addPawnMove(moves, from, to, occ, promotionRank);
        }
        // En passant — needs game-level info (last move); Game adds these.
        return moves;
    }

    private void addPawnMove(List<Move> moves, Square from, Square to, Piece captured, int promotionRank) {
        if (to.rank() == promotionRank) {
            // Generate promotion to all four piece types
            for (PieceType pt : List.of(PieceType.QUEEN, PieceType.ROOK, PieceType.BISHOP, PieceType.KNIGHT))
                moves.add(new Move(from, to, this, captured, pt, MoveKind.PROMOTION));
        } else {
            moves.add(new Move(from, to, this, captured, null, MoveKind.NORMAL));
        }
    }
}
```

### `Game` — the orchestrator (rules engine)

```java
public final class Game {
    private final Board board = new Board();
    private final Deque<Move> history = new ArrayDeque<>();
    private final Deque<GameMemento> mementos = new ArrayDeque<>();
    private Color currentColor = Color.WHITE;
    private GameState state = GameState.WHITE_TURN;

    public Game() { setupInitialPosition(); }

    public synchronized Move makeMove(Square from, Square to, PieceType promotion) {
        if (state == GameState.CHECKMATE || state == GameState.STALEMATE)
            throw new IllegalStateException("Game is over: " + state);

        Piece piece = board.get(from);
        if (piece == null) throw new IllegalArgumentException("Empty square: " + from);
        if (piece.color() != currentColor) throw new IllegalArgumentException("Not your piece");

        Move chosen = legalMovesFor(from).stream()
            .filter(m -> m.to().equals(to) && (promotion == null || m.promotion() == promotion))
            .findFirst()
            .orElseThrow(() -> new IllegalArgumentException("Illegal move: " + from + " → " + to));

        // Memento for undo
        mementos.push(snapshot());

        // Apply
        applyMove(chosen);
        history.push(chosen);

        // Switch turns and detect end conditions
        currentColor = currentColor.opposite();
        recomputeState();

        return chosen;
    }

    public synchronized void undo() {
        if (mementos.isEmpty()) throw new IllegalStateException("Nothing to undo");
        GameMemento m = mementos.pop();
        history.pop();
        restore(m);
    }

    public synchronized List<Move> legalMovesFor(Square from) {
        Piece p = board.get(from);
        if (p == null || p.color() != currentColor) return List.of();
        var pseudo = new ArrayList<>(p.pseudoLegalMoves(board, from));
        // Add castling moves (King) — needs game-level state
        if (p.type() == PieceType.KING) addCastlingMoves(pseudo, from, p.color());
        // Add en passant (Pawn) — needs last-move info
        if (p.type() == PieceType.PAWN)  addEnPassantMoves(pseudo, from, p.color());
        // Filter: drop moves that would leave own king in check
        var legal = new ArrayList<Move>();
        for (Move m : pseudo) {
            GameMemento snap = snapshot();
            applyMove(m);
            if (!isKingInCheck(p.color())) legal.add(m);
            restore(snap);
        }
        return legal;
    }

    private boolean isKingInCheck(Color color) {
        Square kingSquare = board.findKing(color);
        // Generate all opponent pseudo-legal moves; if any targets kingSquare, in check.
        return board.piecesOf(color.opposite())
            .flatMap(sq -> board.get(sq).pseudoLegalMoves(board, sq).stream())
            .anyMatch(m -> m.to().equals(kingSquare));
    }

    private void recomputeState() {
        boolean inCheck = isKingInCheck(currentColor);
        boolean hasLegal = board.piecesOf(currentColor)
            .flatMap(sq -> legalMovesFor(sq).stream())
            .findAny().isPresent();

        if (inCheck && !hasLegal)        state = GameState.CHECKMATE;
        else if (!inCheck && !hasLegal)  state = GameState.STALEMATE;
        else if (inCheck)                state = GameState.CHECK;
        else state = (currentColor == Color.WHITE) ? GameState.WHITE_TURN : GameState.BLACK_TURN;
    }

    private void applyMove(Move m) {
        // Standard move
        board.set(m.from(), null);
        Piece moving = m.piece();
        moving.markMoved();

        if (m.kind() == MoveKind.PROMOTION) {
            board.set(m.to(), createPiece(m.promotion(), moving.color()));
        } else if (m.kind() == MoveKind.CASTLE_KINGSIDE) {
            board.set(m.to(), moving);
            // Move rook from h-file to f-file
            int rank = m.from().rank();
            Piece rook = board.get(new Square(7, rank));
            board.set(new Square(7, rank), null);
            board.set(new Square(5, rank), rook);
            rook.markMoved();
        } else if (m.kind() == MoveKind.CASTLE_QUEENSIDE) {
            board.set(m.to(), moving);
            int rank = m.from().rank();
            Piece rook = board.get(new Square(0, rank));
            board.set(new Square(0, rank), null);
            board.set(new Square(3, rank), rook);
            rook.markMoved();
        } else if (m.kind() == MoveKind.EN_PASSANT) {
            board.set(m.to(), moving);
            int capturedRank = (moving.color() == Color.WHITE) ? m.to().rank() - 1 : m.to().rank() + 1;
            board.set(new Square(m.to().file(), capturedRank), null);
        } else {
            board.set(m.to(), moving);
        }
    }

    // Castling: requires king & rook unmoved, path clear, king not in check on any traversed square
    private void addCastlingMoves(List<Move> moves, Square from, Color color) {
        Piece king = board.get(from);
        if (king.hasMoved() || isKingInCheck(color)) return;
        int rank = from.rank();
        // Kingside
        Piece rook = board.get(new Square(7, rank));
        if (rook instanceof Rook r && r.color() == color && !r.hasMoved()
            && board.isEmpty(new Square(5, rank)) && board.isEmpty(new Square(6, rank))
            && !squareUnderAttack(new Square(5, rank), color.opposite())
            && !squareUnderAttack(new Square(6, rank), color.opposite())) {
            moves.add(new Move(from, new Square(6, rank), king, null, null, MoveKind.CASTLE_KINGSIDE));
        }
        // Queenside — symmetric, omitted here
    }

    private boolean squareUnderAttack(Square sq, Color attacker) {
        return board.piecesOf(attacker)
            .flatMap(s -> board.get(s).pseudoLegalMoves(board, s).stream())
            .anyMatch(m -> m.to().equals(sq));
    }

    private void addEnPassantMoves(List<Move> moves, Square from, Color color) {
        if (history.isEmpty()) return;
        Move last = history.peek();
        if (last.piece().type() != PieceType.PAWN) return;
        // Last move was a 2-square pawn advance landing next to us
        if (Math.abs(last.from().rank() - last.to().rank()) != 2) return;
        if (last.to().rank() != from.rank()) return;
        if (Math.abs(last.to().file() - from.file()) != 1) return;
        int dir = color == Color.WHITE ? 1 : -1;
        Square to = new Square(last.to().file(), from.rank() + dir);
        moves.add(new Move(from, to, board.get(from), board.get(last.to()), null, MoveKind.EN_PASSANT));
    }

    private GameMemento snapshot() { return GameMemento.from(board, currentColor, state); }
    private void restore(GameMemento m) { m.applyTo(board); currentColor = m.currentColor(); state = m.state(); }

    private void setupInitialPosition() { /* place all 32 pieces */ }
    private Piece createPiece(PieceType t, Color c) {
        return switch (t) {
            case KING   -> new King(c);
            case QUEEN  -> new Queen(c);
            case ROOK   -> new Rook(c);
            case BISHOP -> new Bishop(c);
            case KNIGHT -> new Knight(c);
            case PAWN   -> new Pawn(c);
        };
    }

    public synchronized GameState state()             { return state; }
    public synchronized Optional<Move> lastMove()     { return Optional.ofNullable(history.peek()); }
}
```

### `GameMemento` — simple full-board snapshot (clear + correct)

```java
public record GameMemento(Piece[][] squares, Color currentColor, GameState state) {
    public static GameMemento from(Board b, Color c, GameState s) {
        Piece[][] copy = new Piece[8][8];
        for (int f = 0; f < 8; f++) for (int r = 0; r < 8; r++) copy[f][r] = b.get(new Square(f, r));
        return new GameMemento(copy, c, s);
    }
    public void applyTo(Board b) {
        for (int f = 0; f < 8; f++) for (int r = 0; r < 8; r++) b.set(new Square(f, r), squares[f][r]);
    }
}
```

For a real engine, store **only the diff** (the move + captured piece + has-moved bits) to save memory in long games. The interview-friendly version above prioritizes clarity.

---

## Concurrency strategy

A single chess game is turn-based — no concurrency within. `Game.makeMove` is `synchronized` to handle multi-input scenarios (web UI submitting moves, AI thinking on a worker thread). Multiple games run independently in different `Game` instances.

For a server hosting many games:
- `ConcurrentHashMap<String, Game>` indexed by gameId.
- Each turn request looks up its game, calls `makeMove`.
- No per-game lock contention since each game has its own monitor.

For AI moves: the AI runs on a worker thread, computes a move, then calls `makeMove` — the synchronization is the same as a human move.

---

## Edge cases / gotchas

| Case | Handling |
|---|---|
| **Move that exposes own king** | Filtered in `legalMovesFor` — the move is rejected. |
| **Castling through check** | All squares the king passes through must be unattacked. |
| **Pawn promotion without specifying piece** | Throw — caller must specify. Default to Queen if interface allows. |
| **En passant timing** | Only the move *immediately* after the opponent's two-square pawn advance. Captured pawn is on the same rank as the moving pawn, not on the destination. |
| **Stalemate vs checkmate** | Same "no legal moves" condition; differ in whether king is in check. |
| **Self-pin** | A pinned piece can still move *along* the pin line (e.g., pinned rook can still slide on the file). Handled by the `legalMovesFor` filter. |
| **Two kings adjacent** | Forbidden — would put the moving king in check. Handled by check filter. |
| **Game over: still trying to move** | `makeMove` throws when state is CHECKMATE or STALEMATE. |
| **Board ASCII orientation** | Watch your file/rank indexing — the most common bug source. Pick a convention and document. |

---

## Extensions / interview follow-ups

### "Add the 50-move rule and threefold repetition."
- Track `halfMoveClock` — increments each move, resets on capture or pawn move. ≥ 100 → draw.
- Track positions hashed (Zobrist hashing) into a map; if any position repeats 3 times → draw.

### "Add Algebraic notation parsing/output."
- Move → string: piece letter + disambiguator + capture flag + destination + promotion + check/mate.
- String → move: requires legal-move generation to disambiguate (`Nbd2` vs `Nfd2`).

### "Add an AI player."
- `Player` becomes an interface; `HumanPlayer` waits for input, `AIPlayer` runs minimax (or alpha-beta or MCTS).
- Game doesn't change — it just calls `player.chooseMove(game)`.

### "Networked multiplayer."
- Game lives server-side. Each move becomes a request.
- Server validates (`isPlayer's turn? game still running?`).
- Push updates to both clients via WebSocket — Observer applied across the network.

### "Why aren't piece classes responsible for check detection?"
- Check is a *board-level* property — a piece doesn't know whether moving exposes its king.
- Piece responsibility = "where can I geometrically go?" Game responsibility = "of those, which are legal?"
- This separation lets you reuse pseudo-legal generation for AI search without check overhead.

### "Why a Memento for undo, not inverse moves?"
- Inverse moves are tricky — castling restores rook position too; promotion restores the pawn type; en passant restores a pawn the captured piece was actually beside, not on.
- Memento captures the whole state — clear, correct, slightly more memory.
- Optimization: Memento stores only the changed squares + has-moved flags + state.

### "How do you handle concurrent move submissions in a networked game?"
- Server-side `Game` is `synchronized`. Two simultaneous requests serialize.
- The second one will fail validation (`Not your turn`) once the first has been applied.

---

## Scaling

A single chess game is a small object. Scaling concerns appear when you host many games:

### Fleet of games (chess.com / lichess scale)
- Stateless game servers, each holding an in-memory `Game` keyed by gameId.
- Sharding by gameId hash so each game lives on one server.
- On server crash: restore from persistence (see below).

### Live broadcasting
- Each move publishes to a topic (`game:{id}:moves`).
- Subscribers (spectators) receive moves via WebSocket / SSE.
- Aggregate "famous games" feed: game-of-the-day, top-rated.

### Anti-cheat (engine assistance detection)
- Per-move accuracy comparison against engine analysis.
- Move-time patterns; sudden accuracy spikes.
- Out of LLD scope but worth mentioning if asked.

### Bullet / blitz timing
- Server-authoritative clocks (clients can't be trusted with timing).
- Per-move time deductions; flag-fall conditions.

---

## Persistence tradeoffs

A casual game lasts an hour. A correspondence game lasts weeks. Either way, persistence shapes everything the moment the game can outlive a process.

### What persistent data exists
- **Game state**: current position, whose turn, castling rights, en passant target, half-move clock, full-move number. Small (typically < 1KB).
- **Move history**: ordered list of moves. Grows linearly with length of game.
- **Player records**: ratings, game history.
- **Live game updates**: spectators / broadcast.

### Option 1 — Relational (PostgreSQL / MySQL)
```sql
games(id, white_id, black_id, started_at, ended_at, result, fen_state, move_count)
moves(game_id, ply_no, san_notation, from_file, from_rank, to_file, to_rank,
       captured_piece, promotion, kind, ts, PRIMARY KEY (game_id, ply_no))
players(id, username, rating)
```
- **FEN** (Forsyth-Edwards Notation) is the standard string serialization of a position; store it as a column for state recovery.
- Per move: insert into `moves`; update `games.fen_state`. One transaction.
- **Pros**: queries like "Alice's last 50 games" are SQL one-liners. Move list is naturally ordered.
- **Cons**: write rate during a blitz tournament can stress an OLTP DB. Per-move write per game × thousands of concurrent games.
- **When to pick**: standard for any chess platform. Lichess uses MongoDB but the shape is similar.

### Option 2 — Document (MongoDB)
- Store the entire game as one document: `{id, white, black, moves: [...], state, ...}`.
- Append moves via `$push`; read/write the whole doc atomically.
- **Pros**: single-document atomicity. Schema evolves with new fields easily.
- **Cons**: documents grow with move count (Mongo's 16MB limit is huge but not infinite). Long games (correspondence chess) need split storage.
- **When to pick**: when game data has irregular fields, or you want to embed the move list.

### Option 3 — Event sourcing
- Persist moves as events: `MoveMade{gameId, plyNo, from, to, ...}`.
- Game state is the fold: replay all moves to reconstruct.
- **Pros**: complete history is the storage; replays are trivial. Time-travel ("show me the position at move 42") is a slice.
- **Cons**: every "what's the current state" query requires fold or snapshot. Snapshots every N moves mitigate.
- **When to pick**: chess fits event sourcing better than most domains because the game *is* a sequence of events. Lichess's design has this flavor.

### Option 4 — Hybrid: hot in Redis + cold in SQL
- **Redis** holds active games (FEN + move list as a Redis list). Sub-millisecond reads/writes.
- **SQL** archives finished games for queries and analytics.
- **Pros**: latency on the move path is minimal.
- **Cons**: two stores; need a "finalize" step on game-end to flush to SQL.

### Option 5 — Per-game append-only file
- Each game is a file: `games/{id}.pgn` (Portable Game Notation, the chess standard).
- Fastest for write; trivial for human inspection.
- **Cons**: not queryable; no concurrency for a single game (one writer).
- **When to pick**: archival side of a hybrid; not the hot path.

### Recommendation
- **Active games**: Redis or in-memory with periodic snapshots.
- **Finished games**: PostgreSQL (`games` + `moves` tables) plus PGN archive in S3 for download.
- **Real-time spectator feed**: Redis pub/sub or Kafka.

### Concurrency under persistence
For one game: at most one writer at a time (the player whose turn it is). Use:
- **SQL**: `UPDATE games SET fen_state=?, version=version+1 WHERE id=? AND version=?` — optimistic.
- **Redis**: `WATCH game:{id}; ... ; MULTI; ... ; EXEC` — optimistic.
- **MongoDB**: conditional `findAndModify` with version field.

The pattern: read state, validate move, apply, write conditionally. If write loses the version race, reload and retry validation (the opponent's move has now landed).

For correspondence chess (slow): no concurrency concerns; latency dominates.

---

## Talking points for the interview

- "I split pseudo-legal moves (geometry) from legal moves (no king exposure). Pieces own geometry; the Game owns rules. That separation lets the AI generate candidate moves cheaply without check overhead."
- "Check detection runs `pseudoLegalMoves` for the opponent against the king's square — re-using piece logic instead of duplicating."
- "Castling is the King's move but needs Game-level state (rook position, attacked squares); I add castling moves *in the Game*, not in `King`. Keeps `Piece` ignorant of game rules."
- "Memento for undo because inverse-move logic is full of special cases — castling, promotion, en passant. The Memento is brute-force correct."
- "I use `synchronized` per-game; multiple games are independent. No global lock."
- "If we needed event-sourced storage, the `Move` record is already the event — slot it into Kafka without schema changes."

---

## Summary

Patterns load-bearing: **Strategy** (per-piece move generation), **State** (game lifecycle), **Memento** (undo), **Command** (move history).

The mental model: pieces produce candidate moves; the Game filters, applies, and records. Check / checkmate / stalemate are derived properties of the position, not state stored on pieces.

Chess is the canonical "test how you decompose responsibilities" problem. The piece/board/game split is the load-bearing decision; everything else flows from it.
