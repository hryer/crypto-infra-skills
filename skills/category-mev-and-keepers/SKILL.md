---
name: category-mev-and-keepers
description: Use when designing MEV searcher bots, keeper bots, liquidation bots, or any backend service that monitors mempool/state and submits time-sensitive transactions. Covers mempool monitoring, opportunity detection, profitability calculation, Flashbots/MEV-Boost bundle submission, private mempools, and Go-side architecture patterns. Use whenever the user mentions MEV, arbitrage bot, keeper, liquidation bot, sandwich attack, frontrunning, Flashbots, or mempool monitoring.
---

# MEV & Keeper Bot Architecture

## When to use

Trigger this skill when the user asks to:
- Design an arbitrage / MEV searcher bot
- Build a keeper bot (liquidation, oracle update, vault rebalance, batch settlement)
- Choose between public mempool and private order flow
- Architect Flashbots / MEV-Boost bundle submission
- Optimize submission latency end-to-end
- Decide between Go vs Rust for the execution layer

## The categories

| Bot type | Trigger | Profit source | Latency requirement |
|---|---|---|---|
| **Arbitrage** | Price discrepancy across DEXes/CEXes | Spread | Single block (sub-second) |
| **Liquidation keeper** | Under-collateralized position | Liquidation incentive (5-15%) | Single block |
| **Oracle keeper** | Stale price feed | Update reward | Block-level OK |
| **Vault rebalancer** | Imbalanced LP position | Rebalance fee | Block-level OK |
| **Settlement keeper** | Batch ready to settle | Settlement fee | Block-level OK |
| **Sandwich** | Large pending swap | Frontrun + backrun | Sub-millisecond |

Latency requirements determine architecture. A liquidation bot can use a normal RPC. A sandwich bot needs co-located infrastructure.

## Process

### 1. Define the opportunity precisely

Before coding, write down:
- **Detection input:** what event/state change triggers the opportunity?
- **Detection latency budget:** from event to detection, how many ms?
- **Profitability formula:** revenue - (gas cost + bribe + slippage)
- **Failure modes:** what makes the opportunity disappear between detect and execute?

If you can't write this down, you can't build the bot.

### 2. Choose the data source

**Mempool monitoring (pending txs):**
- Public mempool via WebSocket `eth_subscribe("newPendingTransactions")`
- BloXroute / Eden / Chainbound for private mempool access
- Trade-off: public is free + crowded, private is paid + cleaner

**State monitoring (confirmed state):**
- Block subscription + state diff calculation
- Faster than re-reading state every block

**Hybrid:** most production bots use both. Mempool for "what's about to happen," state for "what just happened."

### 3. Detection architecture

For sub-second bots (Go example):

```
RPC WebSocket ──> Parser (decode tx) ──> Strategy filter ──> Profitability calc
                                                                    │
                                                                    ▼
                                                            Builder (sign tx/bundle)
                                                                    │
                                                                    ▼
                                                            Submitter (Flashbots, public)
```

Each stage should be a goroutine with a channel between them. Backpressure flows naturally.

Key Go patterns:
- **Channels for stages**, not for the whole pipeline. Buffered to absorb bursts.
- **Context cancellation** to abort work when opportunity expires
- **Worker pools** with bounded concurrency (don't spawn unbounded goroutines)
- **Atomic counters** for hot-path metrics (avoid Prometheus call overhead)

### 4. Profitability calculation (the hard part)

```
Profit = Revenue - GasCost - BribeAmount - Slippage - InventoryCost

Where:
  Revenue          = the value extracted by the opportunity
  GasCost          = tx_gas_used * effective_gas_price
  BribeAmount      = block.coinbase.transfer(...) to incentivize inclusion
  Slippage         = AMM slippage on swaps (use simulator, not estimates)
  InventoryCost    = cost of capital tied up + risk of being sandwiched yourself
```

**Critical:** simulate on a forked node, don't estimate. Foundry's `cast` or Anvil fork mode does this. Live RPC `eth_call` works for read-only sim but is slower.

**Bribe sizing:** bribe must exceed competitors' bribes. Track historical block winners to learn the going rate. Don't bribe so high that profit < 0.

### 5. Submission strategies

**Public mempool (default):**
- Cheapest, slowest, most exposed to frontrunning
- Use for low-competition opportunities (oracle keepers, vault rebalances)

**Flashbots / MEV-Boost bundles:**
- Submit a bundle (sequence of txs) to block builders
- Builders include it atomically or not at all
- You pay via `coinbase.transfer()` (the bribe) — only if bundle lands
- Use for: arbitrage, liquidations, anything competitive

**Private order flow services:**
- BloXroute Cloud-API, Eden Relay, etc.
- Faster propagation, sometimes exclusive routes to builders
- Cost money

**Bundle submission rules:**
- Target multiple relays (Flashbots, BloXroute, Eden, etc.) — submit same bundle to all
- Sign with the same key across relays (some relays whitelist senders)
- Watch the `eth_callBundle` response for simulation results before paying gas

### 6. Failure handling

The bot must handle:
- **Bundle not included** — opportunity moves to next block. Resubmit? Or abandon?
- **Tx reverts on-chain** — simulation said profit, reality said revert. Investigate. Add to known-bad list.
- **State changed between sim and submit** — opportunity gone. Cancel pending submissions.
- **Nonce gap** — your previous tx is stuck. Replace-by-fee or use 2D nonces with smart accounts.
- **Reorg ate your win** — the block containing your tx got reorged. Profit reverted. Track and re-attempt.

### 7. Keeper-specific: incentive economics

Keeper bots only make sense if:
- The keeper reward > gas cost + opportunity cost of running infra
- There's enough call frequency to amortize infra cost
- You're competitive with other keepers (or the protocol has anti-MEV keeper auctions)

Many "keeper opportunities" advertised by protocols are loss-making for solo operators. Run the math before building.

### 8. Go vs Rust vs Node.js

| Language | When | Tradeoff |
|---|---|---|
| **Go** | Default. Good concurrency, easy ops, fast enough for ~10ms decision loops | Not the absolute fastest |
| **Rust** | Sub-millisecond loops, sandwich bots, top-of-block competition | Steeper dev velocity |
| **Node.js** | Prototyping, keeper bots with 100ms+ latency budget | Worst tail latency |

For your level of opportunity (arbitrage, liquidations), Go is the sweet spot. Don't reach for Rust unless you've measured Go failing.

**If you do go Rust** (sub-ms loops, top-of-block competition), the ecosystem is mature and worth knowing:
- **`revm`** — the EVM in Rust; the standard for fast local simulation of candidate bundles (orders of magnitude faster than round-tripping `eth_call` to a node). Simulate, score, discard losers, only submit winners.
- **`reth`** — Rust execution client; pairs with revm for self-hosted, low-latency state access (kills the 50-200ms public-RPC penalty noted below).
- **`alloy`** — the modern Rust Ethereum library (types, providers, signers); successor to `ethers-rs`.
- Conform to [The Rust Book](https://doc.rust-lang.org/book/) + [Rust API Guidelines](https://rust-lang.github.io/api-guidelines/); `Result` + `?` over `unwrap` even in a bot (a panic mid-bundle = missed block). See [[incremental-implementation]] for idioms and [[category-smart-contract-testing]] for simulating bundles.

## Rationalizations to reject

| Excuse | Reality |
|---|---|
| "I'll start with public mempool, optimize later" | If you're profitable on public mempool, someone is going to beat you to it. Plan the Flashbots integration upfront. |
| "Simulation is too slow, I'll estimate" | Estimation errors cause failed txs. Failed txs cost gas with zero revenue. Simulate. |
| "I'll use Infura/Alchemy for everything" | Public RPCs are 50-200ms slower than self-hosted or dedicated nodes. For MEV, that's the difference between win and loss. |
| "Bribes are unethical" | Bribes (priority fees / coinbase transfers) are how Ethereum allocates block space. Not bribing means not getting included. |
| "I'll worry about reorgs after launch" | Reorgs cost real money. Track reorg-killed wins and discount your reported P&L by them. |

## Verification

Before going live:
- [ ] Backtest against historical mempool/block data
- [ ] Simulate on a forked mainnet (Anvil / Hardhat fork) end-to-end
- [ ] Dry-run on testnet (limited utility but catches dumb bugs)
- [ ] Monitor: opportunities detected, bundles submitted, bundles included, gross profit, net profit, reorg-killed wins
- [ ] Kill switch: ability to stop submission immediately without ending the process
- [ ] Position limits: max inventory at risk, max gas spend per hour
- [ ] Alerts: when net P&L goes negative for N consecutive hours

## References

- `references/flashbots-bundle-submission.md` — Flashbots bundle API + Go example
