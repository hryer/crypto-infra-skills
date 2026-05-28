---
name: vendor-turnkey
description: Backend integration guide for Turnkey — API-first HSM-backed signing infrastructure that works as either consumer (sub-orgs per user) or institutional (your treasury). Covers organization / sub-org model, activity API, policy engine, P256 API auth, webhook handling, and cost considerations at scale. Use whenever the user is integrating Turnkey for programmatic per-customer wallets, B2B SaaS wallets, or high-volume programmatic signing.
---

# Turnkey

API-first, HSM-backed signing infrastructure. Plays two roles: Tier-1 (per-user sub-orgs for consumer apps) and Tier-2 (your own treasury). Cheapest at high volume; requires more integration work than Privy/Dynamic. Start at [[category-wallet-custody]] for vendor choice context.

## What this vendor is for

Turnkey is HSM-backed signing with a programmatic, API-first model. Each customer (or each user) gets a sub-organization with its own policies and wallets. You retain full control over UX. It is the default pick for B2B crypto SaaS (per-customer wallets) and for high-volume programmatic signing where per-tx vendor cost dominates.

## Custody / data / pricing model

- **Custody model:** HSM-backed (CloudHSM-class). Sub-organizations isolate users / tenants. Non-custodial-ish at the per-user level.
- **Pricing:** Cheap at scale (per-API-call, not per-MAU). Pin down volume tiers.

## Auth & API setup

- API public key + API private key (P256 / secp256r1, not secp256k1). Source from KMS / Vault.
- Every request is signed with your API private key; verified by Turnkey.
- Organization ID + sub-organization IDs structure the multi-tenancy.

<!-- TODO: fill in concrete auth steps from https://docs.turnkey.com -->

## SDK usage

### TypeScript

Official SDK: `@turnkey/sdk-server`, `@turnkey/sdk-browser`.

<!-- TODO: paste minimal createSubOrganization + createWallet + signTransaction example -->

### Golang

<!-- TODO: confirm Go SDK availability. If only REST, build a thin client with P256 signing. -->

Follow [Effective Go](https://go.dev/doc/effective_go) — context-first, structured errors, no leaked goroutines on activity-poll loops.

## Webhook / callback handling

Turnkey uses an Activity API model — many operations return an activity ID that you poll, rather than push-based webhooks for every event.

- **Pattern:** create activity → poll until terminal status → handle result.
- For async user flows, layer your own webhook from your backend to your frontend; Turnkey itself is request/response heavy.

<!-- TODO: confirm webhook availability for specific events -->

**Infra wiring:** background worker polls open activities; persist activity ID alongside your domain entity (e.g., `order.turnkey_activity_id`) for idempotency.

## Common integration mistakes

- Using secp256k1 keys for Turnkey API auth — Turnkey API auth uses P256. Different curve.
- Skipping sub-organization isolation in B2B SaaS — putting all tenants in one org collapses the security model.
- Polling activity IDs without bounded backoff — wastes API quota.
- Storing API private key in env vars.

<!-- TODO: extend -->

## MCP integration

<!-- TODO: check Turnkey MCP availability. -->

## Latest docs reference

- Official docs: https://docs.turnkey.com/
- API reference: https://docs.turnkey.com/api-reference/overview
- SDK: https://github.com/tkhq/sdk
- Last-verified: `<YYYY-MM-DD>` <!-- TODO: bump on each re-check -->

## When NOT to use this vendor

- **You need a polished embedded-wallet UX out of the box** → [[vendor-privy]] / [[vendor-dynamic]].
- **You need a comprehensive policy engine UI for non-technical approvers** → [[vendor-fireblocks]].
- **You need regulated qualified custody** → BitGo / Anchorage.
- **DeFi-native institutional flows** → [[vendor-fordefi]].

## Cross-references

- Parent category: [[category-wallet-custody]]
- Reviewed by: [[web3-backend-reviewer]] (§7 vendor integrations)
- Security-reviewed by: [[wallet-security-auditor]] (HSM, P256 key handling, sub-org isolation)
- Alternatives: [[vendor-privy]] (consumer Tier 1), [[vendor-fireblocks]] / [[vendor-fordefi]] (institutional Tier 2)
