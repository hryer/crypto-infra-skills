---
name: vendor-tenderly
description: Backend integration guide for Tenderly — transaction simulation, smart-contract debugging, virtual TestNets, alerts, and Web3 actions. Covers simulation API, gas profiler, fork environments, alerting rules, and the Tenderly MCP server. Use whenever the user is integrating Tenderly for pre-flight tx simulation, on-chain incident debugging, integration testing on forks, or production alerting on contract events.
---

# Tenderly

Smart-contract observability platform: simulation, debugging, forks, alerts. Primary vendor under [[category-smart-contract-testing]] and the standard tool for §10 (Debugging) reviews in [[web3-backend-reviewer]].

## What this vendor is for

Tenderly answers "what will this transaction do" and "what did this transaction do" — pre-flight simulation, post-mortem trace inspection, virtual TestNets for integration testing, and alerts that fire on on-chain events. For any backend that submits non-trivial transactions to L1/L2 EVM chains, Tenderly is the de facto debugging tool.

## Custody / data / pricing model

- **Custody model:** N/A (observability + simulation).
- **Pricing:** Tiered (free → enterprise). Simulation, virtual TestNets, and alert volume are the main metered axes.

## Auth & API setup

- API access key + account / project slug. Source key from KMS / Vault.
- Per-environment projects (sandbox / mainnet) keep alerts and forks isolated.

<!-- TODO: fill in concrete auth steps from https://docs.tenderly.co -->

## SDK usage

### TypeScript

Official SDK: `@tenderly/sdk` (or REST directly).

<!-- TODO: paste minimal simulation example: simulate a tx, parse the trace, extract revert reason -->

### Golang

<!-- TODO: confirm Go SDK. If REST-only, thin client; parse trace JSON into typed structs. -->

Follow [Effective Go](https://go.dev/doc/effective_go) — context-first, error wrapping, parse trace responses defensively (deeply nested JSON).

## Use cases

### 1. Pre-flight simulation
Before broadcasting a tx, POST it to Tenderly's simulation API → inspect trace → reject if would revert or exceed gas budget. Save users from paying for failed txs.

### 2. Post-mortem debugging
A tx reverted in prod. Pull the trace, see exact opcode + storage state at revert. Faster than `cast run` or Foundry replay for one-off triage.

### 3. Virtual TestNets / forks
Spin up an ephemeral fork of mainnet (or any chain) at a specific block. Run integration tests against real contract state. Tear down after.

### 4. Alerts
Define rules ("emit alert when contract X emits event Y", or "when address Z's balance drops > 10%"). Route to Slack / PagerDuty / webhook.

## Webhook / callback handling

Tenderly Web3 Actions allow custom logic on alert triggers; alerts also fire to webhook endpoints you own.

- Verify webhook signature (Tenderly signs payloads).
- **Idempotency key:** Tenderly event ID.
- Buffer through SQS / Pub/Sub / Kafka before DB write.

<!-- TODO: paste signature verification snippet -->

## Common integration mistakes

- Treating simulation result as ground truth without re-running close to broadcast — state can change between simulate and submit (MEV).
- Forgetting to pass current `block_number` to simulation → simulates at stale state.
- Not tearing down virtual TestNets after CI runs → bills pile up.
- Routing alerts straight to PagerDuty without rate limits → alert storms.
- Hard-coding fork block numbers in CI — flaky as time passes; instead anchor to a tagged block.

<!-- TODO: extend -->

## MCP integration

Tenderly has begun offering MCP integration. Wire it into the agent for live simulation and debugging during reviews.

<!-- TODO: verify current Tenderly MCP server location; install and configure -->

## Latest docs reference

- Official docs: https://docs.tenderly.co/
- Simulation API: https://docs.tenderly.co/simulations
- Virtual TestNets: https://docs.tenderly.co/virtual-testnets
- Alerts / Web3 Actions: https://docs.tenderly.co/alerts
- Last-verified: `<YYYY-MM-DD>` <!-- TODO: bump on each re-check -->

## When NOT to use this vendor

- **Solana** — Tenderly is EVM-only; for Solana use Solana Explorer / Helius / Anchor traces.
- **Local unit tests** — use Foundry's `forge test` directly; Tenderly is for integration / production debugging.
- **Lightweight gas estimation** — `eth_estimateGas` against the RPC is cheaper for routine flows.

## Cross-references

- Parent category: [[category-smart-contract-testing]]
- Reviewed by: [[web3-backend-reviewer]] (§10 Debugging — Tenderly is the recommended debugger)
- Related: Foundry / Hardhat for local tests; Phalcon for adversarial replay
