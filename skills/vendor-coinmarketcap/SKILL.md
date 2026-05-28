---
name: vendor-coinmarketcap
description: Backend integration guide for CoinMarketCap (CMC) — broad-coverage token and market price API, often used as a Coingecko alternative or secondary source. Covers tiered API, rate limits, conversion endpoints, and caching strategy. Use whenever the user is integrating CMC for token prices, market caps, or as a price-feed fallback alongside Coingecko.
---

# CoinMarketCap (CMC)

Broad-coverage token + market price API. Vendor under [[category-price-feeds]]. Commonly paired with [[vendor-coingecko]] as the redundant source.

## What this vendor is for

CMC aggregates token prices and market data, similar to Coingecko. The two are often used together for redundancy — if one is rate-limited or down, the other answers. CMC's data sometimes diverges from Coingecko's (different exchange weighting); reconcile carefully if you switch between them.

## Custody / data / pricing model

- **Custody model:** N/A.
- **Pricing:** Tiered (Basic, Hobbyist, Startup, Standard, Professional, Enterprise). Each tier raises call credits and unlocks endpoints (historical, derivatives, etc.). Basic tier is restrictive for production.

## Auth & API setup

- API key via `X-CMC_PRO_API_KEY` header. Source from KMS / Vault.
- Separate keys per environment.

<!-- TODO: fill in concrete setup steps from https://coinmarketcap.com/api/documentation/v1/ -->

## SDK usage

### TypeScript

No first-party SDK; `fetch` with typed wrappers.

<!-- TODO: paste minimal /cryptocurrency/quotes/latest example -->

### Golang

No first-party SDK; thin `net/http` client + typed structs.

<!-- TODO: paste Go fetch example -->

Follow [Effective Go](https://go.dev/doc/effective_go) — context-first, error wrapping, rate-limit-aware retry.

## Caching strategy

Same shape as [[vendor-coingecko]] — Redis-backed, short TTL for prices, long TTL for metadata, stale-while-revalidate during outages.

## Rate limits

CMC credits are per-call and per-endpoint (some endpoints cost multiple credits). Read the credit table for every endpoint you use; monthly credits cap is the operative constraint.

## Common integration mistakes

- Using `symbol` as identifier — CMC's stable identifier is the numeric `id`. Symbols collide.
- Calling CMC per request without caching.
- Mixing CMC and Coingecko prices in the same flow without reconciling — divergence creates bugs.
- Storing API key in env vars.

<!-- TODO: extend -->

## MCP integration

<!-- TODO: check CMC MCP availability. -->

## Latest docs reference

- Official docs: https://coinmarketcap.com/api/documentation/v1/
- Pricing: https://coinmarketcap.com/api/pricing/
- Last-verified: `<YYYY-MM-DD>` <!-- TODO: bump on each re-check -->

## When NOT to use this vendor

- **On-chain settlement** → use [[vendor-chainlink]].
- **Real-time HFT-grade** → [[vendor-coinapi]] or direct exchange WebSocket.
- **Cost-sensitive single-source** → [[vendor-coingecko]] free tier may suffice for prototypes.

## Cross-references

- Parent category: [[category-price-feeds]]
- Reviewed by: [[web3-backend-reviewer]] (§7 vendor integrations, §9 caching / fallback)
- Alternatives: [[vendor-coingecko]], [[vendor-coinapi]], [[vendor-chainlink]] (on-chain)
