---
name: category-smart-contract-testing
description: Entry point for testing and debugging smart-contract integrations from the backend — pre-flight simulation (Tenderly), fork testing (Foundry / Hardhat), ABI verification, virtual TestNets, and revert root-cause analysis. Covers vendor selection, DIY local testing (Foundry preferred), CI patterns, and runbooks for on-chain incident debugging. Use whenever the user mentions Tenderly, Foundry, Hardhat, fork tests, simulation, revert debugging, or smart-contract integration testing.
---

# Smart Contract Testing — Category

Entry point for any "will this transaction work?" or "why did this transaction fail?" workflow in a backend.

## When to use this skill

- Building pre-flight simulation into a tx-submission pipeline
- Setting up fork-based integration tests in CI
- Debugging a production revert
- Verifying an external contract's ABI before integrating
- Running incident response on a stuck or failed on-chain operation

## The use cases

| Use case | Typical answer |
|---|---|
| Pre-broadcast simulation of a user-submitted tx | [[vendor-tenderly]] simulation API |
| Reproduce a production revert locally | Foundry fork at the failing block |
| Integration test against real mainnet state | Foundry `forge test --fork-url` or [[vendor-tenderly]] virtual TestNets |
| Production alerting on contract events | [[vendor-tenderly]] alerts + Web3 Actions |
| Deployment + scripts | Hardhat or Foundry scripts |
| Fuzz / invariant testing | Foundry `forge test` (invariant + fuzz) |
| Unit tests, simple cases | Foundry `forge test` |

## Vendor options

| Vendor | Strength | Best for | Trade-offs | Backend skill |
|---|---|---|---|---|
| **Tenderly** | Hosted simulation, virtual TestNets, alerts | Pre-flight + production debugging + integration tests | EVM only; tiered pricing | [[vendor-tenderly]] |
| OpenZeppelin Defender | Hosted monitoring + admin / Autotasks | Multi-sig admin workflow + monitoring | Less raw simulation power | *(not in scaffold)* |
| Phalcon (Blocksec) | Adversarial replay + exploit analysis | Post-mortem of complex exploits | Narrower scope | *(not in scaffold)* |

## Build-it-yourself option

### Foundry (preferred for new code)
- `forge test` for unit + fuzz + invariant tests
- `forge test --fork-url $MAINNET_RPC` for fork tests
- `anvil` for local devnet
- `cast` for one-off RPC calls + debug
- Faster compilation, Solidity-native testing, no JS runtime

### Hardhat (legacy / TS-heavy teams)
- TypeScript test runner; Mocha/Chai
- Better deployment + plugin ecosystem (verify-on-Etherscan, gas reporter, hardhat-deploy)
- Slower test runs than Foundry

### When to use which
- **Greenfield contract tests** → Foundry.
- **Existing JS-heavy stack, deployment scripts** → Hardhat.
- **Both** (common) — Foundry for tests, Hardhat for deployment.

### Local Solana / Rust program testing
- **Anchor** (`anchor test`) — spins a local validator, runs your TS/Rust tests against the deployed program. Default for Anchor programs.
- **`litesvm`** — in-process SVM, dramatically faster than a validator; ideal for unit-testing program logic in tight loops.
- **`solana-program-test`** (`ProgramTest`) — Rust-native harness for instruction-level tests without a full validator.
- **`solana-test-validator`** — full local validator for end-to-end backend integration tests.
- Conform to [The Rust Book](https://doc.rust-lang.org/book/) + [Rust API Guidelines](https://rust-lang.github.io/api-guidelines/); test the failure paths that matter on-chain: **checked-math overflow**, missing **signer/owner checks**, wrong **PDA seeds**, and unexpected **CPI** targets. See [[wallet-security-auditor]] for the security lens.

## Backend best practices (inline)

### Pre-flight simulation in tx pipeline
1. Build tx
2. Submit to [[vendor-tenderly]] simulation API at current block
3. Inspect trace → if revert, surface reason to user; if gas > budget, reject
4. If simulation passes, broadcast
5. Watch receipt; on revert despite passing simulation, capture trace for diff (MEV between sim + broadcast)

### Fork tests in CI
- Pin fork to a specific block per test for reproducibility (avoid "passes on Monday, fails Tuesday").
- Use `vm.rollFork` to test across multiple block states in one run.
- Cache fork RPC responses locally (Foundry's cache dir) to avoid hammering provider.
- For [[vendor-tenderly]] virtual TestNets: tear down after CI run to control cost.

### Production debugging runbook
1. Get tx hash from logs / monitoring
2. Open in [[vendor-tenderly]] (or Etherscan / Solscan for quick view)
3. If reverted: trace → find reverting call → check storage state at that point
4. If succeeded but wrong result: trace + storage diff
5. For Solana: Solana Explorer + Anchor's `idl-parser` or Helius enhanced tx for parsed instructions

### Language idioms
- **Solidity tests:** Foundry's `Test.sol` patterns; use `vm.expectRevert`, `vm.expectEmit`, fuzz with `function testFuzz_X(uint256 a)`.
- **TypeScript backend tests against forks:** [viem](https://viem.sh) with a local anvil instance (`viem/test`).
- **Golang backend tests against forks:** `go-ethereum/ethclient` against anvil; or testcontainers-go for anvil in CI.
- **Rust:** Anchor program tests (`#[tokio::test]` + `litesvm`/`ProgramTest`); for EVM simulation use **`revm`** to execute against forked state in-process (the same engine MEV bots use — see [[category-mev-and-keepers]]); `proptest` for property-based coverage of program logic. Full Rust idioms in [[incremental-implementation]].

### Infra patterns
- **CI:** parallelize fork tests by test suite; cache `forge` artifacts; budget RPC quota per CI run.
- **Alerting on prod:** [[vendor-tenderly]] alerts → webhook → SQS / Pub/Sub → on-call routing (PagerDuty / Slack).
- **Logs:** structure revert reasons for fast triage. See [[web3-backend-reviewer]] §10 (error attribution).

### Chainlink / oracle mocking
- For fork tests with [[vendor-chainlink]] feeds: either pin fork block (feed state frozen) or mock the `AggregatorV3Interface` with `vm.mockCall`.
- Never deploy production code assuming a fork-time price holds in prod — simulate close to broadcast.

## Decision tree

1. **Pre-flight simulation in prod pipeline** → [[vendor-tenderly]] simulation API.
2. **Unit / fuzz / invariant tests** → Foundry.
3. **Fork integration tests** → Foundry `--fork-url`, or [[vendor-tenderly]] virtual TestNets.
4. **Deployment scripts on EVM** → Hardhat or Foundry scripts (`forge script`).
5. **Production revert post-mortem** → [[vendor-tenderly]] for one-off; Phalcon for complex exploits.
6. **Solana** → Anchor + `solana-test-validator`; [[vendor-tenderly]] is EVM-only.

## Cross-references

- Vendor: [[vendor-tenderly]]
- Related categories: [[category-rpc-and-indexer]] (RPC needed for forks), [[category-mev-and-keepers]] (private mempool simulation), [[category-multichain-tx-management]] (tx broadcast)
- Reviewed by: [[web3-backend-reviewer]] (§10 Debugging — Tenderly is the recommended tool)
- Security-reviewed by: [[wallet-security-auditor]] (signing flow tests)
