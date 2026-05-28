---
name: vendor-coinapi
description: Backend integration guide for CoinAPI — institutional-grade aggregated market data with REST and WebSocket / FIX feeds. Covers REST quotes, WebSocket subscriptions, historical OHLCV / trades, and use as a higher-fidelity alternative to Coingecko / CMC. Use whenever the user is integrating CoinAPI for institutional pricing, trade-grade data, or HFT-adjacent flows.
---

# CoinAPI

Institutional-grade aggregated market data. Vendor under [[category-price-feeds]] when fidelity matters more than free / cheap.

## What this vendor is for

CoinAPI provides REST + WebSocket + FIX feeds with aggregated trades / quotes / OHLCV across hundreds of exchanges, including derivatives and Layer-2 DEXes. It is the right pick when you need trade-grade or near-HFT data (latency, completeness) and are willing to pay institutional rates.

## Custody / data / pricing model

- **Custody model:** N/A.
- **Pricing:** Tiered with significantly higher pricing than Coingecko / CMC, reflecting institutional positioning.

## Auth & API setup

- API key via `X-CoinAPI-Key` header. Source from KMS / Vault.
- WebSocket auth via initial subscription message containing the API key.

<!-- TODO: fill in concrete setup steps from https://docs.coinapi.io -->

## SDK usage

### TypeScript

Several community SDKs; `coinapi-sdk` is one. Or use `fetch` for REST and `ws` for WebSocket.

<!-- TODO: paste minimal REST quote + WebSocket trades subscription -->

### Golang

Community SDKs available. For WebSocket, `nhooyr.io/websocket` or `gorilla/websocket`.

<!-- TODO: paste Go WebSocket example with reconnect logic -->

Follow [Effective Go](https://go.dev/doc/effective_go) — context-first, bounded backoff on WS reconnect, no leaked goroutines on the read/write pumps.

## WebSocket / streaming patterns

- Subscribe per symbol pair you need; don't subscribe to "all" — message volume is huge.
- Run the WS pump on its own goroutine; emit into a buffered channel; let downstream consumers (Redis cache, Kafka producer) drain.
- Reconnect with exponential backoff + jitter; resubscribe on reconnect.
- Monitor staleness (last-message age) — alert if no message for N seconds on an active symbol.

## Common integration mistakes

- Subscribing to too many symbols → message storm → consumer can't keep up → reconnect loop.
- Single-threaded handler doing DB writes inline → blocks the WS read; messages queue up server-side.
- No reconnect logic → silent stale data.
- Storing API key in env vars.

<!-- TODO: extend -->

## MCP integration

<!-- TODO: check CoinAPI MCP availability. -->

## Latest docs reference

- Official docs: https://docs.coinapi.io/
- REST API: https://docs.coinapi.io/market-data/rest-api
- WebSocket API: https://docs.coinapi.io/market-data/websocket
- Last-verified: `<YYYY-MM-DD>` <!-- TODO: bump on each re-check -->

## When NOT to use this vendor

- **UI / display-grade prices only** → [[vendor-coingecko]] / [[vendor-coinmarketcap]] are cheaper.
- **On-chain settlement** → [[vendor-chainlink]].
- **Single-exchange data** → call that exchange directly.

## Cross-references

- Parent category: [[category-price-feeds]]
- Reviewed by: [[web3-backend-reviewer]] (§7 vendor integrations, §9 streaming / fallback)
- Alternatives: [[vendor-coingecko]], [[vendor-coinmarketcap]], direct exchange APIs, Kaiko
