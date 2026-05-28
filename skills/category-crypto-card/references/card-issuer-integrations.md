# Card Issuer Integrations — Vendor Reference

Comparison of card issuing platforms from a crypto-program-builder's perspective.

## Tier 1: Major issuer-processors

### Marqeta
- **Strengths:** mature API, big customer list (Cash App, Affirm), strong webhooks, JIT funding model is well-supported
- **Weaknesses:** US-heavy, slow approvals for crypto programs, expensive per-tx
- **Crypto-friendliness:** medium; has worked with crypto programs but underwriting is conservative
- **Best for:** US-based programs with capital and a long runway

### Galileo (SoFi-owned)
- **Strengths:** big in fintech (Chime, Robinhood), Latin America presence, established processing
- **Weaknesses:** older platform, less modern API ergonomics
- **Crypto-friendliness:** medium; some crypto programs but not their focus
- **Best for:** programs that want Latin America coverage

### GPS (Global Processing Services)
- **Strengths:** European, multi-currency, strong for cross-border programs
- **Weaknesses:** API less polished than US peers
- **Crypto-friendliness:** medium-high; Curve and other crypto-adjacent products use them
- **Best for:** EU-first or multi-region programs

### Stripe Issuing
- **Strengths:** developer experience, fast onboarding, integrated with Stripe ecosystem
- **Weaknesses:** US/UK only for most features, less control over BIN selection
- **Crypto-friendliness:** low-medium; Stripe is restrictive about crypto, but allows certain programs
- **Best for:** consumer-facing programs already on Stripe

### Lithic
- **Strengths:** developer-first, simple API, virtual cards strong, modern stack
- **Weaknesses:** smaller volume players, fewer enterprise features
- **Crypto-friendliness:** medium; allows crypto programs with proper underwriting
- **Best for:** startups, virtual-card-first products

### Highnote
- **Strengths:** new-gen issuer-processor, modern API, growing customer list
- **Weaknesses:** less battle-tested than Marqeta
- **Crypto-friendliness:** medium; case-by-case
- **Best for:** modern programs willing to bet on a newer platform

## Tier 2: Card-as-a-service (faster but less control)

### Swap (Latin America)
- All-in-one BaaS in Brazil/LatAm. Wraps issuer + BIN sponsor + KYC.
- Crypto programs supported case-by-case.

### Stably (some regions)
- USDC-focused card programs.

### Rain (Latin America)
- Crypto-native card platform.

## Authorization webhook patterns

All major issuers send an HTTPS POST with the auth request and expect your response within a tight window:

| Issuer | Webhook timeout | Notes |
|---|---|---|
| Marqeta | 2000ms | Pre-decisioning required for crypto/JIT funding |
| Galileo | 1500ms | "Real-Time Decisioning" feature |
| GPS | 2000ms | "External Host Authorization" |
| Stripe Issuing | 2000ms | "Authorization request" webhook |
| Lithic | 1000ms | "Transaction.Decisioning" webhook |

**Critical:** your decision system must respond well within these. Aim for p99 < 500ms to leave safety margin.

## Webhook payload shape (representative)

```json
{
  "id": "auth_evt_1234",
  "type": "authorization.request",
  "card": {
    "id": "card_abc",
    "last4": "4242",
    "user_id": "user_xyz"
  },
  "amount": {
    "value": 1500,
    "currency": "USD"
  },
  "merchant": {
    "name": "STARBUCKS #5512",
    "category": "5814",  // MCC
    "country": "US"
  },
  "pos": {
    "entry_mode": "chip",
    "is_recurring": false
  }
}
```

You respond with `{"approved": true/false, "decline_reason": "..."}` or by setting headers.

## Settlement / clearing webhook

A separate event, typically hours to days later:

```json
{
  "id": "settle_evt_5678",
  "type": "transaction.cleared",
  "authorization_id": "auth_evt_1234",
  "amount": {
    "value": 1530,  // tip added
    "currency": "USD"
  },
  "scheme_fee": 12,
  "interchange": 25  // your revenue from interchange
}
```

The amount can differ from auth (tips, partial captures, FX adjustments).

## Decline reason codes (Visa/Mastercard standard)

Important ones:
- `05` — Do not honor (generic; use sparingly)
- `14` — Invalid card
- `41` — Lost card
- `43` — Stolen card
- `51` — Insufficient funds
- `54` — Expired card
- `57` — Transaction not permitted to cardholder
- `61` — Exceeds withdrawal limit
- `65` — Exceeds withdrawal frequency
- `R0` / `R1` — Customer revoked authorization (for recurring)
- `91` — Issuer unavailable (avoid; this looks like an outage)

Map your business decline reasons to these codes correctly. "Insufficient crypto" maps to `51`, not `05`. Customers complain about generic declines.

## Things that bite you in production

1. **Auth/settle currency mismatch.** Cross-border txs auth in USD but settle in merchant's local currency at scheme rate. Your system must handle this.

2. **Partial captures.** Hotels auth $200, settle $173. Restaurants auth tip+1%, settle with actual tip. Don't assume auth amount = settle amount.

3. **Recurring transactions.** Netflix auths $1 then settles the real amount later. Your auth-time funding lock will be too small. Have a "recurring tx" exemption from velocity limits.

4. **Auth reversals without settlement.** Merchants reverse auths instead of settling for $0. Your funds lock release must handle this.

5. **Force-post / chargebacks without prior auth.** Some chargebacks land without any matching auth. Your ledger must handle "negative settle" with no parent.

6. **Network token vs PAN.** Visa/Mastercard tokens flow through different rails than raw PANs. Your CVM rules may differ.

7. **PIN debit vs signature credit.** Same card can route as either. Interchange differs.

8. **3DS (3-D Secure) for online txs.** You may need to support 3DS step-up authentication for high-risk online transactions. Issuer support varies.

## Estimated costs (rough, very approximate)

| Item | Cost |
|---|---|
| Issuer setup fee | $25k - $100k |
| Per-card issuance | $0.50 - $5 (virtual is cheaper) |
| Per-authorization | $0.05 - $0.20 |
| Per-settled-transaction | $0.05 - $0.10 |
| Monthly platform fee | $5k - $50k+ |
| Scheme fees (Visa/MC) | Variable, set by scheme |

You make money via interchange (~1-2.5% of tx volume) and your own FX/crypto spread (~2-5%). Your costs must come in well below that.

## Decision checklist

When picking an issuer:
- [ ] Do they support your target geography?
- [ ] Are crypto programs explicitly allowed (in writing) or just tolerated?
- [ ] What's the auth webhook timeout? Can your stack hit p99 < 50% of that?
- [ ] Do they support JIT funding? (Critical for crypto cards.)
- [ ] Do they handle 3DS, network tokens, and recurring txs natively?
- [ ] What's the BIN sponsor's underwriting comfort with crypto?
- [ ] What's the failover plan if they go down? (No good answer — pick the most reliable one)
