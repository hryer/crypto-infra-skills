---
name: vendor-dcs
description: Backend integration guide for DCS (Digital Currency Services) — Singapore-based crypto-card issuer with bank-rails integration, supporting multi-currency cards and crypto-funded spend. Covers card-program setup, KYC, authorization flow, settlement, webhook handling, and reconciliation. Use whenever the user is integrating DCS as the card-issuer vendor, especially for APAC / SG / regulated card programs.
---

# DCS

Singapore-licensed crypto-card issuer with bank-rails integration. Vendor under [[category-crypto-card]].

## What this vendor is for

DCS operates a regulated card program with deeper bank-rails integration than pure crypto-native issuers, well-suited to SG/APAC market and to programs that need fiat on/off-ramp tied directly to the card. As the backend integrator your responsibilities mirror other card issuers: funding, auth, settlement, reconciliation, and state events.

## Custody / data / pricing model

- **Custody model:** DCS manages issuance + bank rails; your platform manages funding wallets and the user-balance ledger.
- **Pricing:** Per-card + per-tx + FX spread + bank-rails fees + monthly minimums. Negotiated; regulated programs typically priced higher.

## Auth & API setup

- API credentials + signing material. Source from KMS / Vault.
- mTLS likely required for auth callback endpoint (regulated programs trend stricter).
- Per-environment (sandbox / production).

<!-- TODO: fill in concrete setup from DCS API docs (often NDA-gated) -->

## Card lifecycle integration

### 1. Customer onboarding / KYC
- KYC handled per regulatory regime (MAS in Singapore); may require eKYC + manual review for higher tiers.
- Persist `dcs_card_id` → your `user_id`.

### 2. Funding
- User deposits crypto → your custodial wallet ([[vendor-fireblocks]] / [[vendor-fordefi]] typical).
- Convert internally (off-chain ledger), notify DCS of available balance.
- For fiat top-ups via bank rails, DCS may handle directly.

### 3. Authorization (auth) flow — **latency-critical**
- Card network → DCS → your auth endpoint.
- Strict ~2s round-trip budget.
- Verify balance, hold via [[category-internal-ledger]], respond.

### 4. Settlement
- Settlement file (daily/periodic).
- Reconcile holds vs settled; release stale holds.

### 5. Card-state events
- Created / activated / suspended / closed → propagate.

## Webhook / callback handling

- **Signature verification:** mandatory; regulated programs may also require IP allowlisting + mTLS.
- **Idempotency key:** DCS tx ID.
- **Latency:** <2s for auth.

**Infra wiring:**
- Auth callback: synchronous; emit async event downstream via SQS / Pub/Sub.
- Settlement: file drop → batch worker → reconciliation.
- State events: webhook → verify → enqueue → worker.

<!-- TODO: paste signature verification snippet -->

## Common integration mistakes

- Auth handler doing inline DB writes that exceed 2s budget.
- No hold mechanism → race condition between concurrent auths.
- Reconciliation only weekly → regulatory issues if discrepancies aren't caught quickly.
- Skipping mTLS / IP allowlist when required.
- Failing to log + retain auth + settlement data per MAS retention requirements.

<!-- TODO: extend -->

## MCP integration

<!-- TODO: check DCS MCP availability. Unlikely for regulated card programs. -->

## Latest docs reference

- Official site: https://www.dcscc.com/ *(verify)*
- Developer docs: *(NDA / direct from DCS)*
- Last-verified: `<YYYY-MM-DD>` <!-- TODO: bump on each re-check -->

## When NOT to use this vendor

- **US-domestic card program** → use a US-licensed issuer.
- **Pure consumer crypto-native UX** → [[vendor-reap]] / [[vendor-rain]] are lighter integration.
- **Non-regulated test product** → DCS is heavy for prototyping.

## Cross-references

- Parent category: [[category-crypto-card]]
- Internal ledger: [[category-internal-ledger]]
- Funding wallet vendors: [[vendor-fireblocks]], [[vendor-fordefi]]
- Reviewed by: [[web3-backend-reviewer]] (§1 financial conservation, §7 vendor integrations, §9 webhook reliability)
- Alternatives: [[vendor-reap]], [[vendor-rain]]
