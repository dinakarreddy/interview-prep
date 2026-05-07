# Problem 9 — Splitwise

A graph + balance + Strategy problem. Tests your handling of money math (`BigDecimal`, currencies), polymorphic split types, and the surprisingly subtle problem of debt simplification.

## Interview opener

> **Interviewer:** Design a Splitwise-like expense splitting system.

---

## Clarification phase

> **You:** Users add expenses, system tracks who owes whom?
>
> **Interviewer:** Yes.

> **You:** Expenses can be split equally, by exact amounts, by percentages, by share weights?
>
> **Interviewer:** All of those.

> **You:** Groups — expenses scoped to a group, or also direct one-to-one?
>
> **Interviewer:** Both. A group is a convenience; the underlying model is pairwise balances.

> **You:** Settlement — how is "I paid X back to Y" recorded?
>
> **Interviewer:** Yes — settlements are just expenses with the payer/payee inverted. They reduce the debt.

> **You:** Multi-currency?
>
> **Interviewer:** Single currency for the core. Mention how multi-currency would extend.

> **You:** Debt simplification — when A owes B and B owes C, can we collapse to A owes C?
>
> **Interviewer:** Yes — discuss the algorithm; it's a known graph problem.

> **You:** Should I worry about negative numbers, idempotency, partial settlements?
>
> **Interviewer:** Yes — production-grade money handling.

> **You:** Concurrency — two people add expenses to the same group simultaneously?
>
> **Interviewer:** Yes; balances must be correct.

> **You:** Persistence?
>
> **Interviewer:** Discuss at the end.

OK — design needs: users, groups, expenses with polymorphic splits, balance graph, debt simplification, BigDecimal money handling, concurrency-safe expense addition.

---

## Refined requirements

### Functional
1. Create users; create groups (or use ad-hoc pairs).
2. Add expense: `(payer, totalAmount, splits[])` where `splits` describes how others share it.
3. Query balances: per pair (`Alice owes Bob ₹X`) or per user (net balance across the system).
4. Settle: a payment from one user to another, reducing the debt.
5. Simplify debts within a group: reduce N×N pair-debts to fewest transactions.

### Non-functional
- **Money correctness**: `BigDecimal` only; rounding modes specified; total split amounts equal expense amount exactly.
- **Concurrency**: balance updates are atomic.
- **Auditability**: every expense and settlement is a permanent record.

### Out of scope (mentioned)
- Multi-currency (extension).
- Receipts / OCR.
- Notifications (extension).
- Payment integrations (we just record settlements; the actual money-movement is external).

### Core operations
1. `addExpense(group, payer, amount, splits)` → `Expense`
2. `settle(from, to, amount)` → `Expense` (settlement is just a special expense)
3. `balance(user1, user2)` → `Money` (positive = user1 owes user2)
4. `userNetBalance(user)` → `Money`
5. `simplifyDebts(group)` → list of suggested settlements

---

## Domain model

### Entities (have ID)
- **`User`**.
- **`Group`** — a collection of users sharing expenses.
- **`Expense`** — payer, splits, total amount, description.

### Value objects (immutable)
- **`Money(amount, currency)`**.
- **`Split`** — abstract base; `EqualSplit`, `ExactSplit(amount)`, `PercentSplit(percent)`, `ShareSplit(shares)`.

### Aggregate boundaries
- `Group` is an aggregate root. Expenses in a group reference the group by ID.
- `User` is its own aggregate.
- The **balance graph** is a derived projection — built from expenses; not separately persisted in the simplest model.

### Patterns
- **Strategy** — `Split` polymorphism (each split type has its own resolution to per-user amounts).
- **Composite** — group memberships and expense splits.
- **Repository** — `ExpenseRepository`, `UserRepository`, `GroupRepository`.
- **Visitor** (extension) — operations over expenses (audit reports, analytics).
- **Event sourcing** (extension) — expenses are events; balance graph is the fold.

---

## Class diagram

```
                ┌──────────────────────────────┐
                │   <<abstract>> Split          │
                ├──────────────────────────────┤
                │ -user : User                  │
                ├──────────────────────────────┤
                │ +amountFor(total, otherSplits)│
                │   : Money                     │
                └──────────────────────────────┘
                          △
        ┌─────────────────┼─────────────────┬────────────────┐
        │                 │                 │                │
┌──────────────┐ ┌────────────────┐ ┌────────────────┐ ┌──────────────┐
│ EqualSplit   │ │ ExactSplit     │ │ PercentSplit   │ │ ShareSplit   │
│ (no extra)   │ │ -amount: Money │ │ -percent: BigD │ │ -shares: int │
└──────────────┘ └────────────────┘ └────────────────┘ └──────────────┘

┌──────────────────────────────┐
│            Expense           │
├──────────────────────────────┤
│ -id : String                 │
│ -groupId : String (nullable) │
│ -payer : User                │
│ -totalAmount : Money         │
│ -splits : List<Split>        │
│ -description : String        │
│ -createdAt : Instant         │
└──────────────────────────────┘

┌──────────────────────────────┐
│            Group             │
├──────────────────────────────┤
│ -id : String                 │
│ -name : String               │
│ -members : List<User>        │
└──────────────────────────────┘

┌──────────────────────────────┐
│       BalanceTracker         │
├──────────────────────────────┤
│ -balances : Map<Pair, Money> │
├──────────────────────────────┤
│ +recordExpense(Expense)      │
│ +balance(u1, u2) : Money     │
│ +netBalance(user) : Money    │
│ +simplify(group) : List<Sett>│
└──────────────────────────────┘

  record UserPair(String userIdA, String userIdB)   (canonical: A < B lexicographically)
```

---

## Key abstractions

```java
public abstract class Split {
    protected final User user;
    protected Split(User user) { this.user = user; }
    public User user() { return user; }

    /**
     * Returns this user's share of the expense.
     * Some splits depend on the other splits (e.g., EqualSplit divides evenly).
     */
    public abstract Money amountFor(Money total, List<Split> allSplits, RoundingMode rm);
}
```

---

## Core code

### Value objects + enums

```java
public record Money(BigDecimal amount, Currency currency) {
    public Money {
        Objects.requireNonNull(amount);
        Objects.requireNonNull(currency);
        amount = amount.setScale(currency.getDefaultFractionDigits(), RoundingMode.HALF_UP);
    }
    public Money plus(Money o)       { ensureSameCurrency(o); return new Money(amount.add(o.amount), currency); }
    public Money minus(Money o)      { ensureSameCurrency(o); return new Money(amount.subtract(o.amount), currency); }
    public Money negate()            { return new Money(amount.negate(), currency); }
    public boolean isZero()          { return amount.compareTo(BigDecimal.ZERO) == 0; }
    public boolean isPositive()      { return amount.compareTo(BigDecimal.ZERO) > 0; }
    public boolean isNegative()      { return amount.compareTo(BigDecimal.ZERO) < 0; }
    public static Money zero(Currency c) { return new Money(BigDecimal.ZERO, c); }
    private void ensureSameCurrency(Money o) {
        if (!currency.equals(o.currency)) throw new IllegalArgumentException("Currency mismatch");
    }
}
```

### Split implementations

```java
public final class EqualSplit extends Split {
    public EqualSplit(User user) { super(user); }

    @Override
    public Money amountFor(Money total, List<Split> allSplits, RoundingMode rm) {
        // Equal share: total / N, with rounding remainder distributed
        BigDecimal n = BigDecimal.valueOf(allSplits.size());
        BigDecimal share = total.amount().divide(n, total.currency().getDefaultFractionDigits(), rm);
        // Note: equal-split "remainder" handled at the orchestration level (see addExpense).
        return new Money(share, total.currency());
    }
}

public final class ExactSplit extends Split {
    private final Money fixed;
    public ExactSplit(User user, Money fixed) { super(user); this.fixed = fixed; }
    @Override
    public Money amountFor(Money total, List<Split> allSplits, RoundingMode rm) { return fixed; }
}

public final class PercentSplit extends Split {
    private final BigDecimal percent;   // e.g., 25.00 for 25%
    public PercentSplit(User user, BigDecimal percent) { super(user); this.percent = percent; }
    @Override
    public Money amountFor(Money total, List<Split> allSplits, RoundingMode rm) {
        BigDecimal share = total.amount()
            .multiply(percent)
            .divide(BigDecimal.valueOf(100), total.currency().getDefaultFractionDigits(), rm);
        return new Money(share, total.currency());
    }
}

public final class ShareSplit extends Split {
    private final int shares;
    public ShareSplit(User user, int shares) { super(user); this.shares = shares; }
    public int shares() { return shares; }
    @Override
    public Money amountFor(Money total, List<Split> allSplits, RoundingMode rm) {
        int totalShares = allSplits.stream()
            .filter(s -> s instanceof ShareSplit)
            .mapToInt(s -> ((ShareSplit) s).shares()).sum();
        BigDecimal share = total.amount()
            .multiply(BigDecimal.valueOf(shares))
            .divide(BigDecimal.valueOf(totalShares), total.currency().getDefaultFractionDigits(), rm);
        return new Money(share, total.currency());
    }
}
```

### `Expense` — value object–ish entity

```java
public final class Expense {
    private final String id;
    private final String groupId;        // nullable
    private final User payer;
    private final Money totalAmount;
    private final List<Split> splits;    // immutable
    private final String description;
    private final Instant createdAt;

    public Expense(String id, String groupId, User payer, Money totalAmount,
                   List<Split> splits, String description, Instant createdAt) {
        this.id = id; this.groupId = groupId; this.payer = payer;
        this.totalAmount = totalAmount; this.splits = List.copyOf(splits);
        this.description = description; this.createdAt = createdAt;
    }

    public String id()             { return id; }
    public Optional<String> groupId() { return Optional.ofNullable(groupId); }
    public User payer()            { return payer; }
    public Money totalAmount()     { return totalAmount; }
    public List<Split> splits()    { return splits; }
    public String description()    { return description; }
    public Instant createdAt()     { return createdAt; }

    /** Resolve splits to per-user owed amounts. */
    public Map<User, Money> resolveSplits(RoundingMode rm) {
        var owed = new LinkedHashMap<User, Money>();
        for (Split s : splits) owed.put(s.user(), s.amountFor(totalAmount, splits, rm));

        // Sanity check: split amounts sum to total (with rounding tolerance).
        Money sum = owed.values().stream().reduce(Money.zero(totalAmount.currency()), Money::plus);
        if (!sum.equals(totalAmount)) {
            // Distribute the rounding remainder to one of the splits (largest, conventionally).
            // For brevity: add the diff to the first split.
            Money diff = totalAmount.minus(sum);
            User first = owed.keySet().iterator().next();
            owed.put(first, owed.get(first).plus(diff));
        }
        return owed;
    }
}
```

The "remainder" handling is critical for equal splits where `total / n` doesn't divide evenly. Distribute the remainder deterministically (e.g., to the alphabetically-first user) so re-resolving the same expense gives the same answer.

### `BalanceTracker` — the balance graph

```java
public record UserPair(String userA, String userB) {
    public static UserPair canonical(User a, User b) {
        if (a.id().compareTo(b.id()) <= 0) return new UserPair(a.id(), b.id());
        return new UserPair(b.id(), a.id());
    }
}

public final class BalanceTracker {
    /** balances[(A, B)] = positive means A owes B; negative means B owes A. */
    private final Map<UserPair, BigDecimal> balances = new ConcurrentHashMap<>();
    private final Currency currency;
    private final RoundingMode roundingMode;

    public BalanceTracker(Currency c, RoundingMode rm) {
        this.currency = c; this.roundingMode = rm;
    }

    public void recordExpense(Expense e) {
        Map<User, Money> owed = e.resolveSplits(roundingMode);
        for (var entry : owed.entrySet()) {
            User debtor = entry.getKey();
            Money amount = entry.getValue();
            if (debtor.equals(e.payer())) continue;   // payer doesn't owe themselves
            adjust(debtor, e.payer(), amount.amount());
        }
    }

    /** Adjust `debtor` owes `creditor` by +amount (debtor's debt to creditor increases). */
    private void adjust(User debtor, User creditor, BigDecimal amount) {
        UserPair key = UserPair.canonical(debtor, creditor);
        // Positive in canonical key means userA owes userB.
        BigDecimal signedDelta = key.userA().equals(debtor.id()) ? amount : amount.negate();
        balances.merge(key, signedDelta, BigDecimal::add);
    }

    /** Returns positive if debtor owes creditor; negative if creditor owes debtor. */
    public Money balance(User debtor, User creditor) {
        UserPair key = UserPair.canonical(debtor, creditor);
        BigDecimal v = balances.getOrDefault(key, BigDecimal.ZERO);
        BigDecimal signed = key.userA().equals(debtor.id()) ? v : v.negate();
        return new Money(signed, currency);
    }

    public Money netBalance(User user) {
        BigDecimal net = BigDecimal.ZERO;
        for (var e : balances.entrySet()) {
            if (e.getKey().userA().equals(user.id())) net = net.subtract(e.getValue());   // owed by user
            if (e.getKey().userB().equals(user.id())) net = net.add(e.getValue());        // owed to user
        }
        return new Money(net, currency);
    }

    /**
     * Simplify the debt graph for a group of users.
     * Returns minimal-ish list of settlements that zeros out balances.
     */
    public List<Settlement> simplify(List<User> users) {
        // Compute net balance per user
        Map<String, BigDecimal> net = new HashMap<>();
        for (User u : users) net.put(u.id(), netBalance(u).amount());

        // Greedy: match largest creditor with largest debtor; settle smaller-of-the-two; repeat.
        var creditors = new TreeMap<BigDecimal, Deque<String>>(Comparator.reverseOrder());
        var debtors   = new TreeMap<BigDecimal, Deque<String>>();   // ascending (most negative first)
        for (var e : net.entrySet()) {
            BigDecimal v = e.getValue();
            if (v.compareTo(BigDecimal.ZERO) > 0) creditors.computeIfAbsent(v, k -> new ArrayDeque<>()).add(e.getKey());
            else if (v.compareTo(BigDecimal.ZERO) < 0) debtors.computeIfAbsent(v, k -> new ArrayDeque<>()).add(e.getKey());
        }

        var settlements = new ArrayList<Settlement>();
        while (!creditors.isEmpty() && !debtors.isEmpty()) {
            var cEntry = creditors.firstEntry(); BigDecimal cAmt = cEntry.getKey(); String cUser = cEntry.getValue().pollFirst();
            var dEntry = debtors.firstEntry();   BigDecimal dAmt = dEntry.getKey(); String dUser = dEntry.getValue().pollFirst();

            BigDecimal pay = cAmt.min(dAmt.negate());
            settlements.add(new Settlement(dUser, cUser, new Money(pay, currency)));

            // Reinsert remainders
            BigDecimal cRem = cAmt.subtract(pay);
            BigDecimal dRem = dAmt.add(pay);
            cleanup(creditors, cEntry, cUser);
            cleanup(debtors,   dEntry, dUser);
            if (cRem.signum() > 0) creditors.computeIfAbsent(cRem, k -> new ArrayDeque<>()).add(cUser);
            if (dRem.signum() < 0) debtors.computeIfAbsent(dRem, k -> new ArrayDeque<>()).add(dUser);
        }
        return settlements;
    }

    private void cleanup(TreeMap<BigDecimal, Deque<String>> map, Map.Entry<BigDecimal, Deque<String>> entry, String user) {
        if (entry.getValue().isEmpty()) map.remove(entry.getKey());
    }

    public record Settlement(String fromUser, String toUser, Money amount) { }
}
```

The greedy debt-simplification algorithm is **not optimal** in the formal sense (the optimal version is NP-hard — equivalent to the 3-partition problem). Greedy gives a good practical answer in O(N log N).

### Service orchestration

```java
public final class ExpenseService {
    private final ExpenseRepository expenseRepo;
    private final UserRepository userRepo;
    private final GroupRepository groupRepo;
    private final BalanceTracker balanceTracker;
    private final Clock clock;
    private final IdGenerator ids;

    public ExpenseService(ExpenseRepository er, UserRepository ur, GroupRepository gr,
                          BalanceTracker bt, Clock clock, IdGenerator ids) {
        this.expenseRepo = er; this.userRepo = ur; this.groupRepo = gr;
        this.balanceTracker = bt; this.clock = clock; this.ids = ids;
    }

    public Expense addExpense(String groupId, User payer, Money total,
                              List<Split> splits, String description) {
        validateSplits(total, splits);
        Expense e = new Expense(ids.next(), groupId, payer, total, splits, description, clock.instant());
        expenseRepo.save(e);
        balanceTracker.recordExpense(e);
        return e;
    }

    public Expense settle(User from, User to, Money amount) {
        // A settlement is just an expense paid by 'from' for 'to' — i.e., 'to' owes 'from' nothing,
        // or modeled as: 'from' pays the total; 'to' is the only split.
        return addExpense(null, from, amount,
            List.of(new ExactSplit(to, amount)),
            "Settlement: " + from.id() + " → " + to.id());
    }

    private void validateSplits(Money total, List<Split> splits) {
        if (splits.isEmpty()) throw new IllegalArgumentException("No splits");
        // Type-specific validation
        if (splits.stream().anyMatch(s -> s instanceof ExactSplit)) {
            BigDecimal sum = splits.stream()
                .filter(s -> s instanceof ExactSplit)
                .map(s -> ((ExactSplit) s).fixed.amount())
                .reduce(BigDecimal.ZERO, BigDecimal::add);
            // (Mixed types could also be supported; here we assume all-or-none of a given type)
            if (sum.compareTo(total.amount()) != 0) throw new IllegalArgumentException("Exact splits don't sum to total");
        }
        if (splits.stream().anyMatch(s -> s instanceof PercentSplit)) {
            BigDecimal totalPct = splits.stream()
                .filter(s -> s instanceof PercentSplit)
                .map(s -> ((PercentSplit) s).percent)
                .reduce(BigDecimal.ZERO, BigDecimal::add);
            if (totalPct.compareTo(BigDecimal.valueOf(100)) != 0) throw new IllegalArgumentException("Percentages don't sum to 100");
        }
    }
}
```

---

## Concurrency strategy

### What's contested
- The balance map (concurrent expense additions modify the same pair).

### Strategy
- **`ConcurrentHashMap.merge`** for the balance map — atomic add. No explicit lock needed.
- **Expense save** and **balance update** are *not* atomic together unless wrapped in a transaction (in-memory: synchronize the service method; persistent: DB transaction).

### Why not use one global lock?
- Independent group expenses don't conflict.
- The balance map's pair-level granularity is sufficient.

### Production concern: balance and expense must be consistent
- Two operations: (1) save expense, (2) update balance.
- If (1) succeeds and (2) fails, balances drift from truth.
- Solutions:
  1. **DB transaction**: both updates in one transaction.
  2. **Event sourcing**: only persist expenses; balance is a projection rebuilt from expenses.

The event-sourced model is cleaner for Splitwise — see Persistence section.

---

## Edge cases / gotchas

| Case | Handling |
|---|---|
| **Equal split with non-divisible total** | Distribute remainder deterministically (e.g., add 1 paisa to first split). |
| **Percent splits not summing to 100%** | Reject at validation. |
| **Exact splits not summing to total** | Reject. |
| **Zero or negative expense** | Reject. |
| **Payer included in splits** | Allowed; payer just owes themselves their share, which cancels out. |
| **Self-settlement (A pays A)** | Reject. |
| **Currency mismatch** | Reject in `Money.plus`. |
| **Concurrent expense to same group** | `ConcurrentHashMap.merge` ensures correctness. |
| **Expense edit / delete** | Reverse the original (negative-of-each-split), then add the new. Audit-friendly. |
| **Decimal precision** | `BigDecimal` with currency-specific scale; rounding mode declared. |
| **Settlement of an exact balance** | Becomes a zero net change in the pair. |

---

## Extensions / interview follow-ups

### "Multi-currency."
- `Group` has a default currency.
- Expenses can be in a different currency; FX rate snapshot at expense time.
- Balances are tracked per-currency-pair, OR converted to a base currency at expense time (different correctness implications).
- Real Splitwise tracks per-currency balances and lets users settle in whichever currency makes sense.

### "Edit / delete an expense."
- Soft-delete: mark expense as deleted; create a reversing entry (each split negated). Keeps audit trail.
- For event-sourced model: emit a `ExpenseReversed` event; balance projection consumes it.

### "Recurring expenses (rent, utilities)."
- `RecurringExpense` schedule with cron-like interval.
- Background scheduler creates the actual `Expense` records on each occurrence.

### "Receipt OCR / itemization."
- Out of LLD scope — extract line items, categorize, suggest splits per item.

### "Comments / activity log."
- `Comment` entity attached to expenses.
- Group activity feed = ordered union of expenses + comments.

### "Why not store the balance graph as a separate aggregate, persisted directly?"
- It can be — that's a denormalized projection.
- Risk: balance updates can drift from expenses if the two writes aren't transactional.
- Cleanest: balance is **derived state**, recomputed from expenses (event sourcing) or in a single transaction with the expense.

### "Why is debt simplification useful?"
- In a group of N people, the natural balance graph has up to N(N-1)/2 edges.
- After many cross-payments, the graph is dense. Simplification gives users a small list of payments to settle everything.
- Practical impact: instead of "Alice pays Bob ₹50, Bob pays Carol ₹50, Carol pays Alice ₹50" (a cycle that nets to zero), simplification shows zero owed.

### "Is the simplification optimal?"
- Optimal minimum-transactions is NP-hard (3-partition reduction).
- Greedy gives a near-optimal solution in linear time.
- Real Splitwise uses greedy. The user-facing UX is good enough that optimality isn't worth the complexity.

---

## Scaling

A single-process system handles thousands of users easily. At Splitwise scale (millions):

### Sharding
- Shard by `groupId` — group-internal ops stay on one shard.
- Cross-group queries (user net balance across all groups) need scatter-gather.

### Caching balances
- Balance graph lookups are hot.
- Cache per-user net balance with invalidation on every expense affecting that user.
- Redis-backed cache; invalidate via pub/sub on expense events.

### Notifications
- Adding an expense fans out notifications to all affected users.
- Async via a queue; back-pressure on rate.

### Real-time updates
- Mobile apps subscribe to group events.
- WebSocket / push for live updates.

### Analytics
- Aggregate spending per category, per period.
- Stream expenses to a warehouse (BigQuery / Snowflake) for analytics.

---

## Persistence tradeoffs

Money + audit + multi-user concurrency = persistence is non-negotiable for any real Splitwise.

### What persistent data exists
- **Users**: profiles. Static-ish.
- **Groups**: members. Slowly-changing.
- **Expenses**: append-mostly; rarely deleted (soft-delete preferred).
- **Balances**: derived; either persisted as cache or recomputed from expenses.

### Option 1 — Relational (PostgreSQL / MySQL) with explicit balance table
```sql
users(id, name, email, ...)
groups(id, name)
group_members(group_id, user_id, joined_at)
expenses(id, group_id, payer_id, total_amount, currency, description, created_at, deleted)
splits(expense_id, user_id, amount, split_type)   -- resolved amounts stored
balances(user_a_id, user_b_id, amount, currency, updated_at)   -- canonical pair: a < b
```
- **Each `addExpense` is one transaction**: insert expense, insert splits, update balance rows.
- **Pros**: ACID. Simple to read balance directly.
- **Cons**: balance row is a hot row for popular pairs. Doesn't gracefully handle "what was Alice's balance on date X" — would need to derive.
- **When to pick**: standard for any Splitwise-like product.

### Option 2 — Event-sourced (expenses as events; balance as projection)
- Persist only `expenses` and `splits`.
- Balance is computed by aggregating splits (`SELECT user_a, user_b, SUM(amount) FROM ...`).
- Cache the result in Redis or a `balances_view` materialized view.
- **Pros**: complete audit trail. No risk of balance/expense divergence — balance *is* the expense fold.
- **Cons**: read latency unless cached. Cache invalidation on every expense.
- **When to pick**: this fits the domain very well; it's how I'd build it from scratch.

### Option 3 — Document store (MongoDB / DynamoDB)
- Each expense as a document; group document holds metadata + member list.
- Balance computed on read or maintained as separate documents.
- **Pros**: schema flexibility for evolving split types.
- **Cons**: aggregations are weaker than SQL; multi-document transactions in Mongo (added in 4.0+) but less ergonomic.
- **When to pick**: schema is rapidly evolving; team prefers NoSQL.

### Option 4 — Hybrid (recommended for production scale)
- **PostgreSQL** for source of truth: users, groups, expenses, splits.
- **Redis** for hot balance lookups; invalidated on each expense.
- **Kafka** for event stream — drives notifications, analytics, search indexing.
- **S3 + analytics warehouse** for cold storage of historical expenses, BI queries.

### Concurrency and money correctness
- **`BigDecimal` throughout** — never `double` for money.
- **DB-level rounding consistent with app-level rounding** — declare both explicitly.
- **Idempotency keys on `addExpense`** — clients retry; we mustn't double-charge.
- **Optimistic locking on expense edits** — version column; affected-rows = 0 → retry with the new state.

### Settlements
- Modeled as expenses with one split (the recipient owes the payer the full amount, which cancels prior debt).
- Or as a separate `settlements` table — same data, different shape. Either works; the former is cleaner conceptually.

### Audit trail
- Never hard-delete expenses. Soft-delete with `deleted_at`, `deleted_by`.
- An expense edit creates a new "version" — the original is preserved.
- Some teams use append-only ledger semantics: every change is a new row; current state is the latest version.

### Recommendation
- **Default**: PostgreSQL, single transaction per `addExpense` (insert expense + splits + update balances).
- **At scale**: Add Redis for balance reads; Kafka event stream for fan-out.
- **For audit-heavy use cases**: event-sourced with materialized balance view.

---

## Talking points for the interview

- "Splits are a Strategy hierarchy because each type has different math. Validation lives at the orchestration layer because some constraints (percents sum to 100) are cross-split."
- "Balance is keyed by a *canonical* pair (lexicographically ordered user IDs) so `balance(A, B)` and `balance(B, A)` access the same row, with the sign indicating direction."
- "Equal-split rounding remainders are distributed deterministically — re-resolving an expense always gives the same answer."
- "Debt simplification is greedy because optimal is NP-hard; greedy is fast and good enough for users."
- "Settlements are just expenses with the payer/payee inverted — keeps the data model uniform."
- "I treat balances as derived state; either via DB transaction or event sourcing. Storing balance as a separate write opens the door to drift if not transactional."
- "BigDecimal everywhere — `double` for money is a bug waiting to happen."

---

## Summary

Patterns load-bearing: **Strategy** (Split), **Repository** (per aggregate), **event sourcing** (optional, fits the domain naturally).

The mental model: expenses are *facts*; balances are *derived projections*. Every interesting operation is "compute balance from expenses." The graph algorithm (debt simplification) is independent of the persistence model — give it a balance graph, it returns settlement suggestions.

The senior signal in this problem is: do you reach for `BigDecimal`? Do you handle equal-split rounding? Do you mention the NP-hardness of optimal simplification? Do you propose event sourcing for audit?
