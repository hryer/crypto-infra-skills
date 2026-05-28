---
name: category-price-feeds
description: Entry point for token price feeds — off-chain price APIs (Coingecko, CoinMarketCap, CoinAPI, Alchemy Prices) vs on-chain oracles (Chainlink Data Feeds, Pyth, RedStone) vs DIY (TWAP from DEX pools). Covers vendor selection by use case (UI display vs on-chain settlement vs HFT), caching, staleness handling, and fallback architecture. Use whenever the user mentions token price, price feed, price oracle, market data, OHLCV, Coingecko, CoinMarketCap, Chainlink, or Pyth.
---

# Price Feeds — Category

Entry point for any "what does this token cost?" question in your backend.

## When to use this skill

- Displaying token prices in a UI
- Computing portfolio values
- Settling on-chain liquidations / swap minOut values (oracle-grade required)
- Backtesting / quant data
- Building a price alerting system

## The use cases

| Use case | Typical answer |
|---|---|
| UI display ("BTC = $X") | Off-chain API: [[vendor-coingecko]] / [[vendor-coinmarketcap]] |
| Portfolio value computation | Same as UI, with longer-cached prices |
| On-chain liquidation / swap settlement | On-chain oracle: [[vendor-chainlink]] (Data Feeds) or Pyth |
| HFT / market-making | Direct exchange WebSocket or [[vendor-coinapi]] |
| Solana memecoin price | [[vendor-birdeye]] |
| Historical OHLCV / backtest | [[vendor-coinapi]] or Kaiko |

## Vendor options

| Vendor | Type | Best for | Trade-offs | Backend skill |
|---|---|---|---|---|
| **Coingecko** | Off-chain REST | UI display, broad coverage | Heavy rate limits on free tier | [[vendor-coingecko]] |
| **CoinMarketCap** | Off-chain REST | Off-chain redundancy with Coingecko | Tiered, slightly less data depth | [[vendor-coinmarketcap]] |
| **CoinAPI** | Off-chain REST + WS | Institutional / near-HFT | Institutional pricing | [[vendor-coinapi]] |
| **Alchemy Prices** | Off-chain REST | If already using Alchemy for RPC | Smaller coverage than Coingecko | [[vendor-alchemy]] (Prices API) |
| **Birdeye** | Off-chain REST + WS | Solana token prices | EVM coverage is mixed | [[vendor-birdeye]] |
| **Chainlink Data Feeds** | On-chain oracle | On-chain settlement | Per-chain availability; staleness checks required | [[vendor-chainlink]] |
| Pyth | On-chain pull oracle | Lower-latency on-chain prices | Pull model adds caller cost | *(not in scaffold)* |
| RedStone | On-chain modular oracle | Long-tail on-chain prices | Less coverage than Chainlink for blue-chip | *(not in scaffold)* |

## Build-it-yourself option

### DIY: TWAP from on-chain DEX pools
- Read Uniswap V3 pool's TWAP oracle for a chosen window (5 min, 30 min, etc.).
- Resistant to short-term manipulation if the window is long enough.
- Use when no Chainlink feed exists for a long-tail token but it has deep DEX liquidity.
- **Pitfall:** thin liquidity → easy to manipulate even with TWAP. Always sanity-check vs an independent off-chain source.

### DIY: aggregating off-chain prices
- Pull from 2-3 sources (Coingecko + CMC + Birdeye), reject outliers, take median.
- Useful when no single vendor covers all your tokens reliably.

## Backend best practices (inline)

### Caching architecture
- **Redis** as the cache layer; tiered TTLs:
  - Volatile prices (BTC, ETH): 5-15s
  - Stablecoin pairs: 60s+
  - Long-tail tokens: 30-60s
  - Token metadata: hours/days
- Stale-while-revalidate so vendor outage doesn't break UI.
- Bulk-fetch endpoints over per-symbol when supported (Coingecko `/simple/price?ids=...`).

### Staleness handling (critical on-chain)
- Every Chainlink feed read must check `block.timestamp - updatedAt < HEARTBEAT * 1.5`.
- Off-chain price used for settlement: timestamp + age check at consume site.
- Alert when a watched feed is stale beyond threshold.

### Source isolation
- For UI: any source is fine; speed matters.
- For settlement: oracle only. Never mix display prices and settlement prices.
- For liquidations: layer oracle + manual circuit breaker on extreme deviation.

### Failure modes
- Primary vendor down → fall back to secondary (Coingecko → CMC → CoinAPI). See [[web3-backend-reviewer]] §9.
- Chainlink feed stale → halt liquidations until resolved; do NOT silently fall back to off-chain.
- WebSocket disconnect → exponential backoff reconnect; alert on stale-data threshold.

### Language idioms
- **TypeScript:** typed `fetch` wrappers; `pino` for structured logging; cache via `ioredis` or `@upstash/redis`.
- **Golang:** `net/http` + typed structs; `go-redis` for cache; bound contracts for on-chain reads. Follow [Effective Go](https://go.dev/doc/effective_go).

### Infra patterns
- **Redis:** primary cache + price stream pub/sub for fan-out.
- **Kafka:** for HFT-adjacent flows, stream price ticks into Kafka topics partitioned by symbol.
- **Postgres / TimescaleDB:** historical OHLCV storage if you backtest.

## Decision tree

1. **UI display only** → [[vendor-coingecko]] with [[vendor-coinmarketcap]] as fallback.
2. **On-chain settlement / liquidation** → [[vendor-chainlink]] Data Feeds, with manual deviation circuit breaker.
3. **Long-tail token, no Chainlink feed, deep DEX liquidity** → DIY TWAP from Uniswap V3 pool.
4. **Solana token discovery** → [[vendor-birdeye]].
5. **HFT / market making** → direct exchange WebSocket; [[vendor-coinapi]] as backup.
6. **Already using Alchemy for RPC, simple price needs** → [[vendor-alchemy]] Prices API; saves a vendor.

## Cross-references

- Vendors: [[vendor-coingecko]], [[vendor-coinmarketcap]], [[vendor-coinapi]], [[vendor-alchemy]], [[vendor-birdeye]], [[vendor-chainlink]]
- Related categories: [[category-onchain-analytics]] (Nansen for signal-grade data), [[category-rpc-and-indexer]] (DIY TWAP needs RPC)
- Reviewed by: [[web3-backend-reviewer]] (§1 correctness, §7 vendor integrations, §9 caching / fallback)
