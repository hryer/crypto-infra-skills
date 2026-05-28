---
name: vendor-chainlink
description: Backend integration guide for Chainlink — decentralized oracle network covering Data Feeds (price oracles), VRF (verifiable randomness), CCIP (cross-chain messaging), Automation (keepers), and Functions (off-chain compute). Covers on-chain consumption, off-chain monitoring, gas-funding, and integration patterns. Use whenever the user is integrating Chainlink price feeds, VRF, CCIP, Automation, or Functions in a smart-contract-adjacent backend.
---

# Chainlink

Decentralized oracle network. Multiple products: Data Feeds, VRF, CCIP, Automation, Functions. Primary vendor under [[category-price-feeds]] for on-chain settlement; also referenced from [[category-smart-contract-testing]] when forking mainnet (Chainlink feeds need to be mocked or pinned).

## What this vendor is for

Chainlink is the standard for **on-chain** oracle services. Where Coingecko / CMC are off-chain informational APIs, Chainlink Data Feeds are signed by a decentralized oracle network and consumed directly by smart contracts as the trusted price for liquidations, swaps, and settlement.

**Products covered here:**
- **Data Feeds** — price oracles consumed on-chain (`AggregatorV3Interface`)
- **VRF (Verifiable Random Function)** — provably random numbers on-chain
- **CCIP (Cross-Chain Interoperability Protocol)** — token transfers + arbitrary messages across chains
- **Automation** — keeper / time-triggered on-chain function execution
- **Functions** — call off-chain APIs from a smart contract

## Custody / data / pricing model

- **Custody model:** N/A (oracle service).
- **Pricing:** Per-request LINK fees (varies per product). VRF, Automation, Functions need a funded subscription. Data Feeds are free to read on-chain.

## On-chain consumption (Solidity / contract side)

```solidity
// Data Feeds
import {AggregatorV3Interface} from "@chainlink/contracts/src/v0.8/shared/interfaces/AggregatorV3Interface.sol";
AggregatorV3Interface internal priceFeed; // e.g. ETH/USD on the target chain
(, int256 price, , uint256 updatedAt, ) = priceFeed.latestRoundData();
require(block.timestamp - updatedAt < STALE_THRESHOLD, "stale oracle");
```

**Critical:** always check `updatedAt` staleness — a frozen feed has caused liquidation cascades on multiple protocols.

<!-- TODO: paste reference snippets for VRF, CCIP, Automation, Functions -->

## Off-chain / backend integration

### Monitoring feed health
- Subscribe to feed events via [[vendor-alchemy]] / [[vendor-quicknode]] webhooks or `eth_getLogs`.
- Alert when `updatedAt` is older than the feed's documented heartbeat × 1.5.
- Alert when reported price deviates more than feed's deviation threshold from your reference source.

### Gas-funding subscription (VRF / Automation / Functions)
- Backend job tops up the subscription's LINK balance when below threshold.
- Use [[vendor-fireblocks]] / [[vendor-fordefi]] / Safe multisig for the funding wallet — not an EOA with hot key.

### CCIP message tracking
- Persist outbound CCIP message IDs.
- Poll destination chain or use Chainlink CCIP Explorer API for delivery status.
- Reconciliation job for stuck messages.

## SDK usage

### TypeScript

- For reading feeds: use [viem](https://viem.sh) with the `AggregatorV3Interface` ABI.
- For CCIP backend tracking: REST against Chainlink CCIP Explorer API.

<!-- TODO: paste viem readContract example against AggregatorV3Interface -->

### Golang

- Reading feeds: `go-ethereum`'s `bind.NewBoundContract` against the ABI.
- CCIP / Automation monitoring: REST clients.

<!-- TODO: paste go-ethereum bound-contract latestRoundData example -->

Follow [Effective Go](https://go.dev/doc/effective_go) — context-first on every chain read, error wrapping.

## Common integration mistakes

- Consuming Chainlink feeds without staleness checks — frozen feed = wrong price = liquidation cascade.
- Using Chainlink feed for liquidation but no circuit breaker if deviation > X% in Y seconds.
- VRF / Automation subscription underfunded → silently stops triggering.
- CCIP without retry-on-revert handling on destination → tokens stuck.
- Forking mainnet for tests without pinning Chainlink feed block / mocking the feed.

<!-- TODO: extend -->

## MCP integration

<!-- TODO: check Chainlink MCP availability. -->

## Latest docs reference

- Official docs: https://docs.chain.link/
- Data Feeds: https://docs.chain.link/data-feeds
- VRF: https://docs.chain.link/vrf
- CCIP: https://docs.chain.link/ccip
- Automation: https://docs.chain.link/chainlink-automation
- Functions: https://docs.chain.link/chainlink-functions
- Last-verified: `<YYYY-MM-DD>` <!-- TODO: bump on each re-check -->

## When NOT to use this vendor

- **Off-chain UI prices** → [[vendor-coingecko]] / [[vendor-coinmarketcap]] are cheaper and good enough.
- **Single-chain non-financial randomness** → block-hash-based RNG may suffice (insecure for high-value flows).
- **High-frequency on-chain prices** → Pyth (pull oracle) may have lower latency for some markets.
- **Generic cross-chain bridging of value (not arbitrary messages)** → use a token bridge ([[vendor-lifi]] aggregator) rather than CCIP for swap flows.

## Cross-references

- Parent category: [[category-price-feeds]] (Data Feeds)
- Related categories: [[category-smart-contract-testing]] (mock / pin in tests), [[category-multichain-tx-management]] (CCIP for cross-chain)
- Reviewed by: [[web3-backend-reviewer]] (§1 correctness, §7 vendor integrations, §10 debugging)
- Alternatives: Pyth (pull oracle), RedStone (modular oracle)
