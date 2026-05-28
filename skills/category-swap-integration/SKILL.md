---
name: category-swap-integration
description: Entry point for any swap / DEX integration in a crypto product — cross-chain aggregators (Li.Fi), single-chain aggregators (1inch, 0x, Paraswap, Jupiter for Solana), and direct DEX router calls (Uniswap, PancakeSwap). Covers vendor selection, DIY direct-router patterns, slippage / MEV protection, status tracking, and backend best practices. Use whenever the user mentions swap, DEX integration, aggregator, bridge, cross-chain swap, in-app swap, or Li.Fi.
---

# Swap Integration — Category

Entry point for any in-app or backend-initiated token swap, including cross-chain bridges.

## When to use this skill

- Integrating in-app swap (user-facing) or backend swap (e.g., auto-rebalance, fee collection)
- Cross-chain swap (token on chain A → token on chain B)
- Choosing between aggregator and direct DEX router
- Designing slippage / MEV / fail-safety policy for swap flows
- Adding swap status tracking + reconciliation to your backend

## The use cases

| Use case | Typical answer |
|---|---|
| Cross-chain swap (any chain ↔ any chain) | Aggregator: [[vendor-lifi]] |
| Single-chain EVM swap, best price across DEXes | Aggregator: 1inch / 0x / Paraswap |
| Solana swap | Aggregator: Jupiter |
| You only ever swap on Uniswap | Direct router call (skip aggregator fee) |
| Backend-initiated treasury rebalance | Direct router with strict slippage cap |

## Vendor options

| Vendor | Coverage | Best for | Trade-offs | Backend skill |
|---|---|---|---|---|
| **Li.Fi** | Cross-chain (EVM + Solana + more), aggregates DEXes + bridges | Default for multi-chain or cross-chain swap | Takes integrator fee; status is poll-based | [[vendor-lifi]] |
| 1inch | EVM only, DEX aggregator | Single-chain EVM, best-price for swap | No cross-chain; rate limits on public API | *(not in scaffold)* |
| 0x | EVM only, DEX aggregator + API-as-a-service | Lots of OTC + RFQ liquidity | Pricing tier for production | *(not in scaffold)* |
| Paraswap | EVM only, DEX aggregator | Solid EVM alternative to 1inch | Smaller routing graph in some markets | *(not in scaffold)* |
| Jupiter | Solana only, DEX aggregator | Default Solana swap | No EVM | *(not in scaffold)* |

## Build-it-yourself option

When you skip the aggregator and call a DEX router directly:

| Scenario | Approach |
|---|---|
| Uniswap V3 (EVM) | Call `SwapRouter02` with explicit pool + fee tier; or `UniversalRouter` for path bundling |
| Uniswap V2 forks (PancakeSwap, SushiSwap, etc.) | Standard V2 router (`Router02`) ABI |
| Curve | Pool-specific (different pool types have different ABIs); use Curve's router for newer integrations |
| Balancer | Vault contract; encoded swap struct |
| Solana (Raydium, Orca) | Program-specific instructions; assemble + sign manually or via Jupiter |

**Use DIY when:**
- You always swap on the same DEX and don't need best-price discovery
- Aggregator's integrator fee is meaningful at your volume
- You need deterministic execution (no aggregator black-box routing changing day-to-day)

**Skip DIY when:**
- You need multi-DEX best execution
- You need cross-chain
- Engineering time on swap > the aggregator fee at your volume

## Backend best practices (inline)

### Slippage and MEV protection
- Set max slippage explicitly; never default to "0.5%" without thought — for stablecoin pairs use 0.05%, for low-liquidity meme tokens 5%+.
- For institutional flows: submit via private mempool ([[category-mev-and-keepers]] for Flashbots-style submission) to avoid sandwich attacks.
- For high-value swaps: simulate with [[vendor-tenderly]] before broadcast.

### Quote freshness
- Quotes go stale fast (seconds for volatile pairs). Refresh just before sign / submit, not at UI render.
- Persist the chosen quote ID + expected output; reject the swap if executed output differs by more than X% from the quote.

### Status tracking
- For aggregators (Li.Fi): poll Status API; persist `(swap_id, source_tx, target_tx, status)`.
- For direct router calls: watch for tx receipt → parse swap event → update domain state.
- Reconciliation job: scan for swaps in non-terminal status older than N minutes; investigate.

### Language idioms
- **TypeScript:** prefer [viem](https://viem.sh) for new EVM swap code (typed ABIs prevent encoding mistakes). For Solana, `@solana/web3.js` or `@solana/kit`.
- **Golang:** `go-ethereum/ethclient` + generated bindings (`abigen`) for type-safe router calls. Follow [Effective Go](https://go.dev/doc/effective_go).

### Infra patterns
- **AWS SQS / GCP Pub/Sub / Kafka:** buffer aggregator webhook / status events (if vendor supports webhooks).
- **Redis:** cache quote responses with short TTL (1-5s); cache token metadata long.
- **Postgres:** persist swap intents with idempotency key (user-id + nonce); never lose track of an in-flight swap.

### Failure modes
- Aggregator API down → fallback to direct router for known pairs, or fail-fast with clear UX.
- Source chain RPC down → see [[category-rpc-and-indexer]] fallback strategy.
- Bridge stuck (cross-chain leg pending > expected) → alert + manual ops procedure.

## Decision tree

1. **User selects token A → wants token B, on same chain, EVM** → aggregator (Li.Fi or 1inch).
2. **User selects token A on chain X → wants token B on chain Y** → cross-chain aggregator ([[vendor-lifi]]).
3. **User on Solana** → Jupiter.
4. **Backend treasury swap, known pair, high volume** → direct router (Uniswap V3 / Curve) with strict slippage cap.
5. **High-value (>$100k) swap on EVM** → private mempool submission; simulate first.

## Cross-references

- Vendor: [[vendor-lifi]]
- Related categories: [[category-multichain-tx-management]] (tx submission), [[category-mev-and-keepers]] (private mempool), [[category-smart-contract-testing]] (Tenderly simulation)
- Reviewed by: [[web3-backend-reviewer]] (§1 correctness, §7 vendor integrations, §10 debugging swap failures)
