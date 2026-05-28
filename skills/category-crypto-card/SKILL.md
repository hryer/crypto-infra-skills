---
name: category-crypto-card
description: Use when designing or reviewing a crypto debit/credit card product. Covers card issuer integration (Marqeta, Stripe Issuing, GPS, Galileo), funding flow (auth → settle), ledger architecture for crypto-to-fiat conversion, KYC/AML compliance, BIN sponsorship, FX and crypto pricing, and chargeback handling. Use whenever the user mentions crypto card, debit card, card issuing, authorization flow, settlement, BIN sponsor, or interchange.
---

# Crypto Card Product Architecture

## When to use

Trigger this skill when the user asks to:
- Design a crypto debit/credit card from scratch
- Pick a card issuer (Marqeta, Stripe Issuing, Galileo, GPS, Lithic)
- Design the auth-time funding flow (crypto → fiat)
- Architect the ledger for crypto card transactions
- Handle KYC/AML for a card product
- Deal with chargebacks, disputes, or settlement timing
- Compete with Crypto.com, Bybit Card, Coinbase Card, Kast, etc.

## The mental model

A crypto card is NOT a "card that spends crypto." Visa/Mastercard rails are fiat-only.

What actually happens:
1. User taps card → merchant requests authorization → issuer (your product or your processor) must respond in <2 seconds
2. Your system decides: do I have enough user crypto to cover this fiat amount at current price?
3. If yes: approve auth, lock crypto (don't sell yet)
4. Hours/days later: settlement comes in (the real amount, possibly different due to tips, FX)
5. Now you sell the crypto to cover the fiat settlement
6. Net into the user's account; any leftover is returned

The card is a fiat card. Crypto is just the funding source.

## Process

### 1. Choose the issuing model

**Direct issuer (rare, very hard):**
- You become a Visa/Mastercard principal member
- 18-24 months, multi-million-dollar capital requirement
- Only worth it at massive scale

**Issuer-processor (most common):**
- Marqeta, Galileo, GPS, Lithic, Stripe Issuing, Highnote
- They handle scheme connections and card lifecycle
- You handle authorization decisions and program design
- BIN sponsorship via their bank partner

**Sponsored program (fastest to launch):**
- Use a "card-as-a-service" provider that wraps issuer-processor + BIN sponsor + program management
- Examples: Swap (in some regions), Lithic in non-US, Stripe Issuing in supported countries
- Trade-off: less control, higher per-tx fees

**Pick based on:** speed-to-market vs unit economics. Early-stage = sponsored. At scale = direct issuer-processor.

### 2. Map the lifecycle states

```
Authorization
├── Approved → Captured (funds locked, awaiting clearing)
│   ├── Cleared (settlement matches auth) → Settled
│   ├── Cleared with adjustment (e.g., tip) → Settled (different amount)
│   └── Reversed (auth expired without settle) → Released
└── Declined (any reason)

Settled transactions
├── Refund (merchant credit) → Refunded
├── Chargeback initiated → Disputed
│   ├── Won → Resolved
│   └── Lost → Charged back (you take the loss or pass to user)
```

Your ledger must model all of these. Don't conflate auth with settlement — they're different objects with different timing.

### 3. Authorization-time funding decision

The hot path. Sub-2-second decision required.

```
On authorization webhook:
1. Validate the auth (card, MCC, merchant, amount, currency)
2. Look up user's crypto balance + price feed
3. Compute required crypto = fiat_amount * (1 + price_buffer) / crypto_price
4. Apply user spending limits (per-tx, per-day, per-month)
5. Apply program rules (blocked MCCs, blocked merchants, velocity)
6. If sufficient + within limits: lock the crypto + approve
7. If not: decline with appropriate reason code
8. Return decision in <500ms (provider timeout is 1-2s, leave margin)
```

**Price buffer:** crypto moves fast. Between auth and settle (can be days), price might drop 5%. Buffer protects against under-collateralization. Typical: 10-20% for volatile assets, 1-2% for stables.

**Velocity rules:** abusive patterns include rapid micro-transactions and merchant-induced double-auth. Have rate limits at user and card level.

### 4. Ledger architecture

Double-entry, append-only, with multi-currency:

```
TransactionEntry
├── id, timestamp, type, status
├── debit_account_id
├── debit_amount + debit_currency
├── credit_account_id
├── credit_amount + credit_currency
├── exchange_rate (if cross-currency)
└── correlation_id (links auth → capture → settle → refund)
```

**Account types:**
- User crypto wallet (e.g., user_btc_wallet)
- User fiat ledger (e.g., user_usd_ledger)
- Card pending account (locked but not settled)
- Card settled account
- Fee accounts (your revenue)
- FX clearing account
- Crypto liquidity account (where you sell user crypto into)

**The auth flow ledger entries:**
```
Auth approved:
  DEBIT  user_btc_wallet         (locked crypto)
  CREDIT card_pending_btc        (now locked)
  + record price snapshot + fiat amount for reconciliation
```

```
Settlement received (fiat amount):
  Resolve the lock + execute the conversion:
  
  DEBIT  card_pending_btc        (release lock)
  CREDIT crypto_liquidity_btc    (we now own the crypto)
  
  DEBIT  fiat_settlement_account (we owe USD)
  CREDIT user_fiat_ledger        (paid for purchase)
  
  Plus a separate sale transaction at current price to cover fiat:
  DEBIT  crypto_liquidity_btc
  CREDIT fiat_treasury (sold for USD at execution price)
```

The reconciliation between auth-time price and settle-time price is where you profit or lose on FX/crypto movement.

### 5. KYC/AML

Card programs have specific KYC requirements that exceed typical exchange KYC:
- Identity verification (typically Tier 1)
- Address verification with proof of address document
- Sometimes enhanced due diligence (EDD) for high-volume users
- Sanctions screening: real-time at every authorization, not just at onboarding
- Transaction monitoring: SAR-eligible patterns (structuring, geographic risk)

**Provider integration:** Persona, Sumsub, Onfido for the identity side. Sanctions via ComplyAdvantage, ChainAnalysis (for crypto-side), or your BIN sponsor's mandated tool.

**Critical:** sanctions screening at auth time must add <50ms to the auth decision. Pre-compute risk scores, cache aggressively, hit a real-time service only for new merchants.

### 6. FX and crypto pricing

Three rates involved in any cross-currency transaction:
- **Quoted rate** — shown to user at auth time
- **Execution rate** — actual rate when you sell crypto to settle
- **Settlement rate** — the rate the scheme uses for cross-border settlement

Your P&L on the card includes interchange (revenue), scheme fees (cost), processing fees (cost), and the spread you charge on FX/crypto conversion.

**Don't undercharge the spread.** Major card products charge 2-5% on FX/crypto spread. Most of your unit economics live here.

### 7. Chargebacks and disputes

Crypto card chargebacks are operationally hard because:
- You sold the crypto at settlement to cover the fiat
- A chargeback months later means you need to RE-buy crypto at current (possibly higher) price to restore user balance
- Or you keep the dispute funds in fiat and let user re-buy crypto

**Policy decisions to make upfront:**
- Do you restore in crypto (you eat the price risk) or in fiat (user eats it)?
- What's your dispute response SLA? (Schemes typically allow 7-30 days)
- Do you have a dispute portal for users or is it CS-handled?

### 8. Reconciliation

Daily reconciliation is mandatory:
- Auth vs settlement counts and amounts (every auth should resolve to settle, reverse, or expire)
- Internal ledger balances vs treasury wallet balances
- Card issuer's reported balance vs your computed balance
- FX positions

Discrepancies > $0.01 should trigger investigation, not be written off.

## Rationalizations to reject

| Excuse | Reality |
|---|---|
| "We'll integrate KYC at launch" | KYC is auth flow infrastructure, not a feature. Integrate from day one. |
| "Let users hold the crypto, we just sell at swipe" | Custody is the entire product. If you don't hold custody, you're a thin wrapper on someone else's card. |
| "We don't need a price buffer, crypto is liquid" | Crypto is volatile. A 10% buffer protects you from a flash crash between auth and settle. |
| "Auth and settle are basically the same thing" | They are not. Different amounts, different timing, different reversal logic. Model them separately. |
| "We can save on issuer fees by using a cheaper BIN" | Premium BINs get higher interchange. Cheap BIN = cheap interchange = thin margins. |
| "Refunds are just negative authorizations" | Refunds are independent settlement events. Some come without a prior auth. Handle them as first-class. |

## Verification

Before launch:
- [ ] End-to-end test: auth → settle → user balance update, with idempotency
- [ ] End-to-end test: auth → reversal (no settle) → released funds
- [ ] End-to-end test: refund without prior auth
- [ ] End-to-end test: chargeback initiated, won, lost
- [ ] FX/crypto rate snapshot logged on every auth (for reconciliation)
- [ ] Decline reason codes mapped to user-facing messages
- [ ] Velocity limits enforced (per-tx, per-day, per-month, per-merchant-class)
- [ ] Sanctions screening at auth time, <50ms p99
- [ ] Daily reconciliation script + alerting on discrepancies
- [ ] Dispute workflow runbook + chargeback response template

## Competitive landscape (for product positioning)

| Product | Custody model | Settlement | Differentiator |
|---|---|---|---|
| **Crypto.com Card** | Custodial | Real-time crypto → fiat | CRO staking rebates |
| **Coinbase Card** | Custodial | Real-time | Fiat-backed with crypto cashback |
| **Bybit Card** | Custodial | Real-time | High limits, strong Asia presence |
| **Bitpanda Card** | Custodial | Real-time | EU focus, regulated |
| **Kast** | Custodial, stablecoin-first | USDC settlement | Stablecoin native, no FX games |
| **Plutus** | Custodial | Real-time | PLU rewards token |

**Strategic implication:** "another crypto card" is hard. Win on geography (underserved region), asset (stablecoins-only, or a specific chain), or program economics (rebates, no FX spread).

## References

- `references/card-issuer-integrations.md` — vendor comparison (Marqeta, Stripe Issuing, etc.)
- `references/ledger-architecture.md` — double-entry ledger patterns for crypto cards
