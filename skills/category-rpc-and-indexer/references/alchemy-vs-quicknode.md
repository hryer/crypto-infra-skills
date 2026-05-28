# Alchemy vs QuickNode — Indexing Provider Comparison

Comparison from a production indexer-builder's perspective, not marketing.

## Quick verdict

- **Default to Alchemy.** Best DX, most mature, biggest ecosystem.
- **Switch to QuickNode** if you need multi-region (especially APAC) or hit a specific Alchemy limit.
- **Consider Goldsky/The Graph** if you're indexing public dApp data and don't need private logic.
- **Self-host** only when monthly bill > $5k and you have a node ops engineer.

## Feature matrix

| Feature | Alchemy | QuickNode |
|---|---|---|
| EVM chains | All majors + most L2s | Slightly broader L2 coverage |
| Non-EVM | Solana, Starknet | Solana, TRON, Bitcoin, more |
| Webhook product | Notify (address activity, custom GraphQL) | Streams (functions + filters) |
| Webhook filter complexity | GraphQL queries — powerful but verbose | Function-based, JS-like |
| Webhook retry semantics | 5 retries, exponential backoff | Configurable |
| Region availability | US primary; secondary EU | Multi-region including APAC |
| Archive node access | Included on higher tiers | Add-on |
| Trace/debug APIs | Yes (eth_call traces, debug_traceTransaction) | Yes |
| Free tier | Generous compute units | Slightly stingier |
| Pricing model | Compute units | Credits + add-ons |

## Webhook product details

### Alchemy Notify

**Strengths:**
- Address Activity webhooks are dead simple ("notify me when this address sends/receives anything")
- Custom GraphQL webhooks let you filter by very specific event signatures
- SDK is well-documented, lots of community examples
- Signature verification headers are clean

**Weaknesses:**
- GraphQL filtering syntax is verbose for simple cases
- Webhook dashboard UI gets clunky at >50 webhooks
- US-region latency: ~150-300ms RTT from Jakarta

### QuickNode Streams

**Strengths:**
- Function-based filtering (write JS to decide what to forward) is more flexible
- Multi-region: deploy webhooks in APAC region, much lower latency
- Supports more chains out of the box

**Weaknesses:**
- Smaller community, fewer tutorials
- The Functions sandbox has cold starts that can add latency
- Pricing model is harder to reason about (credits + Functions execution time + bandwidth)

## Cost modeling

### Rough monthly cost for: 1M events/month, 5 chains, moderate filtering

| Provider | Estimated cost | Notes |
|---|---|---|
| Alchemy | $500-1500 | Depends on compute units used by GraphQL filters |
| QuickNode | $400-1200 | Depends on Function execution time |
| Self-host (Erigon ETH + Solana RPC + ops) | $1500-3000 | 3x dedicated servers + engineer time |

Self-hosting only beats vendors at much higher volumes. For most products under $10k/mo spend, vendor cost beats engineering cost.

## When the vendors fail you

### Outages
Both have had multi-hour outages in the past 24 months. Plan accordingly:
- Don't have all your indexing on one provider
- Have a backfill mechanism that can recover from any 24h gap
- Monitor "blocks behind head" as a SLO

### Specific chains
- **Solana high TPS days:** both providers throttle. You may see lag during NFT mints.
- **Polygon reorgs:** webhook reorg handling is incomplete on both. You'll need to detect and recover yourself.
- **New L2s:** support lags by weeks/months. If you need day-1 support, you'll be self-hosting.

### Geographic latency
If your indexer service is in Singapore/Jakarta and you're using Alchemy's US webhooks:
- Webhook RTT: 200-300ms
- This eats into your <200ms response budget
- Solutions: (a) move indexer service to US-East, (b) switch to QuickNode APAC, (c) buffer locally and ack vendor immediately

## Migration considerations

Switching providers is non-trivial:
- Webhook filter syntax differs
- Signature verification differs
- Event payload formats differ slightly (block_hash naming, log structure)
- Reorg semantics differ

**Recommendation:** wrap vendor SDKs behind your own normalized interface from day one. `IndexerEvent` should be your own type, not a vendor type. Then switching = swap the adapter, not rewrite the pipeline.

## Anti-patterns

- **Don't** rely on the vendor's "exactly-once" claims. They're at-least-once. Dedup yourself.
- **Don't** filter aggressively at the vendor level to save money if it complicates debugging. Receive more, filter in your code.
- **Don't** use eth_subscribe WebSockets for production. Reconnects lose events.
- **Don't** use vendor's pause/resume for maintenance. Use your own DLQ + replay.
