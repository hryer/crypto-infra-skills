---
name: vendor-rain
description: Backend integration guide for Rain — crypto-card issuer with corporate / business card focus, supporting USDC funding for spend in USD. Covers card-program setup, KYC/KYB, authorization flow, settlement, webhook handling, and ledger reconciliation. Use whenever the user is integrating Rain as the card-issuer vendor for a corporate or stablecoin-funded debit/credit card product.
---

# Rain

Crypto-card issuer focused on corporate / business cards backed by USDC. Vendor under [[category-crypto-card]].

## What this vendor is for

Rain issues corporate cards funded directly from a business's USDC balance. The auth + settlement model is similar to other card issuers, but the funding flow is stablecoin-first (vs Reap's broader crypto support, vs DCS's deeper bank-rails integration). Strong fit for SaaS / treasury / startup spend.

## Custody / data / pricing model

- **Custody model:** Rain manages the card program; you (or the corporate end-user) hold USDC in a designated wallet that funds the card.
- **Pricing:** Per-card + per-tx interchange share + FX (if non-USD spend) + monthly minimums. Negotiated.

## Auth & API setup

- API key + signing secret. Source from KMS / Vault.
- Per-environment keys (sandbox / production).
- mTLS may be required for the auth callback endpoint (verify with Rain).

<!-- TODO: fill in concrete setup steps from Rain docs -->

## Card lifecycle integration

### 1. Business onboarding / KYB
- KYB (Know Your Business) is heavier than KYC; allow weeks for new customers.
- On approval, create cards via API; persist `rain_card_id` → your `business_id`/`user_id`.

### 2. Funding
- Business deposits USDC to a designated funding address.
- Backend monitors deposits via [[vendor-alchemy]] / [[vendor-quicknode]] webhooks (with reorg-aware finality — wait for safe/finalized before crediting available balance).
- Notify Rain of available balance via API.

### 3. Authorization (auth) flow — **latency-critical**
- Card network → Rain → your auth endpoint.
- Strict ~2s round-trip budget.
- Verify available balance, place hold via [[category-internal-ledger]], respond approve/decline.

### 4. Settlement
- Daily/periodic settlement file.
- Reconcile each settled tx against the original hold; release stale holds per vendor SLA.

### 5. Card-state events
- Created / activated / blocked / closed → propagate to your domain.

## Webhook / callback handling

- **Signature verification:** mandatory on every callback. Auth callback especially — forged auths can drain cards.
- **Idempotency key:** Rain tx ID.
- **Latency budget:** <2s for auth; do not enqueue auth synchronously.

**Infra wiring:**
- Auth callback: synchronous; downstream notifications via SQS / Pub/Sub.
- Settlement: file drop → batch worker → reconciliation.
- State events: webhook → verify → enqueue → worker.

<!-- TODO: paste signature verification snippet + auth response shape -->

## Common integration mistakes

- Crediting USDC deposits at "confirmed" instead of finalized → reorg risk credits user, then funds disappear.
- Auth handler doing N+1 DB reads → blows the 2s budget.
- No hold mechanism → race condition between concurrent auths.
- Treating USDC 1:1 with USD without verifying the depeg risk in your risk policy.

<!-- TODO: extend -->

## MCP integration

<!-- TODO: check Rain MCP availability. -->

## Latest docs reference

- Official site: https://www.rain.xyz/
- Developer docs: *(get from Rain directly)*
- Last-verified: `<YYYY-MM-DD>` <!-- TODO: bump on each re-check -->

## When NOT to use this vendor

- **APAC consumer card program** → [[vendor-reap]] has stronger regional fit.
- **Need bank-rails / SGD/HKD card** → [[vendor-dcs]] may fit better.
- **Personal consumer cards** → Rain is corporate-focused.

## Cross-references

- Parent category: [[category-crypto-card]]
- Internal ledger: [[category-internal-ledger]]
- Funding wallet vendors: [[vendor-fireblocks]], [[vendor-fordefi]] (if custodial), or non-custodial USDC address
- Reviewed by: [[web3-backend-reviewer]] (§1, §7, §9)
- Alternatives: [[vendor-reap]], [[vendor-dcs]]
