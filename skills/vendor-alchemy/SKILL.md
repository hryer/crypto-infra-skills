---
name: vendor-alchemy
description: Backend integration guide for Alchemy — multi-chain RPC, Notify webhooks (mined tx, address activity, dropped tx), NFT API, Prices API, and embedded wallet products. Covers RPC patterns, webhook signature verification, reorg-aware ingestion, and Alchemy as both primary RPC and fallback. Use whenever the user is integrating Alchemy for chain data, transaction monitoring, NFT data, prices, or as a fallback RPC vendor.
---

# Alchemy

Multi-chain RPC + indexer + webhooks. Primary vendor under [[category-rpc-and-indexer]] and a secondary source under [[category-price-feeds]] (Alchemy Prices API).

## What this vendor is for

Alchemy provides JSON-RPC endpoints across most EVM chains and Solana, plus higher-level APIs: NFT API, Token Prices API, and **Notify** webhooks (mined transaction, dropped transaction, address activity). For most backends, Alchemy is either the primary RPC or the second leg in a fallback pair (e.g., primary [[vendor-quicknode]] + fallback Alchemy, or vice versa).

## Custody / data / pricing model

- **Custody model:** N/A.
- **Pricing:** CU (Compute Units) per call, plus add-ons (Notify, NFT, Prices). Pin down CU rate per method (e.g., `eth_getLogs` is expensive).

## Auth & API setup

- API key per app (per chain + per environment). Source from KMS / Vault.
- Use separate Alchemy apps for sandbox / staging / mainnet — don't share keys.
- Webhook signing key is separate; verify every webhook.

<!-- TODO: fill in concrete setup steps from https://docs.alchemy.com -->

## SDK usage

### TypeScript

Official SDK: `alchemy-sdk`. **Prefer [viem](https://viem.sh)** for new code that talks to standard RPC — point viem's transport at Alchemy's RPC URL. Use `alchemy-sdk` specifically when you need Alchemy-only features (Notify webhooks, NFT API, Prices).

<!-- TODO: paste minimal viem + Alchemy RPC example, and alchemy-sdk for Notify webhook registration -->

### Golang

No official Go SDK; use `github.com/ethereum/go-ethereum/ethclient` against Alchemy's RPC URL, plus a thin REST client for Notify webhooks.

<!-- TODO: paste go-ethereum dial + Notify webhook registration via REST -->

Follow [Effective Go](https://go.dev/doc/effective_go) — context-first on every RPC call, error wrapping (`fmt.Errorf("alchemy getLogs: %w", err)`), no leaked goroutines on subscription loops.

## Webhook / callback handling (Notify)

Alchemy Notify webhook types include `MINED_TRANSACTION`, `DROPPED_TRANSACTION`, `ADDRESS_ACTIVITY`, `GRAPHQL` (custom).

- **Signature verification:** HMAC SHA256 with the webhook signing key. **Always verify.**
- **Idempotency key:** Alchemy `webhookId + eventId`.
- **Retry semantics:** non-2xx triggers retry with exponential backoff.

**Infra wiring (critical):**
- Webhook → verify signature → enqueue (AWS SQS / GCP Pub/Sub / Kafka) → ACK immediately.
- Worker consumes from queue → updates DB transactionally → emits domain event.
- **Reorg awareness:** `MINED_TRANSACTION` fires at mined, NOT at safe/finalized. For high-value flows, wait N confirmations before treating as final. See [[web3-backend-reviewer]] §1 (reorg handling).
- **Reconciliation job:** poll `eth_getLogs` or `eth_getTransactionReceipt` for any address you've registered, in case webhooks were missed.

<!-- TODO: paste signature verification snippet (Node + Go) -->

## Common integration mistakes

- Skipping signature verification "because it's behind WAF" — verify anyway.
- Treating `MINED_TRANSACTION` as finalized → reorg can reverse it. Wait for safe/finalized.
- Logging full webhook payloads (PII / wallet address exposure).
- Single Alchemy region — for HA, configure fallback to QuickNode / Infura at the RPC layer.
- Forgetting CU pricing — `eth_getLogs` on a wide block range can drain monthly quota in one call.

<!-- TODO: extend -->

## MCP integration

<!-- TODO: check Alchemy MCP availability. -->

## Latest docs reference

- Official docs: https://docs.alchemy.com/
- Notify webhooks: https://docs.alchemy.com/reference/notify-api-quickstart
- TypeScript SDK: https://github.com/alchemyplatform/alchemy-sdk-js
- Prices API: https://docs.alchemy.com/reference/prices-api-quickstart
- Last-verified: `<YYYY-MM-DD>` <!-- TODO: bump on each re-check -->

## When NOT to use this vendor

- **Solana-heavy product** → [[vendor-helius]] has stronger Solana coverage.
- **You want only WebSocket subscriptions and stream-style indexing** → [[vendor-quicknode]] Streams may fit better.
- **You need self-hosted node** → run geth / erigon / reth yourself (see [[category-rpc-and-indexer]] DIY).

## Cross-references

- Parent category: [[category-rpc-and-indexer]]
- Secondary category: [[category-price-feeds]]
- Reviewed by: [[web3-backend-reviewer]] (§1 reorg handling, §2 failure modes, §7 vendor integrations, §9 fallback paths)
- Alternatives: [[vendor-quicknode]], [[vendor-helius]] (Solana), Infura
