---
name: vendor-lifi
description: Backend integration guide for Li.Fi — cross-chain swap and bridge aggregator across EVM chains, Solana, and beyond. Covers quote API, status API, route execution, fee handling, webhook / status polling, supported chains and tokens. Use whenever the user is integrating Li.Fi for in-app swap, cross-chain bridge, or any aggregated swap routing at the backend level.
---

# Li.Fi

Cross-chain swap + bridge aggregator. Primary vendor under [[category-swap-integration]].

## What this vendor is for

Li.Fi aggregates DEXes (Uniswap, PancakeSwap, Curve, etc.) and bridges (Stargate, Across, Hop, etc.) across many chains. You pass an input (chain + token + amount + recipient), Li.Fi returns the best route as a serialized transaction, you submit it from the user's wallet (or your relayer). Status API tells you when the cross-chain leg lands.

## Custody / data / pricing model

- **Custody model:** Non-custodial. Li.Fi returns transaction data; user / your relayer signs and submits.
- **Pricing:** Li.Fi takes an integrator fee (configurable, set by you); plus the underlying DEX / bridge fees. Free to call the API.

## Auth & API setup

- API key (optional but recommended for higher rate limits and analytics).
- Source from KMS / Vault if used.
- Set the `integrator` field on every quote request to attribute traffic and (optionally) collect fees.

<!-- TODO: fill in concrete auth/quote example -->

## SDK usage

### TypeScript

Official SDK: `@lifi/sdk` (or `@lifi/widget` for the embedded UI).

<!-- TODO: paste minimal getRoutes + executeRoute example -->

Strict mode on. Quote responses include serialized tx data — type the response carefully; do NOT submit untyped data to viem/ethers.

### Golang

<!-- TODO: confirm Go SDK; if REST-only, build a thin client. -->

Follow [Effective Go](https://go.dev/doc/effective_go) — context-first, error wrapping. Important: never log full quote responses (they include user addresses).

## Webhook / callback handling

Li.Fi is poll-based for status (no first-party webhooks for cross-chain status, as of last check). Submit the source tx, then poll the Status API with `txHash`, `bridge`, `fromChain`, `toChain` until status is `DONE` / `FAILED`.

- **Idempotency key:** use the source tx hash.
- **Backoff:** exponential, capped — cross-chain settlement can take seconds (Across) to hours (Stargate).
- **Infra wiring:** persist `(swap_id, source_tx_hash, status)` in DB; background worker polls status; emit your own internal event when terminal.

<!-- TODO: verify whether Li.Fi now offers push webhooks; update if so -->

## Common integration mistakes

- Submitting the same quote tx multiple times after a Quote refresh — refresh creates new tx data; track the chosen quote ID.
- Ignoring slippage settings — default may be too loose for institutional flows; lock down explicitly.
- Not handling partial fills on bridges that support them (e.g., Across).
- Logging full quote responses (PII / wallet address exposure).
- Assuming "DONE" means funds are usable — for some bridges DONE means in-flight; check the destination chain.

<!-- TODO: extend with real-world gotchas -->

## MCP integration

<!-- TODO: check Li.Fi MCP availability. -->

## Latest docs reference

- Official docs: https://docs.li.fi/
- API reference: https://apidocs.li.fi/
- SDK: https://github.com/lifinance/sdk
- Last-verified: `<YYYY-MM-DD>` <!-- TODO: bump on each re-check -->

## When NOT to use this vendor

- **Single-chain DEX swap with no bridge** → call the DEX router directly (Uniswap, PancakeSwap); save the fee.
- **Solana-only product** → use Jupiter (Solana-native aggregator) for better quotes.
- **You need on-chain only execution with no off-chain components** → not applicable; Li.Fi requires API calls.

## Cross-references

- Parent category: [[category-swap-integration]]
- Reviewed by: [[web3-backend-reviewer]] (§7 vendor integrations, §9 status polling)
- Alternatives: 1inch (EVM aggregator), 0x (EVM aggregator), Jupiter (Solana), Paraswap (EVM)
