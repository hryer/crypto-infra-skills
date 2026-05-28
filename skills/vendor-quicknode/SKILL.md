---
name: vendor-quicknode
description: Backend integration guide for QuickNode — multi-chain RPC plus QuickNode Streams (managed indexer), Functions (serverless on chain events), and Marketplace add-ons. Covers RPC patterns, Streams filter design, webhook signature verification, and QuickNode as primary or fallback RPC. Use whenever the user is integrating QuickNode for RPC, managed indexing via Streams, or as a fallback / failover RPC vendor.
---

# QuickNode

Multi-chain RPC + managed indexer (Streams) + serverless event functions. Primary vendor under [[category-rpc-and-indexer]]; commonly paired with [[vendor-alchemy]] for RPC failover.

## What this vendor is for

QuickNode provides JSON-RPC across EVM chains, Solana, Bitcoin, and others. **Streams** is their managed indexer (filter on chain events, get a webhook or push to S3 / Postgres / Snowflake). **Functions** runs serverless code triggered by chain events. For backends that want managed indexing without operating their own pipeline, Streams is the main draw.

## Custody / data / pricing model

- **Custody model:** N/A.
- **Pricing:** Credits-per-call (RPC), separate metering for Streams and Functions. Plus per-endpoint base subscription.

## Auth & API setup

- Per-endpoint URL contains an auth token in the path — **treat the full URL as a secret**. Source from KMS / Vault.
- For Streams / Functions, separate API key.
- Rotate URLs (rotate auth) periodically.

<!-- TODO: fill in concrete setup steps from https://www.quicknode.com/docs -->

## SDK usage

### TypeScript

No first-party comprehensive SDK; use [viem](https://viem.sh) for RPC, REST for Streams.

<!-- TODO: paste minimal viem + QuickNode RPC example + Streams filter registration -->

### Golang

Use `github.com/ethereum/go-ethereum/ethclient` against QuickNode RPC URL. REST for Streams.

<!-- TODO: paste go-ethereum dial + Streams REST registration -->

Follow [Effective Go](https://go.dev/doc/effective_go) — context-first, error wrapping, bounded backoff on RPC retries.

## Webhook / callback handling (Streams)

Streams pushes matched events to your destination: webhook, S3, Postgres, Snowflake, etc.

- **Signature verification:** verify QuickNode's signed payload before processing.
- **Idempotency key:** Streams provides an event ID; use it.
- **Retry semantics:** non-2xx triggers retry.

**Infra wiring:**
- For webhook destination: receive → verify → enqueue (SQS / Pub/Sub / Kafka) → ACK → worker.
- For S3 destination: process via S3 event notifications → Lambda / Cloud Function → DB.
- **Reorg handling:** Streams emits both `confirmed` and (depending on chain) `finalized` markers — design DB writes around the right finality level for your product.

<!-- TODO: paste signature verification snippet -->

## Common integration mistakes

- Leaking the full QuickNode endpoint URL in client code, logs, or sentry breadcrumbs.
- Filtering Streams too broadly → massive event volume, runaway costs.
- Treating "confirmed" as "final" on EVM — wait for finalized for settlement flows.
- No fallback RPC — see [[web3-backend-reviewer]] §9 (fallback strategy).
- Forgetting that Streams backfill is a separate API from live streaming.

<!-- TODO: extend -->

## MCP integration

<!-- TODO: check QuickNode MCP availability. -->

## Latest docs reference

- Official docs: https://www.quicknode.com/docs
- Streams: https://www.quicknode.com/docs/streams
- Functions: https://www.quicknode.com/docs/functions
- Last-verified: `<YYYY-MM-DD>` <!-- TODO: bump on each re-check -->

## When NOT to use this vendor

- **Solana-heavy product** → [[vendor-helius]] is purpose-built.
- **Embedded NFT API needs** → [[vendor-alchemy]] NFT API is more mature.
- **Want fully self-hosted** → run geth / erigon / reth (see [[category-rpc-and-indexer]] DIY).

## Cross-references

- Parent category: [[category-rpc-and-indexer]]
- Reviewed by: [[web3-backend-reviewer]] (§1 reorg, §2 failure modes, §7 vendor integrations, §9 fallback paths)
- Alternatives: [[vendor-alchemy]], [[vendor-helius]] (Solana), Infura
