---
name: vendor-birdeye
description: Backend integration guide for Birdeye — token data and security screening across Solana, EVM L1s/L2s, and other chains. Covers price API, token metadata, security scoring, holders, top traders, and webhook / streaming patterns. Use whenever the user is integrating token data, token-safety screening, holder analytics, or in-app token discovery at the backend.
---

# Birdeye

Token data + security screening across multiple chains, originally Solana-strong. Primary vendor under [[category-token-security-screening]] and a secondary source under [[category-price-feeds]].

## What this vendor is for

Birdeye exposes token metadata (name, symbol, decimals, logo), prices (per-token, OHLCV), holders, top traders, and security signals (mint authority status, freeze authority, LP locked %, suspicious patterns). For consumer apps and DEX front-ends doing in-app token discovery, Birdeye is one of the easiest off-the-shelf sources.

## Custody / data / pricing model

- **Custody model:** N/A (data API, no custody).
- **Pricing:** Tiered API plans (free → paid). Some endpoints (real-time WebSocket, deep history) require higher tiers. Pin down your call volume and required endpoints.

## Auth & API setup

- API key via `X-API-KEY` header (or `Authorization` per endpoint, check current docs).
- Source from KMS / Vault.
- Use separate keys per environment.

<!-- TODO: fill in concrete auth steps from https://docs.birdeye.so -->

## SDK usage

### TypeScript

<!-- TODO: confirm official TS SDK. If REST-only, build a typed wrapper around fetch with strict response types. -->

Cache aggressively — token metadata barely changes; prices change constantly but a 5-15s cache is usually fine for UI.

### Golang

<!-- TODO: confirm Go SDK. If REST-only, thin net/http client. -->

Follow [Effective Go](https://go.dev/doc/effective_go) — context for every HTTP call, error wrapping, no leaked goroutines on streaming connections.

## Webhook / callback handling

Birdeye primarily exposes REST + WebSocket streams (not classical webhooks).

- For real-time prices, WebSocket subscription per token / pool.
- Buffer WebSocket events through a local channel / queue; do NOT block the WS handler on DB writes.
- **Infra wiring:** WS handler → in-memory ring buffer → background worker → Redis cache + Postgres write.

<!-- TODO: paste WebSocket subscribe example -->

## Common integration mistakes

- Hammering the REST API in a tight loop instead of using WebSocket for high-frequency price updates.
- Treating Birdeye security signals as a complete answer — they're useful signals, not guarantees. Layer with on-chain heuristics (LP locked, owner-can-mint, blacklist function) and other sources (GoPlus, Honeypot.is).
- Caching token metadata for too short — wastes API quota.
- Logging full price stream into Kafka without aggregation — drowns downstream consumers.

<!-- TODO: extend -->

## MCP integration

<!-- TODO: check Birdeye MCP availability. -->

## Latest docs reference

- Official docs: https://docs.birdeye.so/
- API reference: https://public-api.birdeye.so/docs/ *(verify)*
- Last-verified: `<YYYY-MM-DD>` <!-- TODO: bump on each re-check -->

## When NOT to use this vendor

- **You need only EVM coverage with deep CEX prices** → [[vendor-coingecko]] or [[vendor-coinapi]].
- **You need oracle-grade prices for on-chain settlement** → use [[vendor-chainlink]] feeds, not Birdeye.
- **You need ground-truth token security audits** → professional audit, not API.

## Cross-references

- Parent category: [[category-token-security-screening]]
- Secondary category: [[category-price-feeds]]
- Reviewed by: [[web3-backend-reviewer]] (§7 vendor integrations)
- Alternatives: GoPlus, Honeypot.is, TokenSniffer, De.Fi for security; [[vendor-coingecko]], [[vendor-coinapi]] for prices
