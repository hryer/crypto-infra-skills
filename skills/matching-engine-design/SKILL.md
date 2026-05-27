---
name: matching-engine-design
description: Use when designing or reviewing a crypto/financial matching engine. Covers order book data structures, matching algorithms (price-time priority, pro-rata), Go-side architecture (hexagonal/clean), single-pair vs multi-pair, in-memory vs persistent, hot-path optimization, and concurrency patterns. Use whenever the user mentions order book, matching engine, exchange backend, CEX architecture, limit order, or order matching.
---

# Matching Engine Design

## When to use

Trigger this skill when the user asks to:
- Design a matching engine (CEX-style or AMM-adjacent)
- Pick data structures for an order book
- Architect concurrency for a single-pair engine
- Plan persistence + recovery for an in-memory engine
- Optimize the hot path (sub-microsecond matching)
- Add multi-pair support to an existing single-pair engine

## Process

### 1. Define the matching rules explicitly

Before any code:
- **Priority rule:** price-time (FIFO at price level) or pro-rata?
- **Order types supported:** limit, market, stop-limit, post-only, FOK, IOC?
- **Tick size and lot size:** how do you quantize prices and quantities?
- **Self-trade prevention:** cancel-newest, cancel-oldest, or decrement-both?
- **Fee model:** maker/taker rebates? Applied where (engine output or settlement)?

Write these as a one-page spec before touching code. They drive every design choice downstream.

### 2. Pick the order book data structure

For price-time priority matching:

```
OrderBook
├── BidSide:  sorted descending by price (best bid = highest)
│   └── PriceLevel
│       ├── price
│       └── orders: FIFO list of orders at this price
└── AskSide:  sorted ascending by price (best ask = lowest)
    └── PriceLevel (same shape)
```

**Implementation choices:**

| Structure | Insert | Best price | Trade-off |
|---|---|---|---|
| Red-black tree (price levels) + doubly-linked list (orders) | O(log P) | O(1) | Mature, predictable. Most production engines. |
| Skip list | O(log P) | O(1) | Easier to make concurrent |
| Sorted array | O(P) | O(1) | Only works if price range is small |
| HashMap<price, level> + sorted price list | O(log P) | O(1) | Common, with TreeMap or skip list for the price ordering |

P = number of price levels (typically 10s to 1000s, not millions).

**Practical pick:** `map[price]*PriceLevel` + a sorted slice or skip list of prices. Go's `sync.Map` is NOT what you want here — use a regular map with explicit locking, the access pattern is too structured for `sync.Map`'s tradeoffs.

### 3. Matching algorithm (price-time)

For an incoming buy order at price P:

```
1. Walk ask side from best (lowest) ask upward
2. For each ask level where ask_price <= P:
   3. For each order at this level (in FIFO order):
      4. match = min(incoming.remaining, resting.remaining)
      5. emit fill event (incoming, resting, match, level_price)
      6. decrement both
      7. if resting fully filled: remove from level
      8. if incoming fully filled: stop
   9. if level empty: remove level from ask side
10. if incoming has remainder and is limit: add to bid side at price P
    if remainder and is market: cancel (or partial fill report)
```

Two invariants to verify in tests:
- After matching, no resting bid >= any resting ask (the book is "uncrossed")
- Total quantity in = total quantity out + remaining resting

### 4. Hexagonal architecture (clean / ports & adapters)

```
                  ┌──────────────────────────┐
   Order API ──▶  │                          │  ──▶ Event sink (Kafka, etc.)
                  │   Domain: MatchingEngine │
   Admin API ──▶  │                          │  ──▶ State snapshot
                  └──────────────────────────┘  ──▶ Metrics
                            │
                            ▼
                  Pluggable storage (in-memory, Redis, etc.)
```

**Domain layer (pure Go, no I/O):**
- `OrderBook`, `Order`, `Fill`, `PriceLevel`
- `MatchingEngine.SubmitOrder(order) -> []Fill`
- No goroutines, no channels, no I/O — pure data manipulation

**Application layer:**
- Wraps the engine, handles concurrency (single goroutine per pair)
- Drains an input channel of commands, emits output channel of events
- Hooks: metrics, logging, persistence

**Adapter layer:**
- HTTP/gRPC server feeding the input channel
- Kafka/Redis consumer for replication or upstream sources
- Storage drivers for snapshots

The domain layer is where your tests live. It should run thousands of orders/sec with zero I/O.

### 5. Concurrency model

**The cardinal rule:** one engine per pair, one goroutine per engine.

```
Input channel ──▶ [Engine goroutine] ──▶ Output channel
                  │
                  └── owns the OrderBook exclusively
```

- No locks inside the engine (it's single-threaded by goroutine)
- All concurrency happens AT THE BOUNDARY (channels)
- Output channel drains to: persistence, market data publisher, audit log

**Why not multi-goroutine matching?**
- Lock contention on hot data structures kills throughput
- Total ordering of events becomes ambiguous (who matched first?)
- Easier to scale by sharding by pair than by parallelizing within a pair

**When you outgrow single-goroutine:**
- 99% of trading pairs never do (BTC-USD on Coinbase peaks ~50k orders/sec — fits one goroutine)
- If you really need it: pre-shard by price range, but you're now solving a much harder problem

### 6. Persistence and recovery

In-memory engines are fast but lose state on crash. Options:

**Event sourcing (recommended):**
- Append every command (order, cancel) to a durable log (Kafka, RocksDB WAL)
- On crash: replay from log to rebuild in-memory state
- Snapshot periodically to truncate replay time
- Fills/trades are derived, not stored separately

**Snapshot + replay:**
- Periodically dump full order book to disk
- Replay only events since last snapshot

**Trade-offs:**
- Event sourcing = perfect audit, but log grows
- Snapshots = bounded recovery time, but more complex to make consistent

For a single-pair engine: event sourcing + snapshot every N events. Replay should be <60s from snapshot.

### 7. Hot-path optimization

If you've measured a real bottleneck (and only then):

- **Pool allocations.** Use `sync.Pool` for `Order` and `Fill` structs.
- **Avoid interface dispatch in the hot loop.** Concrete types only.
- **Avoid `interface{}` boxing.** Specialize containers.
- **Cache-line friendliness.** Pack hot fields together, separate cold fields.
- **Atomic counters for metrics.** Don't call into Prometheus in the matching loop.
- **No goroutine spawning in the hot path.** Pre-spawn worker pools.

Don't do any of this before profiling. Premature optimization here usually makes code worse without measurable wins.

### 8. Multi-pair extension

Each pair = independent engine instance + goroutine + input/output channels.

```
Router (round-robin or pair-based hash)
   ├── BTC-USD engine
   ├── ETH-USD engine
   └── SOL-USD engine
```

Cross-pair concerns (e.g., position margin across pairs) live OUTSIDE the matching engines, in a separate risk service that consumes the output streams.

## Rationalizations to reject

| Excuse | Reality |
|---|---|
| "We need lock-free data structures for performance" | Channels + single goroutine per pair beats lock-free for 99% of crypto matching workloads. Less code, fewer bugs. |
| "Let's start with Postgres for the order book" | A relational DB in the matching hot path is a dealbreaker for any non-toy volume. In-memory + WAL is the standard. |
| "We'll add persistence later" | Adding event sourcing after the engine exists is a months-long retrofit. Design it in from day one. |
| "Pro-rata is fairer than price-time" | It's also slower to implement and harder to audit. Use price-time unless you have a specific reason for pro-rata. |
| "We don't need market orders, just limit" | Market orders are a UX requirement for retail. Just IOC-with-no-price under the hood. |

## Verification

Before considering the engine production-ready:
- [ ] Order book uncrossed invariant tested (property-based test with random orders)
- [ ] Conservation invariant tested (qty in = qty out + remaining)
- [ ] Crash recovery tested (kill mid-trade, restart, verify state)
- [ ] Snapshot + replay produces identical state to original
- [ ] Throughput benchmark documented (orders/sec, p50/p99/p999 latency)
- [ ] Self-trade prevention rule tested
- [ ] FOK/IOC/post-only types tested if supported
- [ ] Audit log: every event reproducible from input commands

## Reference architecture: your matching engine portfolio piece

If reviewing or extending a single-pair, in-memory, hexagonal Go matching engine:

**Strengths to highlight in portfolio context:**
- Hexagonal/ports-adapters = clean separation of domain from I/O
- Single goroutine per pair = correct-by-construction concurrency
- In-memory = realistic for production CEX hot path
- Go = idiomatic for backend infra roles

**Common gaps to address:**
- Event sourcing for recovery (often missing in portfolio pieces)
- Multi-pair sharding (next obvious extension)
- Market data publishing (book deltas, last trade, OHLC)
- Risk hooks (pre-trade checks: balance, max order size, kill switch)
