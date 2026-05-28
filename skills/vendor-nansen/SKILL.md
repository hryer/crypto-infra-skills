---
name: vendor-nansen
description: Backend integration guide for Nansen — on-chain analytics platform with labeled addresses, smart-money signals, and token-flow analytics. Covers Nansen API (Query), Alerts, Smart Alerts, and integration patterns for trading / quant / monitoring backends. Use whenever the user is integrating Nansen for wallet labeling, smart-money tracking, token-flow analytics, or quant signal generation.
---

# Nansen

On-chain analytics with labeled addresses and smart-money signals. Primary vendor under [[category-onchain-analytics]].

## What this vendor is for

Nansen labels millions of on-chain addresses ("Smart Money," "MEV Bot," exchange hot wallets, fund wallets) and exposes that lens via dashboards, alerts, and an API. For backends that drive trading signals, copy-trade strategies, fund-flow monitoring, or DD on protocols, Nansen's labeled data is the differentiator vs Dune (which exposes raw data without labels).

## Custody / data / pricing model

- **Custody model:** N/A.
- **Pricing:** Tiered, with API access typically gated to higher (Alpha / Pro / enterprise) tiers.

## Auth & API setup

- API key via header. Source from KMS / Vault.
- Per-environment keys.

<!-- TODO: fill in concrete setup from https://docs.nansen.ai -->

## SDK usage

### TypeScript

<!-- TODO: confirm SDK availability. If REST-only, thin typed fetch wrapper. -->

### Golang

<!-- TODO: confirm SDK availability. If REST-only, thin net/http client. -->

Follow [Effective Go](https://go.dev/doc/effective_go) — context-first, error wrapping, careful with potentially large response payloads (paginate, stream).

## Use cases

### Smart-money signal generation
Subscribe to "Smart Money" wallet movements → ingest events → run your own signal logic (e.g., emit a buy signal when N labeled wallets buy the same token within a window).

### Token-flow monitoring
Track inflows / outflows for a watched token at the labeled-wallet granularity (e.g., "exchange inflows over the last hour" → arbitrage signal).

### Fund-wallet alerts
Alert when a labeled fund / venture wallet changes position — useful for fund-of-funds reporting or copy-trading.

## Webhook / callback handling

Nansen Alerts can push to webhooks.

- **Signature verification:** verify Nansen's signed payload.
- **Idempotency key:** Nansen event ID.
- **Infra wiring:** webhook → verify → enqueue (SQS / Pub/Sub / Kafka) → worker → trading / signal engine. Reconciliation: periodic REST poll of recent wallet activity for label sets you care about.

<!-- TODO: verify webhook format and signature scheme -->

## Common integration mistakes

- Treating Nansen labels as ground truth — labels are heuristic; verify against your own watchlist for high-stakes decisions.
- Querying Nansen per request from a hot path — cache aggressively.
- No fallback when Nansen is down — for trading signals, degrade gracefully (e.g., reduce position size rather than miss entirely).
- Logging full wallet lists into Kafka without aggregation.

<!-- TODO: extend -->

## MCP integration

<!-- TODO: check Nansen MCP availability. -->

## Latest docs reference

- Official site: https://www.nansen.ai/
- API docs: https://docs.nansen.ai/ *(verify URL)*
- Last-verified: `<YYYY-MM-DD>` <!-- TODO: bump on each re-check -->

## When NOT to use this vendor

- **You need raw on-chain SQL access** → Dune is more flexible (and cheaper).
- **You need adversarial / illicit-flow tracing** → Arkham / Chainalysis are more focused.
- **You need on-chain settlement-grade data** → use a node ([[category-rpc-and-indexer]]) or oracle.

## Cross-references

- Parent category: [[category-onchain-analytics]]
- Reviewed by: [[web3-backend-reviewer]] (§7 vendor integrations, §9 fallback strategy)
- Alternatives: Dune Analytics, Arkham, Glassnode, DefiLlama
