---
name: vendor-reap
description: Backend integration guide for Reap — crypto-card issuer with APAC focus, supporting USD/HKD/SGD card programs funded from stablecoin balances. Covers card-program setup, KYC handoff, authorization (auth) flow, settlement, webhook handling, and ledger reconciliation. Use whenever the user is integrating Reap as the card-issuer vendor for a crypto-funded debit/credit card product.
---

# Reap

Crypto-card issuer with strong APAC footprint. Vendor under [[category-crypto-card]].

## What this vendor is for

Reap issues debit/credit cards backed by user crypto / stablecoin balances. Strong in APAC (HK, SG) with multi-currency support. As the backend integrator, your responsibilities are: deposit / fund cards from user wallet balances, respond to authorization (auth) callbacks with approve/decline within strict latency, reconcile settlement files, and propagate card-status events back to your domain.

## Custody / data / pricing model

- **Custody model:** Reap acts as card issuer / program manager; your platform manages the funding source (custodial wallet via [[vendor-fireblocks]] / [[vendor-fordefi]] or non-custodial flow).
- **Pricing:** Per-card + per-tx interchange share + FX spread + monthly minimums. Negotiated.

## Auth & API setup

- API key + signing secret. Source from KMS / Vault.
- mTLS may be required for auth callback endpoint.
- Per-environment (sandbox / production) keys; sandbox uses Reap's mock card network.

<!-- TODO: fill in concrete setup steps from Reap docs / API reference -->

## Card lifecycle integration

### 1. Customer onboarding / KYC
- KYC: Reap-managed, your platform handles handoff, or shared. Confirm which model your contract uses.
- On KYC pass, create card in Reap; persist `reap_card_id` → your `user_id`.

### 2. Funding
- User deposits crypto into your custodial address.
- Your backend converts (off-chain ledger entry) and notifies Reap of available balance.

### 3. Authorization (auth) flow — **latency-critical**
- Cardholder swipes / taps → Mastercard/Visa → Reap → your auth endpoint
- You have **~2 seconds** to approve / decline. Beyond that the network declines automatically.
- Verify available balance, deduct hold via your internal ledger, respond.
- See [[category-internal-ledger]] for hold semantics.

### 4. Settlement
- Daily/periodic settlement file from Reap.
- Reconcile each settled tx against the corresponding hold; release any unsettled holds after vendor SLA.

### 5. Card-state events
- Card created / activated / blocked / closed → propagate to your domain.

## Webhook / callback handling

- **Signature verification:** verify Reap's signing on every callback. **Critical for auth callbacks** — anyone hitting your auth URL with a forged payload could drain a card.
- **Idempotency key:** Reap tx ID.
- **Latency budget for auth:** entire round-trip must complete in <2s; allocate accordingly across DB read, ledger update, response.

**Infra wiring:**
- Auth callback: synchronous, no queue (latency budget); but emit an async event to SQS / Pub/Sub for downstream consumers (notifications, analytics).
- Settlement file: file drop (S3 / SFTP) → batch worker → reconciliation.
- Card-state events: webhook → verify → enqueue → worker.

<!-- TODO: paste signature verification snippet + auth response shape -->

## Common integration mistakes

- Auth handler doing slow DB writes inline → 2s latency budget blown → declines.
- No hold mechanism in internal ledger → double-spend race condition between auth and balance read.
- Skipping signature verification on auth callback (catastrophic).
- Settlement reconciliation done weekly instead of daily → discrepancies pile up.
- Treating crypto-to-fiat FX rate as fixed for the day → FX moves; price holds at the moment of auth.

<!-- TODO: extend -->

## MCP integration

<!-- TODO: check Reap MCP availability. Unlikely; card issuers rarely offer one. -->

## Latest docs reference

- Official site: https://www.reap.global/
- Card developer docs: *(get from Reap directly; not publicly indexed last-verified)*
- Last-verified: `<YYYY-MM-DD>` <!-- TODO: bump on each re-check -->

## When NOT to use this vendor

- **US-domestic-only program** → [[vendor-rain]] or [[vendor-dcs]] may fit better depending on region.
- **Non-crypto traditional card program** → Marqeta / Stripe Issuing / Galileo / GPS.
- **You don't have AML / compliance staffing** → card programs are heavy; reconsider scope.

## Cross-references

- Parent category: [[category-crypto-card]]
- Internal ledger (holds, settlement): [[category-internal-ledger]]
- Funding wallet vendors: [[vendor-fireblocks]], [[vendor-fordefi]]
- Reviewed by: [[web3-backend-reviewer]] (§7 vendor integrations, §9 webhook reliability, §1 financial conservation)
- Alternatives: [[vendor-rain]], [[vendor-dcs]]
