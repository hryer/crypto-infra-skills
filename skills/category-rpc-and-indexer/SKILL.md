---
name: category-rpc-and-indexer
description: Use when designing or reviewing a blockchain event indexer — webhook ingestion from Alchemy/QuickNode, Pub/Sub or Kafka fan-out, dead-letter queue (DLQ) recovery, idempotent processing, reorg handling, and backfill strategy. Covers vendor selection trade-offs and failure-mode design. Use whenever the user mentions indexer, event listener, webhook ingestion, blockchain data pipeline, reorg handling, or transaction tracking.
---

# Blockchain Indexer Design

## When to use

Trigger this skill when the user asks to:
- Design a blockchain event/transaction indexer
- Compare Alchemy vs QuickNode vs self-hosted node + indexer
- Add DLQ / retry / reorg recovery to an existing indexer
- Diagnose missing or duplicated events in a webhook pipeline
- Design a backfill strategy for historical data
- Architect a multi-chain indexer

## Process

### 1. Clarify the data contract

Ask before designing:
- Which chains? (EVM-only? Solana? TRON? Bitcoin?)
- Which events? (ERC-20 transfers, custom contract events, native transfers, internal calls, contract deployments)
- Latency requirement? (sub-second, seconds, eventual)
- Reorg tolerance? (how many confirmations before "finalized")
- Throughput? (events per second peak, not average)

The right architecture for a 10-event/second indexer is very different from one handling Uniswap-scale traffic.

### 2. Pick the ingestion layer

| Option | When to use | Avoid when |
|---|---|---|
| **Alchemy Webhooks** | Best DX, fast to start, mainstream chains | Cost > $1k/mo, need APAC latency, exotic chains |
| **QuickNode Streams** | Better filtering, multi-region, fair pricing | Less mature SDK than Alchemy |
| **Goldsky Mirror** | Real-time subgraph + DB sync | Want pull-based control |
| **The Graph (Substreams)** | Public dApp data, decentralized | Need private/custom logic |
| **Self-hosted (Erigon/Reth + custom)** | Cost > $5k/mo with vendor, exotic chains, data sovereignty | You don't have a node ops engineer |
| **WebSocket + eth_subscribe** | Cheap, simple, low-volume | Production — connections drop, you'll miss events |

**Decision rule:** start with Alchemy. Move off only when you've measured a real constraint (cost, latency, missing chain).

### 3. Design the durable buffer

The webhook handler must be **fast and dumb**:
- Receive webhook → push to durable queue → ACK in <200ms
- Never process synchronously inside the handler
- Vendors retry aggressively on timeout — you WILL double-process if you're slow

Recommended queue choices:
- **GCP Pub/Sub** — best for GCP-native stacks, at-least-once delivery, built-in DLQ
- **AWS SQS + SNS** — AWS-native equivalent
- **Kafka** — when you need replay, ordering, and high throughput
- **Redis Streams** — for low-volume, simple setups

### 4. Idempotency (mandatory, not optional)

You will get duplicate events. Plan for it.

**Idempotency key by chain:**
- EVM: `(chain_id, tx_hash, log_index)`
- Solana: `(signature, instruction_index, inner_instruction_index)`
- TRON: `(tx_id, contract_index)`
- Bitcoin: `(txid, vout)` for outputs, `(txid, vin)` for inputs

**Two-layer dedup:**
1. **Redis (hot path)** — `SET NX` with TTL of ~24h. Fast.
2. **DB (cold storage)** — UNIQUE constraint on the idempotency key. Source of truth.

If Redis says "already processed," skip. If Redis says "new" but DB constraint fires on insert, also skip (logged as a Redis cache miss). Both layers are needed.

### 5. DLQ design

After N retries (typically 5 with exponential backoff: 1s, 5s, 30s, 5min, 1hr), push to DLQ.

**DLQ rules:**
- DLQ consumer is a SEPARATE service with relaxed SLA
- DLQ triggers operator alerts, not auto-retry
- Poison messages will exhaust your main worker capacity if you keep retrying them inline

**What goes in the DLQ message:**
- Original event payload
- Last error + stack
- Retry count + first/last attempt timestamps
- Worker that processed it

Without the last error, debugging DLQ entries is hell.

### 6. Reorg handling

Most missed reorgs corrupt balances permanently. Handle this on day one.

**Track `block_hash` alongside `block_number`** in every record.

**Two-phase commit:**
1. **Pending phase** — record the event with status="pending"
2. **Confirmation phase** — after N confirmations, promote to status="confirmed"

**Confirmation thresholds (rule of thumb):**
- Ethereum mainnet: 12 blocks (~2.5 min)
- Polygon: 32 blocks (~1 min)
- BSC: 15 blocks (~45s)
- Arbitrum / Optimism: rely on L1 finalization, ~7 days for true safety, or 64 blocks for practical
- Solana: 32 slots (`finalized` commitment)

**On reorg detection:**
- If `block_hash` for a known `block_number` changes:
  - Mark all events at that block as "reorged"
  - Re-emit them through the normal processor (which will produce new events with the new block_hash)
  - DO NOT just update in place — downstream consumers must see the reorg

### 7. Backfill

You'll need backfill for:
- Cold-start (new contract to track)
- Recovery from indexer downtime
- Adding a new chain
- Discovering a bug in old processing

**Backfill via RPC `getLogs`:**
- Chunk by block range (typically 1000–10000 blocks per request)
- Honor rate limits — backfill should be 10-30% of your normal RPC capacity
- Use the SAME processor as live ingestion (idempotency makes this safe)

**Never write a separate backfill code path.** That's how you end up with backfilled data that doesn't match live data.

### 8. Multi-chain abstraction

Resist the urge to write one indexer service that handles all chains. Instead:

```
chain_ingestor (per chain) → unified queue → processor (chain-agnostic where possible)
```

Each ingestor knows its chain's quirks (EVM logs vs Solana logs vs TRON triggers). The downstream processor works on a normalized event format.

Don't normalize too aggressively — preserve chain-specific fields you'll need for debugging.

## Rationalizations to reject

| Excuse | Reality |
|---|---|
| "Webhooks are reliable, we don't need a DLQ" | Vendors have outages. Network blips happen. You will lose events without a DLQ. |
| "We can dedupe at the DB layer with UNIQUE constraints" | True, but you still burn worker capacity on every retry. Add Redis. |
| "Reorgs are rare on mainnet" | They're not rare on L2s. And one missed reorg corrupts user balances forever. |
| "We'll handle Solana later" | Solana's commitment model is fundamentally different from EVM finality. Design the abstraction now or rip it apart later. |
| "Our backfill is a one-off script, it doesn't need to be production-grade" | Until the next cold-start. Until the next migration. Until the next bug. |
| "We can use WebSocket subscriptions for production" | WS connections drop. The provider doesn't replay missed events on reconnect. You will lose data. |

## Verification

Before considering the design complete:
- [ ] Webhook handler returns in <200ms (no DB writes on hot path)
- [ ] Idempotency key documented for each supported chain
- [ ] Duplicate-replay test passes (re-send the same event, processed exactly once)
- [ ] DLQ exists with documented operator runbook
- [ ] Reorg recovery tested on a testnet (forced fork via Hardhat or Anvil)
- [ ] Backfill and live processing share the same handler code
- [ ] Multi-chain: each chain has its own ingestor; processor is chain-aware but unified
- [ ] Monitoring: lag (block height vs chain head), DLQ size, retry rate are all metrics

## References

- `references/alchemy-vs-quicknode.md` — vendor comparison
- `references/pubsub-dlq-patterns.md` — GCP Pub/Sub DLQ patterns with NestJS examples
