---
name: vendor-coingecko
description: Backend integration guide for Coingecko — broad-coverage token and market price API. Covers public vs pro API, rate limits, historical OHLCV, derivatives, and caching strategy. Use whenever the user is integrating Coingecko for token prices, market caps, exchange data, or as a price-feed fallback.
---

# Coingecko

Broad-coverage token + market price API. Vendor under [[category-price-feeds]].

## What this vendor is for

Coingecko aggregates token prices from many CEXes / DEXes and exposes per-token spot price, market cap, OHLCV history, exchange listings, derivatives, and metadata. For UI display ("here's BTC at $X"), Coingecko is the default. For settlement / on-chain pricing, use an oracle ([[vendor-chainlink]]) — Coingecko is informational, not authoritative.

## Custody / data / pricing model

- **Custody model:** N/A.
- **Pricing:** Free public API (rate-limited, no SLA) + paid Pro / Demo tiers with higher RPS + commercial use rights.

## Auth & API setup

- Free API: no key required (with low rate limits).
- Pro / Demo: API key via `x-cg-pro-api-key` / `x-cg-demo-api-key` header. Source from KMS / Vault.
- Separate keys per environment.

<!-- TODO: fill in concrete setup steps from https://docs.coingecko.com -->

## SDK usage

### TypeScript

No first-party SDK; use `fetch` with typed response wrappers. Several community SDKs exist; verify maintenance before adopting.

<!-- TODO: paste minimal /simple/price + /coins/{id}/market_chart example -->

### Golang

No first-party SDK; thin `net/http` client + typed structs. Community SDKs exist.

<!-- TODO: paste Go fetch example -->

Follow [Effective Go](https://go.dev/doc/effective_go) — context-first, error wrapping, rate-limit-aware retry.

## Caching strategy

- Cache prices in Redis with short TTL (15-60s for UI; 1-5s if you really need latest).
- Cache token metadata (id, symbol, name, image URL) with long TTL (hours / days).
- Use stale-while-revalidate for resilience during Coingecko outages.

## Rate limits

Public API is severely rate-limited (varies by endpoint, ~10-30 RPM historically). Pro tiers offer higher RPS but still finite. **Don't call Coingecko per-request from your hot path** — always go through a cache.

## Common integration mistakes

- Calling Coingecko per request from the API layer → 429s under load.
- Using Coingecko price for on-chain settlement → not oracle-grade; use [[vendor-chainlink]] feeds.
- Not handling 429 with backoff.
- Free-tier in production with no failover.
- Trusting `symbol` as identifier — Coingecko's stable identifier is `id` (the slug). Symbols collide (USDC on multiple chains).

<!-- TODO: extend -->

## MCP integration

<!-- TODO: check Coingecko MCP availability. -->

## Latest docs reference

- Official docs: https://docs.coingecko.com/
- Public API: https://www.coingecko.com/api/documentation
- Pro API: https://www.coingecko.com/en/api/pricing
- Last-verified: `<YYYY-MM-DD>` <!-- TODO: bump on each re-check -->

## When NOT to use this vendor

- **On-chain price oracle for settlement** → [[vendor-chainlink]] feeds, not Coingecko.
- **Real-time HFT-grade prices** → [[vendor-coinapi]] or direct exchange WebSocket.
- **Solana-native token discovery** → [[vendor-birdeye]] is more current.
- **You only need one specific token from one exchange** → call that exchange's API directly.

## Cross-references

- Parent category: [[category-price-feeds]]
- Reviewed by: [[web3-backend-reviewer]] (§7 vendor integrations, §9 caching / fallback)
- Alternatives: [[vendor-coinmarketcap]], [[vendor-coinapi]], [[vendor-chainlink]] (on-chain), [[vendor-birdeye]] (Solana)
