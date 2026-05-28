---
name: vendor-fordefi
description: Backend integration guide for Fordefi — MPC custody platform with DeFi-native UX and faster chain enablement than legacy custodians. Covers vault setup, transaction API, webhooks, policy engine, and the API signer. Use whenever the user is integrating Fordefi for DeFi treasury, automated signing, or institutional flows that need fresh-L2 coverage.
---

# Fordefi

Tier-2 institutional MPC custody, modern alternative to Fireblocks with stronger DeFi UX and faster support for new L2s. For decision-level context — when to pick Fordefi vs Fireblocks / Turnkey — start at [[category-wallet-custody]].

## What this vendor is for

Fordefi is an MPC-based institutional signer focused on DeFi-native flows. Vaults are EOA-style addresses (no smart-contract proxy), so they interact with DeFi protocols natively. It is the default pick for teams that need treasury + active DeFi participation without the chain-support lag of Fireblocks.

## Custody / data / pricing model

- **Custody model:** Custodial-with-MPC. Fordefi runs MPC nodes; you control policy.
- **What Fordefi guarantees:** SOC 2 Type II, policy engine, audit logs, DeFi-app-aware approvals (sees what protocol you're interacting with).
- **Pricing:** Per-workspace + per-tx. Generally less opaque than Fireblocks; still ask for caps. See [[category-wallet-custody]] "Hidden vendor costs."

## Auth & API setup

- API key + signer secret (used to sign API requests).
- The API signer secret is sensitive — **never** plaintext. Use KMS / Vault.
- Fordefi exposes a separate **API Signer** binary that you can self-host for added isolation between API calls and signing material.

<!-- TODO: fill in concrete auth steps from https://docs.fordefi.com/api -->

## SDK usage

### TypeScript

<!-- TODO: confirm official TS SDK availability. If only REST, build a thin wrapper using fetch with typed responses. -->

Strict mode on. Wrap Fordefi errors in your domain error types at the boundary.

### Golang

<!-- TODO: confirm official Go SDK availability. If only REST, build a thin client using net/http. -->

Follow [Effective Go](https://go.dev/doc/effective_go) — context-first, error wrapping, no leaked goroutines on polling.

## Webhook / callback handling

Fordefi pushes webhooks for transaction lifecycle events.

- **Signature verification:** Fordefi signs every webhook — verify before processing.
- **Idempotency key:** use Fordefi `transaction_id` as your idempotency key.
- **Retry semantics:** non-2xx triggers retry; consumer must be idempotent.

**Infra wiring:**
- Webhook → verify → enqueue (SQS / Pub/Sub / Kafka) → ACK → worker processes.
- **Reconciliation job:** poll `GET /api/v1/transactions` for non-terminal txs every 5 minutes.

<!-- TODO: paste signature verification snippet -->

## Common integration mistakes

- Storing the API signer secret in env vars or plaintext.
- Skipping webhook signature verification.
- Not modeling Fordefi vault accounts as first-class entities in your DB (causes confusion when one vault holds multiple addresses across chains).
- Hard-coding chain IDs Fordefi supports today without a chain-enablement check at startup.

<!-- TODO: extend with concrete pitfalls as encountered -->

## MCP integration

<!-- TODO: check if Fordefi offers an MCP server. If not, omit this section. -->

## Latest docs reference

- Official docs: https://docs.fordefi.com/
- API reference: https://docs.fordefi.com/api
- Webhook reference: https://docs.fordefi.com/api/webhooks *(verify URL)*
- Last-verified: `<YYYY-MM-DD>` <!-- TODO: bump on each re-check -->

## When NOT to use this vendor

- **Consumer-facing user wallet** → wrong tier; use [[vendor-privy]] or [[vendor-dynamic]].
- **Enterprise compliance shop that already runs Fireblocks** → switching cost rarely justified; see [[vendor-fireblocks]].
- **Pure programmatic signing at huge volume** → [[vendor-turnkey]] is cheaper.
- **Need regulated qualified custody** → BitGo / Anchorage.

## Cross-references

- Parent category: [[category-wallet-custody]]
- Reviewed by: [[web3-backend-reviewer]] (§7 vendor integrations, §9 webhook handling)
- Security-reviewed by: [[wallet-security-auditor]]
- Alternatives in same tier: [[vendor-fireblocks]], [[vendor-turnkey]]
