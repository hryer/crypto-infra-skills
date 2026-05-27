---
name: internal-ledger-design
description: Use when designing or reviewing the internal accounting ledger for a crypto exchange, custodian, wallet product, or any system that holds user funds. Covers double-entry primitives (accounts, journals, transactions, entries), conservation invariants per currency, multi-currency entries without FX inside the ledger, pending/hold semantics, idempotency on inbound events, reversal via compensating entries, append-only storage, derived vs materialized balances, and the split between trade ledger and settlement ledger. Use whenever the user mentions internal ledger, double-entry, balances, user balances, accounting, ledger schema, conservation, idempotent debits/credits, hold accounts, settlement ledger, or "how do we track who owns what".
---

# Internal Ledger Design

## When to use

Trigger this skill when the user asks to:

- Design the user-balance / accounting layer of a CEX, DEX, custodian, or wallet product
- Decide whether single-entry rows or double-entry is appropriate
- Model multi-currency balances (crypto + fiat) without baking FX into the ledger
- Add pending/hold semantics (in-flight deposits, withdrawals, open orders)
- Make a ledger idempotent against retried webhooks or replayed events
- Reverse a credit/debit safely after a chargeback, reorg, or operational error
- Split a single "balances" table into a proper trade ledger + settlement ledger
- Tune balance reads under load (derived from journal vs materialized cache)
- Diagnose why "exchange balance ≠ on-chain balance" (handoff to `settlement-reconciliation`)

If the user is asking about reconciliation against on-chain state, use [[settlement-reconciliation]] instead.
If the user is asking about confirmation thresholds and the deposit/withdrawal pipeline itself, use [[deposit-withdrawal-pipelines]].

## Process

### 1. Decide if you actually need double-entry

Single-entry ("just a balances table that goes up and down") is tempting for v0. It is wrong as soon as you have:

- More than one currency
- Any concept of "pending" vs "available"
- Auditors, accountants, or regulators
- Reversals (chargebacks, reorgs, operational corrections)
- Multiple subsystems writing to balances (deposits, trades, withdrawals, fees)

Double-entry is the default. The only valid "single-entry" cases are throwaway demos and pure read-only views.

The rule: **every event that changes a balance posts a balanced transaction with two or more entries summing to zero per currency**. No exceptions, no "internal-only" debits without a credit somewhere.

### 2. Lock in the primitives

Four objects. Do not invent a fifth.

| Object | Purpose | Mutability |
|---|---|---|
| **Account** | A bucket of funds in one currency, owned by one party (user, exchange treasury, fee pool, hold reserve). | Append-only metadata; balance is derived. |
| **Transaction** | An atomic, balanced event with a correlation_id, posted_at, and a type (`deposit`, `trade_fill`, `withdrawal`, `fee`, `reversal`, etc.). | Immutable once written. |
| **Entry** | One signed amount in one currency against one account, belonging to exactly one transaction. | Immutable. |
| **Journal** | Append-only log of transactions in commit order. | Append-only. |

A canonical Go shape:

```go
type Account struct {
    ID       AccountID   // ULID or UUID
    OwnerID  string      // user ID, "EXCHANGE", "FEE_POOL", "HOT_WALLET_BTC", ...
    Currency Currency    // "BTC", "USDT", "IDR", "SOL"
    Kind     AccountKind // user / liability / asset / revenue / hold
    Status   AccountStatus
}

type Transaction struct {
    ID            TxID
    CorrelationID string    // idempotency key from upstream (e.g. deposit_txhash, order_id+fill_seq)
    Type          TxType
    PostedAt      time.Time // when the ledger committed it (NOT when the source event happened)
    OccurredAt    time.Time // source event time
    Metadata      map[string]string
}

type Entry struct {
    ID            EntryID
    TransactionID TxID
    AccountID     AccountID
    Currency      Currency
    Amount        decimal.Decimal // signed; sum-per-currency-per-tx MUST equal 0
    Direction     Direction       // DEBIT / CREDIT — derived from sign, but stored for index access
}
```

**Non-negotiables:**

- `Amount` is `decimal.Decimal` (shopspring or equivalent). Never `float64`. Never `int` without a fixed scale baked into the currency definition (and even then, prefer Decimal).
- `Currency` is a typed value, not a string passed around. A function that takes `(amount, currency)` is a bug waiting to happen — pass `Money{amount, currency}`.
- Every entry has a currency; the transaction itself does not. A trade `BTC/USDT` produces entries in both BTC and USDT, and each currency must independently sum to zero across the transaction.

### 3. The conservation invariant

The one rule that makes a ledger a ledger:

> **For every transaction T and every currency C: sum of entries(T, C).Amount = 0.**

If you do not enforce this at write time (DB constraint, server validation, or both), you will silently leak balance during incidents. Enforce it twice:

- **At the application layer** before INSERT: validate the entry set sums to zero per currency.
- **At the database layer** with a deferred constraint, a trigger, or a stored procedure that rejects unbalanced commits.

Postgres example:

```sql
CREATE OR REPLACE FUNCTION enforce_balanced_tx() RETURNS TRIGGER AS $$
BEGIN
  IF EXISTS (
    SELECT 1 FROM entries
    WHERE transaction_id = NEW.transaction_id
    GROUP BY currency
    HAVING SUM(amount) <> 0
  ) THEN
    RAISE EXCEPTION 'transaction % is unbalanced', NEW.transaction_id;
  END IF;
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE CONSTRAINT TRIGGER tx_balanced
  AFTER INSERT ON entries
  DEFERRABLE INITIALLY DEFERRED
  FOR EACH ROW EXECUTE FUNCTION enforce_balanced_tx();
```

Deferred is important: you insert all entries inside one DB transaction, the constraint fires at COMMIT, not after each row.

### 4. Multi-currency: never convert inside the ledger

A trade `0.1 BTC for 6,500 USDT` is **not** a single-currency move. It is two simultaneous transfers, recorded as four entries in two currencies:

```
TX trade_fill_42:
  DEBIT  user_alice.BTC   -0.1 BTC      -- alice loses BTC
  CREDIT user_bob.BTC     +0.1 BTC      -- bob gains BTC
  DEBIT  user_bob.USDT   -6500 USDT     -- bob loses USDT
  CREDIT user_alice.USDT +6500 USDT     -- alice gains USDT

per-currency sum: BTC = 0, USDT = 0   ✓
```

**Do not** write entries like `user_alice: -0.1 BTC, +6500 USDT` in the same currency column with a "price" field. The ledger stores movements of value, not opinions about its USD-equivalent worth. Mark-to-market belongs in a downstream P&L view, never inside the ledger.

Rationalizations against this rule are universal and always wrong. See the rejection table below.

### 5. Pending vs available: hold accounts, not "available_balance" columns

Two common cases:

- A user places a buy order for 100 USDT. That 100 USDT is committed but not spent yet.
- A user requests a withdrawal of 0.5 BTC. The 0.5 BTC must not be spendable while the withdrawal is processing, even if it has not yet hit the chain.

The wrong solution is a second column on the balance ("available" and "locked"). It rots: every code path that touches balances has to remember to update both, and reconciliation has no source of truth for what "locked" means.

The right solution is a **hold account per user per currency**, modeled as a normal account:

```
TX open_order_99 (alice buys at 65000):
  DEBIT  user_alice.USDT     -6500 USDT
  CREDIT user_alice_hold.USDT +6500 USDT

per-currency sum: USDT = 0   ✓
```

The user's available balance is `user_alice.USDT`. Their total balance is `user_alice.USDT + user_alice_hold.USDT`. There is no special "available_balance" code path — it's just a sum over normal entries.

When the order fills, you post compensating entries to release the hold and a settlement transaction for the trade. When the order cancels, you post the reverse hold-release alone.

### 6. Idempotency: every transaction needs a correlation_id

Inbound events repeat. Webhooks retry. Consumers re-deliver. Operators rerun jobs after a partial failure. Without idempotency, the ledger double-credits.

**Rule:** every transaction has a `correlation_id` unique per source event, with a UNIQUE constraint at the database level. On collision, the second insert MUST fail and the application MUST treat that failure as "already done" rather than an error.

Examples of correlation_id construction:

| Source | correlation_id |
|---|---|
| Bitcoin deposit | `btc_deposit:<txid>:<vout>` |
| EVM ERC-20 deposit | `evm_deposit:<chain_id>:<txhash>:<log_index>` |
| Solana SPL deposit | `sol_deposit:<signature>:<instruction_index>` |
| Trade fill | `fill:<trade_id>` |
| Withdrawal broadcast | `withdrawal:<withdrawal_id>:broadcast` |
| Withdrawal confirmation | `withdrawal:<withdrawal_id>:confirmed` |
| Reorg reversal | `reversal:<original_tx_id>` |

Two things to notice:

- The chain identifier is part of the key on multi-chain systems. `0xabc` on Ethereum and `0xabc` on Polygon are different events.
- The log_index / vout / instruction_index is part of the key because a single chain transaction can credit multiple users in different logs.

The application MUST NOT generate this server-side from `time.Now()` or a UUID. The correlation_id MUST be deterministic from the source event so retries collapse.

### 7. Reversal: compensating entries, never DELETE

Bad things happen. A 2-confirmation deposit gets reorged out. An operator credits the wrong account. A trade settlement fires twice somehow. The ledger never deletes and never updates entries. It posts a reversing transaction whose `metadata.reverses_tx_id = <original>`.

```
TX deposit_reversed_for_reorg:
  Type = "reversal"
  Metadata: { reverses_tx_id: "deposit_btc_42", reason: "5-block reorg" }
  DEBIT  user_alice.BTC      -0.1 BTC   -- pull back the credit
  CREDIT exchange_pending.BTC +0.1 BTC  -- return to the pending pool
```

The original transaction is still there. Audit can see what happened and when. Reconciliation can match originals and reversals to confirm conservation.

If you need to "fix" a balance via a one-sided adjustment, you don't have a ledger — you have a spreadsheet.

### 8. Storage shape: append-only journal, balances derived

The journal (entries table) is the source of truth. Balances are derived by `SUM(amount) WHERE account_id = ?`.

At scale that is slow. Two production-proven options:

| Approach | When to use | Trade-off |
|---|---|---|
| **Materialized balance row** updated inside the same DB tx as the entries | Default. Reads are O(1), conservation is checked once per tx. | Need to make the update transactional with the entry insert — use `SELECT ... FOR UPDATE` on the balance row. |
| **Periodic balance snapshot** (e.g. nightly) + sum entries since the snapshot | Cold/archival accounts; very high entry volume where row-level locking on hot accounts is the bottleneck. | Slightly stale balances; reconciliation has to be aware of snapshot cutoffs. |

Avoid: caching balances in Redis without a transactional path to the journal. The cache and the journal will diverge, and reconciliation will not be able to tell which one is right.

Sketch of the materialized pattern in Postgres + Go:

```go
func PostTransaction(ctx context.Context, db *sql.DB, tx Transaction, entries []Entry) error {
    return WithSerializableTx(ctx, db, func(q *sql.Tx) error {
        if err := insertTransaction(q, tx); err != nil {
            // UNIQUE violation on correlation_id => already posted, treat as success
            if isUniqueViolation(err, "transactions_correlation_id_key") {
                return nil
            }
            return err
        }
        for _, e := range entries {
            if err := insertEntry(q, e); err != nil {
                return err
            }
            if err := upsertBalanceRow(q, e.AccountID, e.Currency, e.Amount); err != nil {
                return err
            }
        }
        // Conservation trigger fires at COMMIT (deferred).
        return nil
    })
}
```

Isolation level matters. `SERIALIZABLE` or `REPEATABLE READ` with `SELECT ... FOR UPDATE` on the balance row prevents lost updates. `READ COMMITTED` will silently corrupt balances under contention.

### 9. Two ledgers, not one: trade ledger vs settlement ledger

This is the most common architectural mistake on first attempt.

- **Trade ledger** ("the matching engine's view"): records what trades happened, at what price, with what fees. Entries credit and debit user accounts inside the exchange.
- **Settlement ledger** ("the treasury's view"): records movements of actual on-chain or fiat-rail value. Entries credit and debit exchange-owned wallet/bank accounts.

A single user deposit of 1 BTC flows through both:

```
SETTLEMENT TX deposit_btc_42:
  DEBIT  hot_wallet_btc       -1 BTC
  CREDIT exchange_liability_btc +1 BTC

TRADE TX user_credit_for_deposit_42:
  DEBIT  exchange_liability_btc -1 BTC
  CREDIT user_alice_btc         +1 BTC
```

Two separate balanced transactions linked by a shared correlation chain. The exchange's on-chain holdings grow by 1 BTC; the exchange's liability to users grows by 1 BTC; user Alice's balance grows by 1 BTC. The fundamental accounting equation (assets = liabilities + equity) holds at every step.

Conflating them — writing a single `CREDIT user_alice.BTC` against `DEBIT hot_wallet_btc` — breaks audit, breaks reconciliation, and makes the exchange's actual financial position invisible.

### 10. What goes in `metadata` (and what doesn't)

`metadata` is for facts that are useful but not part of the conservation arithmetic. Good candidates:

- `source_chain`, `source_txhash`, `source_log_index`
- `order_id`, `trade_id`, `fill_seq`
- `operator_id` for manual adjustments
- `reverses_tx_id` for reversals
- `compliance_flags` (sanctions_screening_passed, kyc_tier)

Do NOT put inside metadata:

- Amounts that affect balances. Those are entries.
- Prices, FX rates, USD-equivalents. Those belong in a pricing service or P&L view.
- Free-form user comments. The ledger is not a chat log.

## Rationalizations to reject

| Rationalization | Why it's wrong |
|---|---|
| "We're small, single-entry is fine for now." | The migration from single to double-entry is one of the worst migrations in fintech because there is no historical pair-of-entries to reconstruct. Start double-entry on day 1. |
| "We'll store available_balance and locked_balance as columns and update them transactionally." | Every code path that touches balances must remember to update both. There is no source of truth for what "locked" means. Hold accounts cost the same to implement and never rot. |
| "Float64 is fine, our amounts are small." | `0.1 + 0.2 != 0.3` in IEEE 754. The cents you lose to representation error will not show up in tests; they will show up in reconciliation differences and you will spend a week finding them. |
| "We'll record the trade as one entry: alice's BTC went down, USDT went up, with a `price` field." | The ledger is now opinionated about FX. When BTC/USDT moves between when you wrote the entry and when audit runs, the ledger does not match reality. Two-currency trades are four entries. |
| "Reorgs are rare, we can just UPDATE the entry to reverse it." | Audit trail destroyed. Replaying the journal no longer produces the same balance. Reversal is a new transaction with a new correlation_id, full stop. |
| "We'll cache balances in Redis and write-through to Postgres." | Cache and DB diverge under partial failure. When they disagree, you cannot tell which is right. If you must cache, derive the cache from the journal asynchronously and treat Postgres as the source of truth. |
| "Idempotency at the application layer is enough — we have a check before insert." | TOCTOU race between check and insert under retry. Idempotency must be enforced by a UNIQUE constraint that the database refuses to violate. |
| "We'll generate correlation_id server-side as a UUID." | Retries no longer collapse. Every retry inserts a new transaction. You will double-credit deposits the first time the webhook layer hiccups. correlation_id is deterministic from the source event. |
| "We'll use READ COMMITTED for performance." | Under contention on the same balance row, two concurrent transactions both read the old balance, both compute new balances, both commit. One update is lost. Use SERIALIZABLE or `FOR UPDATE`. |
| "The trade ledger and the settlement ledger are the same thing, no need to split." | You can no longer answer "what does the exchange owe users vs what does it actually hold on-chain". Reconciliation between trading and treasury becomes impossible. Split them on day 1. |
| "We'll just DELETE accounts that are no longer used." | Entries referencing the account become orphaned. Audit and replay break. Accounts have a status (`active`, `frozen`, `closed`), never a DELETE. |

## Verification

Before this design is considered done:

- [ ] Every entry has a typed `Currency` and a `decimal.Decimal` `Amount`. No `float64` anywhere in the money path. Grep for `float64` in the ledger packages and confirm zero hits.
- [ ] A unit test posts a deliberately unbalanced transaction (e.g. 2 entries in same currency summing to non-zero) and the write fails — both at the application layer AND at the DB constraint layer. Remove one of the two enforcement points and confirm the other still catches it.
- [ ] A unit test replays the same source event twice (same correlation_id) and the second call is a no-op. Run the test 1000× concurrently; balances do not change beyond the first credit.
- [ ] Property test: post N random valid transactions across M accounts in K currencies; assert `SUM(entries.amount) GROUP BY currency = 0` over the whole journal.
- [ ] Pending semantics implemented via hold accounts, not balance columns. `available_balance` is computed as `SUM(entries) WHERE account.kind = 'user'`. No code path bypasses this.
- [ ] Reversal flow demonstrated end-to-end: deposit → reorg detected → compensating transaction posted → user balance returns to pre-deposit state → original deposit row still exists in the journal with its original timestamp.
- [ ] Trade ledger and settlement ledger are distinct DB tables (or at least distinct transaction types with non-overlapping account kinds). A nightly job confirms `SUM(exchange_liability_<currency>) == SUM(user_balances_<currency>)` for every currency.
- [ ] Conservation alert wired to monitoring: any time the per-currency journal sum is non-zero (which should be never), page on-call. This is your last line of defense — if it ever fires you have a bug or a tampered DB.
- [ ] Withdrawal and deposit pipelines write to the ledger using deterministic correlation_ids derived from `(chain, txhash, log_index/vout)`. Replaying the indexer from genesis produces the same final balances.
- [ ] Hot path (single-account credit/debit under load) benchmarked: balance read p99 latency known, balance write p99 latency known, contention on a single popular account characterized.
- [ ] Runbook exists for "ledger and treasury disagree" — handoff to [[settlement-reconciliation]].
