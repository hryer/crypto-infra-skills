# Ledger Architecture for Crypto Cards

Double-entry, append-only, multi-currency ledger patterns specifically for crypto card products.

## Why double-entry

- **Conservation:** every value movement has a source and a destination. Total system value never appears or disappears.
- **Auditability:** every entry has a paired counter-entry. Discrepancies become trivially detectable.
- **Reversibility:** to reverse a transaction, post inverse entries. Original entries are never modified.

If you've been using single-entry "decrement balance, increment balance" patterns, switching to double-entry is the single biggest improvement you can make to a financial backend.

## Core data model

```sql
-- An account is just a name + a currency + an owner
CREATE TABLE accounts (
    id              UUID PRIMARY KEY,
    owner_type      VARCHAR(32) NOT NULL,  -- 'user', 'system', 'fee', etc.
    owner_id        UUID,                  -- nullable for system accounts
    account_type    VARCHAR(64) NOT NULL,  -- 'user_crypto_btc', 'card_pending', 'fees', etc.
    currency        VARCHAR(16) NOT NULL,  -- 'BTC', 'USD', 'USDC', etc.
    created_at      TIMESTAMPTZ NOT NULL,
    metadata        JSONB
);

CREATE UNIQUE INDEX ON accounts (owner_type, owner_id, account_type, currency);

-- An entry is one side of a transaction
CREATE TABLE entries (
    id                  UUID PRIMARY KEY,
    transaction_id      UUID NOT NULL,
    account_id          UUID NOT NULL REFERENCES accounts(id),
    direction           CHAR(1) NOT NULL,  -- 'D' for debit, 'C' for credit
    amount              NUMERIC(38, 18) NOT NULL CHECK (amount > 0),
    currency            VARCHAR(16) NOT NULL,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX ON entries (transaction_id);
CREATE INDEX ON entries (account_id, created_at);

-- A transaction is a group of entries that must sum to zero (per currency)
CREATE TABLE transactions (
    id                  UUID PRIMARY KEY,
    type                VARCHAR(64) NOT NULL,
    correlation_id      UUID,               -- links related txs (auth ↔ settle)
    external_id         VARCHAR(128),       -- e.g., card auth id, blockchain tx hash
    status              VARCHAR(32) NOT NULL,
    posted_at           TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    metadata            JSONB
);

CREATE UNIQUE INDEX ON transactions (external_id) WHERE external_id IS NOT NULL;
CREATE INDEX ON transactions (correlation_id);
```

## The conservation invariant

For every transaction, per currency, sum of debits = sum of credits.

Enforce in a check, not just in code:

```sql
CREATE OR REPLACE FUNCTION check_transaction_balance() RETURNS TRIGGER AS $$
DECLARE
    imbalance NUMERIC;
BEGIN
    SELECT SUM(CASE direction WHEN 'D' THEN amount ELSE -amount END)
    INTO imbalance
    FROM entries
    WHERE transaction_id = NEW.transaction_id
      AND currency = NEW.currency;

    -- Allow during in-progress insert; check at end of transaction
    -- Actual enforcement: do this as a deferred constraint or in application logic
    -- shown here for illustration
    
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;
```

Practical approach: enforce in application code, verify with a periodic invariant check that scans recent transactions.

## Patterns for crypto cards

### Pattern 1: Card authorization (funds lock)

User has 0.1 BTC. Approves a $50 auth at BTC = $50,000.

```
Transaction: auth_approved (correlation_id = auth_xyz)

Entry 1: DEBIT  user_btc_wallet         0.001 BTC
Entry 2: CREDIT card_pending_btc        0.001 BTC

(Plus metadata: price_snapshot = 50000 USD/BTC, fiat_target = 50.00 USD)
```

User's spendable BTC is `available = balance - pending`. The pending amount is locked but still ledger-tracked.

### Pattern 2: Settlement (auth → captured)

Card scheme reports settlement for $52 (tip added).

```
Transaction: auth_settled (correlation_id = auth_xyz)

# Step 1: release the pending lock
Entry 1: DEBIT  card_pending_btc        0.001 BTC
Entry 2: CREDIT user_btc_wallet         0.001 BTC

# Step 2: sell user's BTC for the settlement fiat amount
# Suppose execution rate is now 49000 USD/BTC, so we need more BTC
Required BTC = 52.00 / 49000 = 0.0010612 BTC

Entry 3: DEBIT  user_btc_wallet         0.0010612 BTC
Entry 4: CREDIT crypto_liquidity_btc    0.0010612 BTC

# Step 3: fiat side - convert that BTC to USD via your liquidity provider
Entry 5: DEBIT  crypto_liquidity_btc    -- (already credited above; same account, net)
Entry 6: CREDIT fiat_treasury_usd       52.00 USD  (just received)

# Step 4: pay the settlement
Entry 7: DEBIT  fiat_treasury_usd       52.00 USD
Entry 8: CREDIT card_scheme_payable     52.00 USD
```

Note: steps 3-4-5-6 are conceptually two transactions (a crypto sale and a fiat payout). You can model them as one or two — two is cleaner for audit.

### Pattern 3: Reversal (auth never settled)

Auth expires after 7 days without a settlement.

```
Transaction: auth_reversed (correlation_id = auth_xyz)

Entry 1: DEBIT  card_pending_btc        0.001 BTC
Entry 2: CREDIT user_btc_wallet         0.001 BTC
```

Just reverses the lock. No fiat conversion happened.

### Pattern 4: Refund

Customer returns purchase, merchant refunds $52.

```
Transaction: refund_received (correlation_id = refund_abc, related: auth_xyz)

Entry 1: DEBIT  card_scheme_receivable  52.00 USD
Entry 2: CREDIT user_fiat_pending_usd   52.00 USD

(Then, asynchronously, you convert that fiat back to BTC and credit the user wallet)

# Conversion (separate transaction)
Entry 3: DEBIT  user_fiat_pending_usd   52.00 USD
Entry 4: CREDIT fiat_treasury_usd       52.00 USD

Entry 5: DEBIT  crypto_treasury_btc     0.0010612 BTC
Entry 6: CREDIT user_btc_wallet         0.0010612 BTC
```

The refund executes at TODAY'S rate, not the original purchase rate. User absorbs FX/crypto price movement (unless your product chooses to make this whole — a P&L decision).

### Pattern 5: Chargeback (lost dispute)

```
Transaction: chargeback_lost (correlation_id = chargeback_def)

# You eat the loss to operating expense
Entry 1: DEBIT  losses_chargebacks_usd  52.00 USD
Entry 2: CREDIT card_scheme_payable     52.00 USD
```

User keeps their original purchase value (you don't claw it back).

If your policy is to claw back from the user:
```
Entry 1: DEBIT  user_fiat_ledger_usd    52.00 USD  (negative balance allowed?)
Entry 2: CREDIT card_scheme_payable     52.00 USD
```

This is a customer-experience landmine. Most card products eat small chargebacks and only claw back at high amounts.

## Multi-currency considerations

A user has accounts in multiple currencies:
- `user_btc_wallet` (currency: BTC)
- `user_eth_wallet` (currency: ETH)
- `user_usdc_wallet` (currency: USDC)
- `user_fiat_ledger_usd` (currency: USD)

The conservation invariant is **per-currency, per-transaction**. A transaction that converts BTC to USD has:
- Net BTC change: 0 (debits = credits in BTC)
- Net USD change: 0 (debits = credits in USD)

This is satisfied by routing through intermediate accounts (liquidity, treasury) that carry the cross-currency exposure.

## Available balance vs ledger balance

Two distinct concepts:
- **Ledger balance** = sum of all entries on the account
- **Available balance** = ledger balance - pending holds - reserves

User-facing displays should show available balance, not raw ledger balance.

```sql
-- Available balance for a user's BTC account
SELECT
  COALESCE(SUM(CASE direction WHEN 'C' THEN amount ELSE -amount END), 0) AS ledger_balance,
  COALESCE(
    (SELECT SUM(amount) FROM entries
     WHERE account_id = ? AND direction = 'C'
       AND transaction_id IN (
         SELECT id FROM transactions WHERE status = 'pending'
       )), 0
  ) AS pending_holds
FROM entries
WHERE account_id = ?;
```

## Idempotency in posting

Every transaction must be idempotent on `external_id` (e.g., the card auth ID, or the blockchain tx hash). UNIQUE constraint enforces it at the DB.

```go
func (l *Ledger) PostAuthApproved(authID string, userID UUID, cryptoAmount Decimal) error {
    return l.db.Transaction(func(tx *DB) error {
        // Check idempotency
        var existing Transaction
        if err := tx.Where("external_id = ?", authID).First(&existing).Error; err == nil {
            return ErrAlreadyProcessed
        }

        txID := uuid.New()
        // Insert transaction + entries atomically
        // ...
    })
}
```

## Reconciliation

Daily, run:
- For each account, sum of all entries should equal the externally reported balance (if any)
- For each currency, sum of all credits across all accounts = sum of all debits (system-wide conservation)
- For each correlation_id, the related transactions should be in a consistent state (auth → settle, or auth → reverse, never auth without resolution)

Discrepancies > epsilon = high-severity alert. Don't write off.

## What to never do

- **Never UPDATE entries.** Entries are immutable. To "fix" an entry, post a reversing entry plus a corrective entry, both in a new transaction.
- **Never DELETE entries.** Same reason.
- **Never store floats.** Always `NUMERIC(38, 18)` or equivalent. Float math will eventually cost you a penny that compounds into a chargeback.
- **Never store amounts as strings.** Use a proper decimal type. Strings can't be summed in SQL safely.
- **Never skip the transactions table.** "Just entries" is tempting but loses correlation_id and external_id idempotency.
- **Never write to the ledger from multiple services.** One ledger service, all writes go through it. Read replicas elsewhere are fine.
