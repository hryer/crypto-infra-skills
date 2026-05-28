---
name: vendor-helius
description: Backend integration guide for Helius — Solana-specialized RPC, enhanced transactions API, parsed events, webhooks, and DAS (Digital Asset Standard) for NFTs / compressed assets. Covers RPC patterns, enhanced tx parsing vs raw tx, webhook subscription, and Helius vs Solana-RPC providers. Use whenever the user is integrating Solana RPC, parsing Solana program events, or indexing compressed NFTs.
---

# Helius

Solana-specialized RPC + enhanced data API + webhooks. Primary Solana vendor under [[category-rpc-and-indexer]].

## What this vendor is for

Helius is to Solana what Alchemy is to EVM, with deeper Solana-specific enrichment: parsed transactions (instructions decoded to human-readable form), Digital Asset Standard (DAS) for NFTs (including compressed NFTs), priority fee API, and webhooks scoped to addresses or programs. For Solana backends, Helius is the default RPC + indexer pick.

## Custody / data / pricing model

- **Custody model:** N/A.
- **Pricing:** Credits-per-call with tiered plans. Enhanced APIs (parsed tx, DAS) cost more credits than raw RPC.

## Auth & API setup

- API key embedded in RPC URL — treat as secret. Source from KMS / Vault.
- Webhook auth via signing secret; verify every payload.
- Separate keys per environment.

<!-- TODO: fill in concrete setup steps from https://docs.helius.dev -->

## SDK usage

### TypeScript

Official SDK: `helius-sdk`. For raw RPC, use `@solana/web3.js` (or the newer `@solana/kit`) pointed at the Helius RPC URL.

<!-- TODO: paste minimal enhanced-tx fetch + webhook registration -->

### Golang

No first-party Go SDK; use community Solana libs (e.g., `github.com/gagliardetto/solana-go`) against the Helius RPC URL; REST for enhanced APIs.

<!-- TODO: paste gagliardetto/solana-go + Helius enhanced-tx REST call -->

Follow [Effective Go](https://go.dev/doc/effective_go) — context-first, error wrapping, careful with Solana's WebSocket subscriptions (long-lived; need ping/pong + reconnect logic).

## Webhook / callback handling

Helius webhooks scope on addresses or programs. Two types: `enhanced` (parsed) and `raw`.

- **Signature verification:** verify the signing secret before processing.
- **Idempotency key:** Helius event ID + transaction signature.
- **Retry semantics:** non-2xx triggers retry.

**Infra wiring:** webhook → verify → enqueue (SQS / Pub/Sub / Kafka) → ACK → worker. Reconciliation via `getSignaturesForAddress` periodic poll for missed events.

**Solana finality model differs from EVM:** confirmed → finalized. Choose the right commitment level for your product. For high-value flows, wait for `finalized` (~13 seconds at typical conditions).

<!-- TODO: paste signature verification snippet -->

## Common integration mistakes

- Treating `confirmed` as final — a `confirmed` tx can still be dropped before finalization. Wait for `finalized` for settlement.
- Using enhanced tx for every read — credits add up fast; use raw tx + parse locally when possible.
- Ignoring priority fees during congestion — txs sit unconfirmed; use Helius priority fee API.
- One Solana RPC node — for HA, configure fallback to another Solana RPC provider (Triton, Jito-shred, etc.).
- Logging full RPC URL (contains API key).

<!-- TODO: extend -->

## MCP integration

<!-- TODO: check Helius MCP availability. -->

## Latest docs reference

- Official docs: https://docs.helius.dev/
- Enhanced transactions: https://docs.helius.dev/api-reference/enhanced-transactions-api
- Webhooks: https://docs.helius.dev/webhooks-and-websockets/webhooks
- TypeScript SDK: https://github.com/helius-labs/helius-sdk
- Last-verified: `<YYYY-MM-DD>` <!-- TODO: bump on each re-check -->

## When NOT to use this vendor

- **EVM-only product** → use [[vendor-alchemy]] or [[vendor-quicknode]].
- **You need a chain not on Solana** → wrong vendor.
- **You want fully self-hosted** → run a Solana validator (heavy).

## Cross-references

- Parent category: [[category-rpc-and-indexer]]
- Reviewed by: [[web3-backend-reviewer]] (§1 reorg / finality, §3 multi-chain, §7 vendor integrations)
- Alternatives: Triton, Jito (Solana), [[vendor-quicknode]] (also covers Solana but less specialized)
