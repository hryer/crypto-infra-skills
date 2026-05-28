---
name: vendor-dynamic
description: Backend integration guide for Dynamic — Tier 1 embedded wallet with strong multi-chain (EVM + Solana + Bitcoin) coverage. Covers JWT verification, server-side SDK, wallet provisioning, webhook handling, and multi-chain considerations. Use whenever the user is integrating Dynamic for consumer wallet UX in a multi-chain app or DeFi front-end.
---

# Dynamic

Tier-1 (consumer) embedded wallet, multi-chain-native. Default pick when the product spans EVM + Solana (or Bitcoin). For decision context — Dynamic vs Privy vs Web3Auth — start at [[category-wallet-custody]].

## What this vendor is for

Dynamic provides embedded wallets with auth (email / social / passkey) across EVM, Solana, and Bitcoin from a single SDK. It is the default pick for crypto-native or multi-chain consumer apps where chain diversity matters more than the most polished email/social UX.

## Custody / data / pricing model

- **Custody model:** Non-custodial-ish — MPC or TKey-based, depending on configuration.
- **Pricing:** Per-MAU, per-feature add-ons. Pin down MAU definition. See [[category-wallet-custody]] "Hidden vendor costs."

## Auth & API setup

- Environment ID + API key. Source from KMS / Vault; **never** commit secrets.
- Frontend uses public Environment ID; backend uses API key for server-side actions.
- JWT verification via Dynamic JWKS endpoint.

<!-- TODO: fill in concrete auth steps from https://docs.dynamic.xyz -->

## SDK usage

### TypeScript

Official SDK: `@dynamic-labs/sdk-react-core` (client), `@dynamic-labs/sdk-api` (server).

<!-- TODO: paste minimal JWT verification + user lookup example -->

Strict mode on. Cache JWKS appropriately.

### Golang

<!-- TODO: confirm Go SDK availability. If REST-only, build a thin verifier using golang-jwt against Dynamic's JWKS. -->

Follow [Effective Go](https://go.dev/doc/effective_go) — context, error wrapping, no leaks.

## Webhook / callback handling

Dynamic supports webhooks for user / wallet events.

- **Signature verification:** verify HMAC before processing.
- **Idempotency key:** Dynamic event ID.
- **Retry semantics:** non-2xx triggers retry.

**Infra wiring:** webhook → verify → enqueue (SQS / Pub/Sub / Kafka) → worker. Reconciliation via REST poll.

<!-- TODO: paste HMAC verification snippet -->

## Common integration mistakes

- Treating EVM and Solana wallets as the same key — they're different curves (secp256k1 vs ed25519). One Dynamic user has multiple chain wallets, not one wallet across chains. See [[category-wallet-custody]] "Multi-chain key management."
- Storing the API key in client code.
- Skipping JWT signature verification on backend routes.

<!-- TODO: extend -->

## MCP integration

<!-- TODO: check Dynamic MCP availability. -->

## Latest docs reference

- Official docs: https://docs.dynamic.xyz/
- Server SDK: https://docs.dynamic.xyz/api-reference/overview
- Webhook reference: https://docs.dynamic.xyz/webhooks/overview
- Last-verified: `<YYYY-MM-DD>` <!-- TODO: bump on each re-check -->

## When NOT to use this vendor

- **Pure EVM consumer app, polished email UX matters** → [[vendor-privy]] has the edge.
- **Treasury / signing infra** → wrong tier; use [[vendor-fireblocks]] / [[vendor-fordefi]].
- **You want to own the MPC stack** → Web3Auth.

## Cross-references

- Parent category: [[category-wallet-custody]]
- Reviewed by: [[web3-backend-reviewer]]
- Security-reviewed by: [[wallet-security-auditor]]
- Alternatives in same tier: [[vendor-privy]], [[vendor-turnkey]]
