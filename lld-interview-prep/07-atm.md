# Problem 7 — ATM

A state machine + transaction problem. Tests State pattern, transaction atomicity (debiting an account safely), cash dispensing logic, and the discipline to keep banking-side concerns separate from machine-side concerns.

## Interview opener

> **Interviewer:** Design an ATM.

---

## Clarification phase

> **You:** What operations? Withdraw, deposit, balance check — anything else?
>
> **Interviewer:** Those three for the core. Add transfer if you have time.

> **You:** Authentication — card + PIN?
>
> **Interviewer:** Yes.

> **You:** What happens after wrong PIN attempts? Block the card?
>
> **Interviewer:** Block after 3 attempts; eject card.

> **You:** Should the ATM dispense bills? Need to handle "out of cash" or "can't make exact denomination"?
>
> **Interviewer:** Yes. If the requested amount can't be dispensed with available denominations, refuse and offer alternative amounts.

> **You:** Single user at a time?
>
> **Interviewer:** Yes — physical machine.

> **You:** Network connection to the bank — what if it's down?
>
> **Interviewer:** Refuse transactions. Don't store credit; the bank is the source of truth.

> **You:** Concurrent operations on the same account from different ATMs?
>
> **Interviewer:** Yes — assume it's possible. Two ATMs can't double-spend the same balance.

> **You:** Daily withdrawal limits?
>
> **Interviewer:** Yes — per-account, configurable.

> **You:** Receipt printing, statement printing?
>
> **Interviewer:** Mention but don't implement.

> **You:** Maintenance mode for operators?
>
> **Interviewer:** Yes — simple toggle.

> **You:** Persistence — discuss at end?
>
> **Interviewer:** Yes.

OK — design needs: state machine for the ATM session, account abstraction (with concurrency-safe debits), bill dispensing logic, and a clean separation between the *physical machine* and the *banking system*.

---

## Refined requirements

### Functional
1. Insert card → authenticate (PIN, with 3-attempt lockout).
2. Choose operation: balance, withdraw, deposit, transfer.
3. Withdraw N → validate (sufficient funds, daily limit, dispensable amount), debit account, dispense bills.
4. Deposit N → accept envelope/cash, credit account.
5. Transfer N to account X → debit + credit atomically.
6. Eject card; print receipt.
7. Maintenance mode for operators.

### Non-functional
- **Concurrency**: same account from multiple ATMs / channels — must not double-spend.
- **Atomicity**: debit + dispense must not diverge (don't dispense without debit; don't debit without dispense).
- **Reliability**: network partitions, mid-transaction power loss must be recoverable.

### Out of scope (mentioned)
- Receipt printing / statement details.
- Card production / encoding standards.
- Currency conversion.
- Compliance / KYC.

---

## Domain model

### Entities
- **`ATM`** — aggregate root; the physical machine.
- **`CashDispenser`** — owned by ATM; tracks bill counts per denomination.
- **`Account`** — bank-side entity. Lives outside the ATM (in `BankService`).

### Value objects
- **`Card(number, holderName, expiry)`** — immutable.
- **`PinAttempt`**.
- **`Money(amount, currency)`**.

### State hierarchy (State pattern)
- `IdleState` — no card, awaiting input.
- `CardInsertedState` — awaiting PIN.
- `AuthenticatedState` — main menu after PIN ok.
- `TransactingState` — performing an operation.
- `MaintenanceState` — operator mode.
- `OutOfServiceState` — error / unrecoverable.

### Patterns
- **State** — ATM lifecycle.
- **Strategy** — bill dispensing algorithm; transaction type.
- **Command** — each transaction (withdraw, deposit, transfer) as a command for audit + retry.
- **Chain of Responsibility** — bill dispenser chain (₹2000 → ₹500 → ₹200 → ₹100, dispensing greedily).
- **Adapter** — wrapping external bank API behind a clean `BankService` interface.

---

## Class diagram

```
                ┌────────────────────────┐
                │   <<interface>>        │
                │    ATMState            │
                ├────────────────────────┤
                │ +insertCard(atm, card) │
                │ +enterPin(atm, pin)    │
                │ +selectOp(atm, op)     │
                │ +submitAmount(atm, m)  │
                │ +cancel(atm)           │
                │ +ejectCard(atm)        │
                └────────────────────────┘
                         △
       ┌─────────────────┼──────────────────┬───────────────┐
       │                 │                  │               │
┌─────────────┐  ┌──────────────────┐ ┌──────────────────┐ ┌─────────────┐
│  Idle       │  │ CardInserted     │ │  Authenticated   │ │ Transacting │
└─────────────┘  └──────────────────┘ └──────────────────┘ └─────────────┘

┌────────────────────────────────────┐
│              ATM                   │
├────────────────────────────────────┤
│ -id : String                       │
│ -state : ATMState                  │
│ -cashDispenser : CashDispenser     │
│ -bankService : BankService         │
│ -currentSession : Session (null OK)│
├────────────────────────────────────┤
│ +insertCard / enterPin / ...       │
│ +setState(ATMState) [pkg]          │
└────────────────────────────────────┘
        │           │
        ◆ 1         ◆ 1
        ▼           ▼
┌──────────────────┐ ┌────────────────────────┐
│  CashDispenser   │ │  <<interface>>          │
├──────────────────┤ │    BankService          │
│ -counts : Map<>  │ ├────────────────────────┤
│ +canDispense(amt)│ │ +authenticate(card, pin)│
│ +dispense(amt)   │ │ +balance(account)      │
└──────────────────┘ │ +withdraw(account, amt)│
                     │ +deposit(account, amt) │
                     │ +transfer(from, to, m) │
                     └────────────────────────┘

┌────────────────────────────┐
│        Session             │
├────────────────────────────┤
│ -card : Card               │
│ -accountId : String        │
│ -authToken : String        │
│ -dailyWithdrawn : Money    │
│ -pinAttempts : int         │
└────────────────────────────┘
```

---

## Key abstractions

```java
public interface ATMState {
    void insertCard(ATM atm, Card card);
    void enterPin(ATM atm, String pin);
    void selectOperation(ATM atm, Operation op);
    void submitAmount(ATM atm, Money amount);
    void cancel(ATM atm);
    void ejectCard(ATM atm);
    String name();
}

public interface BankService {
    AuthResult authenticate(Card card, String pin);
    Money balance(String accountId, String authToken);
    /** Atomic debit; returns confirmation or throws. */
    TxnResult withdraw(String accountId, String authToken, Money amount, String idempotencyKey);
    TxnResult deposit(String accountId, String authToken, Money amount, String idempotencyKey);
    TxnResult transfer(String fromAccount, String toAccount, String authToken, Money amount, String idempotencyKey);
}

public interface BillDispensingStrategy {
    /** Returns Optional.empty() if amount cannot be made with available bills. */
    Optional<Map<Integer, Integer>> compute(int amount, Map<Integer, Integer> available);
}
```

`BankService` is **the boundary** between machine and bank. The implementation does the network call. The interface is what the ATM depends on — testable, swappable.

`idempotencyKey` is critical: if the network drops between debit and confirmation, the ATM retries the same transaction with the same key — the bank dedupes.

---

## Core code

### Value objects + enums

```java
public enum Operation { BALANCE, WITHDRAW, DEPOSIT, TRANSFER }

public record Card(String number, String holderName, LocalDate expiry) { }
public record AuthResult(boolean success, String accountId, String authToken, int remainingAttempts) { }
public record TxnResult(boolean success, String txnId, Money newBalance, String message) { }
```

### `CashDispenser` + greedy bill dispensing

```java
public final class CashDispenser {
    // Denomination (rupees) -> count
    private final TreeMap<Integer, Integer> counts;

    public CashDispenser(Map<Integer, Integer> initial) {
        this.counts = new TreeMap<>(Comparator.reverseOrder());   // highest first
        counts.putAll(initial);
    }

    public synchronized boolean canDispense(int amount) {
        return computeBills(amount).isPresent();
    }

    public synchronized Map<Integer, Integer> dispense(int amount) {
        Map<Integer, Integer> bills = computeBills(amount)
            .orElseThrow(() -> new InsufficientCashException(amount));
        bills.forEach((denom, n) -> counts.merge(denom, -n, Integer::sum));
        return bills;
    }

    private Optional<Map<Integer, Integer>> computeBills(int amount) {
        var result = new LinkedHashMap<Integer, Integer>();
        int remaining = amount;
        for (var e : counts.entrySet()) {
            int denom = e.getKey(), have = e.getValue();
            int use = Math.min(remaining / denom, have);
            if (use > 0) {
                result.put(denom, use);
                remaining -= use * denom;
            }
        }
        return remaining == 0 ? Optional.of(result) : Optional.empty();
    }

    public synchronized void replenish(int denom, int n) { counts.merge(denom, n, Integer::sum); }

    public synchronized Map<Integer, Integer> snapshot() { return new LinkedHashMap<>(counts); }
}
```

For canonical denominations (₹100, 200, 500, 2000), greedy is optimal. For non-canonical sets, swap in a DP-based `BillDispensingStrategy`.

### State implementations (excerpt)

```java
public final class IdleState implements ATMState {
    public void insertCard(ATM atm, Card card) {
        if (card.expiry().isBefore(LocalDate.now())) {
            atm.physicallyEjectCard();
            return;
        }
        atm.startSession(card);
        atm.setState(new CardInsertedState());
    }
    public void enterPin(ATM atm, String pin)      { throw new IllegalStateException("Insert card first"); }
    public void selectOperation(ATM atm, Operation op) { throw new IllegalStateException("Insert card first"); }
    public void submitAmount(ATM atm, Money m)     { throw new IllegalStateException("Insert card first"); }
    public void cancel(ATM atm)                    { /* idempotent */ }
    public void ejectCard(ATM atm)                 { /* nothing to eject */ }
    public String name()                           { return "IDLE"; }
}

public final class CardInsertedState implements ATMState {
    public void insertCard(ATM atm, Card card)     { throw new IllegalStateException("Card already inserted"); }
    public void enterPin(ATM atm, String pin) {
        AuthResult r = atm.bankService().authenticate(atm.session().card(), pin);
        if (r.success()) {
            atm.session().authenticated(r.accountId(), r.authToken());
            atm.setState(new AuthenticatedState());
        } else {
            atm.session().incrementPinAttempts();
            if (atm.session().pinAttempts() >= 3) {
                atm.captureCard();   // retain the card per security policy
                atm.endSession();
                atm.setState(new IdleState());
            }
            // else stay in this state for retry
        }
    }
    public void selectOperation(ATM atm, Operation op) { throw new IllegalStateException("Authenticate first"); }
    public void submitAmount(ATM atm, Money m)     { throw new IllegalStateException("Authenticate first"); }
    public void cancel(ATM atm) {
        atm.physicallyEjectCard();
        atm.endSession();
        atm.setState(new IdleState());
    }
    public void ejectCard(ATM atm)                 { cancel(atm); }
    public String name()                           { return "CARD_INSERTED"; }
}

public final class AuthenticatedState implements ATMState {
    public void insertCard(ATM atm, Card card)     { throw new IllegalStateException("Eject card first"); }
    public void enterPin(ATM atm, String pin)      { /* already authenticated */ }
    public void selectOperation(ATM atm, Operation op) {
        if (op == Operation.BALANCE) {
            // Quick read-only path
            Money balance = atm.bankService().balance(atm.session().accountId(), atm.session().authToken());
            atm.display(balance);
            // Stay in AuthenticatedState
            return;
        }
        atm.session().selectOperation(op);
        atm.setState(new TransactingState());
    }
    public void submitAmount(ATM atm, Money m)     { throw new IllegalStateException("Select operation first"); }
    public void cancel(ATM atm) {
        atm.physicallyEjectCard();
        atm.endSession();
        atm.setState(new IdleState());
    }
    public void ejectCard(ATM atm)                 { cancel(atm); }
    public String name()                           { return "AUTHENTICATED"; }
}

public final class TransactingState implements ATMState {
    public void submitAmount(ATM atm, Money amount) {
        Operation op = atm.session().selectedOperation();
        switch (op) {
            case WITHDRAW -> handleWithdraw(atm, amount);
            case DEPOSIT  -> handleDeposit(atm, amount);
            case TRANSFER -> handleTransfer(atm, amount);
            default       -> throw new IllegalStateException("Unsupported op");
        }
    }

    private void handleWithdraw(ATM atm, Money amount) {
        // Pre-checks: daily limit, dispensability
        if (!atm.checkDailyLimit(amount)) throw new DailyLimitExceededException();
        if (!atm.cashDispenser().canDispense((int) amount.amount().longValue()))
            throw new CannotDispenseDenominationException();

        // Atomic with the bank — idempotent retry on partial failure
        String idem = atm.session().beginTransactionId();
        TxnResult r = atm.bankService().withdraw(atm.session().accountId(),
                                                  atm.session().authToken(),
                                                  amount, idem);
        if (!r.success()) throw new TransactionFailedException(r.message());

        // Bank confirmed — physical dispense
        Map<Integer, Integer> bills = atm.cashDispenser().dispense((int) amount.amount().longValue());
        atm.physicallyDispenseBills(bills);
        atm.session().recordWithdrawal(amount);

        atm.printReceipt(r);
        atm.setState(new AuthenticatedState());
    }

    private void handleDeposit(ATM atm, Money amount)  { /* similar */ }
    private void handleTransfer(ATM atm, Money amount) { /* similar; also takes target account */ }

    public void insertCard(ATM atm, Card card)     { throw new IllegalStateException(); }
    public void enterPin(ATM atm, String pin)      { /* ignored */ }
    public void selectOperation(ATM atm, Operation op) { throw new IllegalStateException("Mid-transaction"); }
    public void cancel(ATM atm) {
        // Mid-transaction cancel: only valid before bank-confirmation step.
        // After bank-confirmed but before dispense, we MUST dispense — already debited.
        atm.setState(new AuthenticatedState());
    }
    public void ejectCard(ATM atm)                 { cancel(atm); }
    public String name()                           { return "TRANSACTING"; }
}
```

### `ATM` — Context

```java
public final class ATM {
    private ATMState state;
    private final CashDispenser cashDispenser;
    private final BankService bankService;
    private Session currentSession;
    private final Money dailyLimit;

    public ATM(CashDispenser dispenser, BankService bank, Money dailyLimit) {
        this.cashDispenser = dispenser;
        this.bankService = bank;
        this.dailyLimit = dailyLimit;
        this.state = new IdleState();
    }

    public synchronized void insertCard(Card c)              { state.insertCard(this, c); }
    public synchronized void enterPin(String pin)            { state.enterPin(this, pin); }
    public synchronized void selectOperation(Operation op)   { state.selectOperation(this, op); }
    public synchronized void submitAmount(Money m)           { state.submitAmount(this, m); }
    public synchronized void cancel()                        { state.cancel(this); }
    public synchronized void ejectCard()                     { state.ejectCard(this); }

    // Helpers used by states (package-private)
    void setState(ATMState s)              { this.state = s; }
    void startSession(Card c)              { this.currentSession = new Session(c); }
    void endSession()                      { this.currentSession = null; }
    Session session()                      { return currentSession; }
    BankService bankService()              { return bankService; }
    CashDispenser cashDispenser()          { return cashDispenser; }
    boolean checkDailyLimit(Money request) {
        var sum = currentSession.dailyWithdrawn().plus(request);
        return sum.amount().compareTo(dailyLimit.amount()) <= 0;
    }

    void physicallyEjectCard()             { /* hardware */ }
    void captureCard()                     { /* retain */ }
    void physicallyDispenseBills(Map<Integer, Integer> bills) { /* hardware */ }
    void display(Money balance)            { /* screen */ }
    void printReceipt(TxnResult r)         { /* printer */ }
}
```

### `Session`

```java
public final class Session {
    private final Card card;
    private String accountId;
    private String authToken;
    private Operation selectedOperation;
    private int pinAttempts = 0;
    private Money dailyWithdrawn = Money.zero(Currency.getInstance("INR"));
    private String txnIdSeed;

    public Session(Card c)                            { this.card = c; }
    public Card card()                                { return card; }
    public String accountId()                         { return accountId; }
    public String authToken()                         { return authToken; }
    public int pinAttempts()                          { return pinAttempts; }
    public Money dailyWithdrawn()                     { return dailyWithdrawn; }
    public Operation selectedOperation()              { return selectedOperation; }

    public void authenticated(String acc, String tok) { this.accountId = acc; this.authToken = tok; }
    public void incrementPinAttempts()                { pinAttempts++; }
    public void selectOperation(Operation op)         { this.selectedOperation = op; }
    public void recordWithdrawal(Money m)             { this.dailyWithdrawn = dailyWithdrawn.plus(m); }
    public String beginTransactionId() {
        // Deterministic per-session per-attempt — used for idempotency on retry
        return UUID.randomUUID().toString();
    }
}
```

---

## Concurrency strategy

### Two distinct concurrency stories

**1. Inside the ATM** — single user at a time. `synchronized` on every public method on `ATM`. State machine guarantees atomic transitions.

**2. Across the system** — same account, different channels (this ATM, another ATM, the mobile app, a bank teller). The **bank** is the source of truth and must enforce serialization on the account.

The key boundary: **the ATM never holds account state**. It reads via `bankService.balance(...)` and mutates via `bankService.withdraw(...)`. Concurrency safety on the account lives entirely on the bank side.

### How the bank handles it (for awareness)
- **Pessimistic**: `SELECT ... FOR UPDATE` locks the account row; debit; commit. Other transactions queue.
- **Optimistic**: `UPDATE accounts SET balance = balance - ? WHERE id = ? AND balance >= ? AND version = ?`. Affected rows = 0 → reject.
- Either way, the operation is one DB transaction; the ATM either gets a success or a clean failure.

### Idempotency for retries
The ATM's `withdraw` call may time out (network blip). The ATM must retry with the **same idempotency key** so the bank doesn't double-debit. The bank stores `(idempotencyKey → result)` for ~24h.

### What if the bank confirms but the dispense fails?
Worst-case scenario. The user has been debited but no cash came out. Mitigation:
1. Detect dispense failure (mechanical sensors).
2. Issue a **reversal** transaction back to the bank with the same idempotency key family.
3. Log incident; flag for operator review; reconcile by audit (the cash count vs the debit log).

This is why dispense → debit ordering matters: **debit first, dispense second** (rollback debit on dispense failure is recoverable; dispense without debit isn't — money is gone).

Actually, re-reading: in real ATMs the order is sometimes "debit and dispense are coupled in a single coordinated step with sensors." But the safer software model is: debit (with idempotency), then dispense, then on dispense failure, refund.

---

## Edge cases / gotchas

| Case | Handling |
|---|---|
| **Wrong PIN 3 times** | Capture card; reset state to IDLE. |
| **Cancel mid-PIN entry** | Eject card; reset to IDLE. |
| **Cancel mid-transaction (after bank debit)** | Forbidden — already committed. Continue dispense. |
| **Network down during auth** | Refuse; eject card. |
| **Network down after debit, before dispense** | Retry dispense locally; if fails, issue reversal next time bank is reachable. Critical: idempotency key. |
| **Dispenser jammed** | Mark out of service; alert operator. Reverse debit. |
| **Insufficient cash for amount** | Refuse before debit; suggest alternative amounts. |
| **Cannot make exact denomination** | Same — refuse and suggest. |
| **Card expired** | Reject at insert. |
| **Account frozen / blocked** | Bank returns error on auth; ATM displays and ejects. |
| **Daily limit exceeded** | Refuse before debit. |
| **Amount = 0 or negative** | Reject at validation. |
| **User walks away** | Inactivity timeout; eject card; return to IDLE. |
| **Maintenance during active session** | Forbidden — wait for session end. |

---

## Extensions / interview follow-ups

### "Add transfer between accounts."
- New operation: `TRANSFER` with target account.
- `BankService.transfer(from, to, amount, idem)` is one atomic operation (single transaction on the bank side covering both debit and credit).
- ATM doesn't need to coordinate — it's bank-side.

### "Add multi-currency support."
- Card → account → currency known to bank.
- ATM dispenser per currency. If user requests a currency the dispenser doesn't have, refuse.
- For cross-currency withdrawal: bank does FX conversion; ATM dispenses local currency.

### "Add card type variants — debit, credit, prepaid."
- All accessed via `BankService` — credit account check has different rules (credit limit vs balance).
- ATM doesn't care; bank-side abstraction handles the difference.

### "Why not store the account balance on the ATM?"
- ATMs would diverge from the bank.
- A user could double-spend by attacking one ATM's storage.
- The bank is the single source of truth. The ATM is a thin presentation layer.

### "What's the State pattern adding here vs `if (status == X)`?"
- 6 states × 6 operations = 36 cells. The matrix is too large to handle with conditionals cleanly.
- State pattern: each state is one class; each operation is one method per state. The compiler enforces exhaustiveness when adding a new operation.

### "What if the same person inserts the card on a different ATM mid-session?"
- Each ATM has its own session — `BankService` doesn't enforce single-session-per-card.
- The bank can detect concurrent sessions and refuse the second auth (a fraud signal). Out of LLD scope.

### "What about 2FA — OTP after PIN?"
- Add an `AwaitingOTPState` between `CardInsertedState` and `AuthenticatedState`.
- Bank service returns "needs OTP" + sends OTP separately; ATM transitions; user enters OTP.

---

## Scaling

A single ATM handles one user; "scaling" here means the network of ATMs:

### Fleet management
- Each ATM is a client of the central banking service.
- Telemetry: cash levels, paper levels, errors, uptime → operations dashboard.
- Predictive cash replenishment (ML) — minimize empty-machine incidents.

### Network resilience
- Some ATMs operate **offline** (in remote areas) — store transactions locally, sync when connectivity returns.
- Offline capability requires per-card daily limits enforced locally by the card itself (chip-card secure storage).

### Anti-fraud
- ATM-network-level fraud detection: flag improbable patterns (cards used in two cities within minutes).
- ML on transaction patterns; freeze cards on anomalies.

### High availability
- Bank's transaction service is a multi-region replicated database.
- ATMs route to the nearest healthy region; failover transparent.

---

## Persistence tradeoffs

Most ATM persistence is **bank-side**, not ATM-side. The ATM is largely stateless between transactions — it persists *minimally* (cash levels, transaction logs for audit, configuration).

### What persistent data exists
- **Bank-side**: accounts, balances, transactions, cards, daily withdrawal counters.
- **ATM-side (small)**: cash dispenser counts, transaction log (idempotency keys + outcomes), configuration.

### Bank-side options

#### Option A — Relational (PostgreSQL / Oracle)
The default for retail banking. Schema:
```sql
accounts(id, customer_id, balance, currency, status, daily_withdrawn, version)
transactions(id, account_id, type, amount, ts, channel, idempotency_key, status)
cards(number, account_id, expiry, status, pin_hash, attempt_count)
```
- **Pros**: ACID, mature operational tooling, transactions natively atomic.
- **Cons**: scaling write throughput on a single account (one row per account is a hotspot if popular). Solved by sharding by `customer_id`.
- **When to pick**: default. Banking ran on Oracle for decades; PostgreSQL is the modern choice.

#### Option B — Event-sourced ledger
Store every credit/debit as an event; balance is the running sum.
```sql
ledger(account_id, txn_id, delta, ts, idempotency_key, ...)
```
- **Pros**: complete audit trail (regulators love this). Every penny accounted for. Reconciliation is sum-of-deltas.
- **Cons**: balance reads require sum (mitigated by snapshot rows or materialized views).
- **When to pick**: regulatory environments mandating immutable history. Many real banks use this internally even when their query API looks like balance lookups.

#### Option C — Distributed strongly consistent (Spanner / CockroachDB / FoundationDB)
- Globally distributed with serializable transactions.
- **Pros**: multi-region with ACID across regions.
- **Cons**: write latency higher than single-region SQL.
- **When to pick**: international banks needing global account access with strong consistency.

### ATM-side options

#### Option A — Embedded SQLite
Local DB for: cash levels, transaction log, configuration. Synchronized commit ensures durability.
- **Pros**: ACID; queryable; well-supported.
- **When to pick**: standard for ATMs.

#### Option B — Append-only journal
Plain file with sequential transaction records. Written synchronously with `fsync`.
- **Pros**: minimal dependency; crash-recoverable.
- **Cons**: less queryable.
- **When to pick**: ultra-resource-constrained machines; usually combined with periodic upload to central system.

### Idempotency store
The bank must persist `(idempotencyKey → outcome)` for at least a day so retries are deduped. Common backings:
- **Redis** with TTL — fast lookup; ephemeral but acceptable for this purpose.
- **DB table** with TTL via batch cleanup.
- **DynamoDB** with TTL — managed expiration.

### Concurrency semantics
- **Pessimistic**: `SELECT ... FOR UPDATE` on the account row. Simple, scales to moderate load.
- **Optimistic**: `UPDATE ... WHERE balance >= ? AND version = ?`. Better throughput; retries on contention.
- **Event-sourced**: append a delta event with idempotency check; balance derived. Concurrent appends are usually OK because the order is the natural order.

### The classic crash-safety question
If the bank debits the account but the ATM crashes before dispensing:
- The next time the ATM boots, it reads its local journal: "I started transaction T, sent debit, never confirmed dispense."
- It calls `bankService.status(idempotencyKey)` — bank returns "completed."
- ATM either dispenses now (if cash is still claimed for that transaction) or marks for operator review.
- This is why the **local journal must be persisted before the bank request**, not after.

The ordering rule: **persist intent, then perform, then persist outcome**. Crash anywhere in that chain → recoverable from the journal.

### Recommendation
- **Bank-side**: PostgreSQL (or distributed SQL at scale). Optionally event-sourced ledger for audit. Idempotency table with TTL.
- **ATM-side**: SQLite for local journal. Sync-commit on every transaction state change.

---

## Talking points for the interview

- "The ATM holds no account state — the bank is the single source of truth. The ATM is a thin presentation + dispense layer."
- "Every bank-touching operation has an idempotency key so retries on network blips don't double-debit."
- "Order matters: debit first, dispense second. If dispense fails, reverse the debit (with the same key family)."
- "State pattern because 6 states × 6 operations is too big for `if (state == X)`. Each state owns its rules."
- "I'm using `synchronized` on the ATM's public API rather than fine-grained locks because the machine is one state machine — only one user."
- "Concurrency for the *account* lives in the bank — that's where two ATMs hitting the same account are serialized."
- "The state machine plus idempotent operations make crash recovery tractable: on boot, the ATM reads its journal, asks the bank about pending operations, and resolves."

---

## Summary

Patterns load-bearing: **State** (ATM lifecycle), **Strategy** (dispensing), **Adapter / Repository-like** (BankService boundary), **Command + idempotency** (transactions).

The mental model: ATM = state machine for the user session; bank = source of truth for accounts; the boundary between them is `BankService`, mediated by idempotent transactions.

The hardest concept here is the failure-mode reasoning: every step in a transaction can fail, and the design must recover correctly without double-spend or lost money. State machine + idempotency + journaling = the answer.
