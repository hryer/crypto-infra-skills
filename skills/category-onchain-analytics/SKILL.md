---
name: category-onchain-analytics
description: Entry point for on-chain analytics — labeled wallet data (Nansen), raw SQL access (Dune), illicit-flow tracing (Arkham, Chainalysis), and DIY self-hosted indexer + analytics. Covers vendor selection by use case (smart-money signal vs compliance vs research), and architecture for analytics-driven products and trading signals. Use whenever the user mentions on-chain analytics, smart money, wallet labels, Nansen, Dune, Arkham, fund tracking, or quant signal.
---

# On-Chain Analytics — Category

Entry point for "what's happening on chain?" beyond raw RPC reads.

## When to use this skill

- Generating trading signals from labeled wallet activity
- Tracking fund / venture / smart-money positions for copy-trading or fund reporting
- Computing protocol-level metrics (TVL, fees, active users) for dashboards
- Compliance / illicit-flow investigation
- Backtesting / historical research

## The use cases

| Use case | Typical answer |
|---|---|
| Smart-money signal generation | [[vendor-nansen]] |
| Raw SQL on indexed chain data | Dune Analytics |
| Illicit-flow / OFAC compliance | Chainalysis / Arkham |
| Protocol metrics for a dashboard | DefiLlama (free) or DIY indexer |
| Quant backtest | Dune (raw) or Kaiko |
| Specific contract / address activity | Direct RPC + own indexer |

## Vendor options

| Vendor | Strength | Best for | Trade-offs | Backend skill |
|---|---|---|---|---|
| **Nansen** | Labeled wallets, smart-money signals | Trading signals, fund tracking | Expensive API tier | [[vendor-nansen]] |
| Dune Analytics | SQL on indexed chain data | Custom queries, dashboards | API access tiered; query latency varies | *(not in scaffold)* |
| Arkham | Wallet labels + entity attribution | Investigation, illicit-flow | Different label set than Nansen | *(not in scaffold)* |
| Chainalysis | Compliance + sanctions screening | OFAC compliance, KYT | Enterprise pricing | *(not in scaffold)* |
| Glassnode | Macro on-chain metrics (BTC heavy) | Macro / market research | Heavier on BTC, lighter on alt | *(not in scaffold)* |
| DefiLlama | Protocol TVL + fees | Public dashboards, free | No granular wallet data | *(not in scaffold)* |

## Build-it-yourself option

When you need exact data, no vendor latency, or to avoid recurring fees:

### Self-hosted indexer
- See [[category-rpc-and-indexer]] for indexer architecture.
- Index specific contracts / programs you care about; emit structured events to Postgres / ClickHouse.
- Build labeled views on top (your own "smart money" definition).

### When DIY makes sense
- You have a narrow set of contracts to index (e.g., your own protocol).
- You need data the vendor doesn't surface (custom metrics).
- Recurring vendor fees > engineering cost.
- You need millisecond latency that polling an API can't deliver.

### When DIY is overkill
- You need broad coverage across many protocols.
- You need labels (Nansen / Arkham took years to build).
- Your usage volume is small.

## Backend best practices (inline)

### Architecture
- **Read-replica DBs:** analytics queries are heavy; keep them off the OLTP database.
- **ClickHouse / TimescaleDB:** when queries are time-series / OLAP shaped, dedicated columnar store.
- **Materialized views:** pre-compute common dashboards; refresh on cadence.

### Caching + freshness
- Cache vendor responses in Redis; vendor labels rarely change.
- For real-time signals: WebSocket / poll → Kafka → signal engine → trading system.
- For dashboards: hourly / daily refresh is usually fine.

### Failure modes
- Vendor API down → graceful degradation for trading signals (reduce position, don't miss entirely).
- Indexer fell behind → alert on lag; pause downstream consumers if a strict freshness SLA is breached.
- Bad data: vendor labels are heuristic; never treat as ground truth for high-stakes decisions.

### Language idioms
- **TypeScript:** typed fetch wrappers; for heavy analytics consider Node worker threads or moving to a dedicated service.
- **Golang:** strong fit for analytics ingestion pipelines (concurrency, low overhead). Follow [Effective Go](https://go.dev/doc/effective_go).

### Infra patterns
- **Kafka:** core to high-volume on-chain event ingestion + fan-out to multiple consumers (trading, dashboards, alerting).
- **AWS S3 / GCP GCS:** raw event archive for replay / new analytics.
- **Postgres + Redis + ClickHouse:** common trio for analytics product.

## Decision tree

1. **You need smart-money trading signals** → [[vendor-nansen]].
2. **You need to run custom SQL on chain data** → Dune.
3. **OFAC / sanctions compliance** → Chainalysis.
4. **Your own protocol's metrics** → DIY indexer ([[category-rpc-and-indexer]]).
5. **Public dashboard / TVL only** → DefiLlama.
6. **Investigation / forensics** → Arkham.

## Cross-references

- Vendor: [[vendor-nansen]]
- Related categories: [[category-rpc-and-indexer]] (DIY indexer), [[category-matching-engine]] (signal consumer), [[category-mev-and-keepers]] (signal-driven bots)
- Reviewed by: [[web3-backend-reviewer]] (§7 vendor integrations, §9 fallback strategy)
