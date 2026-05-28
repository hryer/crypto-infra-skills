---
name: category-token-security-screening
description: Entry point for token security screening — detecting honeypots, rug-pull patterns, mint-authority risk, blacklist functions, and LP-lock status before showing or transacting a token in your app. Covers vendor-based screening (Birdeye, GoPlus, Honeypot.is, TokenSniffer) and DIY on-chain heuristics. Use whenever the user mentions token security, honeypot detection, rug pull, scam token, token vetting, or pre-trade safety check.
---

# Token Security Screening — Category

Entry point for vetting a token before listing, displaying, or trading it.

## When to use this skill

- Building a token-search / discovery feature where users can find any token
- Allowing user-initiated swaps to arbitrary tokens (vs a curated allowlist)
- Showing user balances of tokens that arrive unprompted (airdrop / scam dust)
- Adding pre-trade safety checks for a swap UI
- Reviewing whether a listed token should remain listed

## The use cases

| Use case | Typical answer |
|---|---|
| Show "is this token safe?" badge on swap UI | Vendor screening API ([[vendor-birdeye]] / GoPlus) |
| Pre-trade hard block on known honeypots | Vendor + on-chain heuristics (belt + suspenders) |
| Curate a "verified" tokens allowlist | Manual review of vendor signals + your own policy |
| Detect newly suspicious behavior on existing listings | Periodic re-scan + alert on change |
| Solana token discovery (memecoin era) | [[vendor-birdeye]] is the strongest signal source |

## Vendor options

| Vendor | Coverage | Best for | Trade-offs | Backend skill |
|---|---|---|---|---|
| **Birdeye** | Solana + many EVM chains | Default for multi-chain, esp. Solana | Mixed depth across chains | [[vendor-birdeye]] |
| GoPlus | Mostly EVM | Wide EVM coverage, free tier | API maturity varies | *(not in scaffold)* |
| Honeypot.is | EVM (Ethereum, BSC focus) | Simulate-based honeypot detection | Narrow scope | *(not in scaffold)* |
| TokenSniffer | EVM | Established source for token audits | Less real-time | *(not in scaffold)* |
| De.Fi | EVM + Solana | Scanner + audit aggregator | Mid-tier coverage | *(not in scaffold)* |

## Build-it-yourself option

Vendor signals are useful but not sufficient. Layer in on-chain heuristics you check yourself.

### EVM heuristics
- **Owner can mint?** Check the ERC-20 contract for unrestricted `mint()` callable by owner.
- **Blacklist / transfer disable?** Check for functions like `setBlacklist`, `pause`, `setTradingEnabled`.
- **Owner not renounced?** `owner() != address(0)` means the owner can change tokenomics.
- **Fee on transfer?** Honeypot pattern: 0% buy fee, 100% sell fee.
- **LP locked?** Check Unicrypt / Team.Finance / PinkSale lockers; or directly that the LP token is held by a known locker contract.
- **Proxy upgradeable?** Owner can swap implementation; treat with extra suspicion.

### Solana heuristics
- **Mint authority renounced?** `mintAuthority == null` for fixed supply.
- **Freeze authority renounced?** `freezeAuthority == null` so accounts can't be frozen.
- **LP locked?** Raydium / Orca LP — check known burn / lock addresses.
- **Update authority on metadata?** If retained, metadata (name, image) can change.

### Heuristic execution
- Run on token first-seen (initial scan).
- Re-run periodically for tokens in any active user position.
- Diff results — alert when a previously-safe token becomes unsafe (owner re-enabled mint, etc.).

## Backend best practices (inline)

### Caching
- Cache vendor screen results in Redis with TTL of hours (12-24h typical).
- Cache token metadata long (days).
- For tokens with no active user holdings, eager cache only on demand.

### Idempotency + freshness
- Persist the scan result + timestamp; never overwrite without keeping the prior result for diff.
- On user-facing display, show "last scanned X minutes ago" — transparency builds trust.

### Failure modes
- Vendor down: fall back to DIY heuristics + display "screening unavailable, proceed with caution" UI hint.
- DIY heuristic failure (RPC error): retry; if persistent, fall back to vendor-only.
- See [[web3-backend-reviewer]] §9 (fallback strategy).

### Language idioms
- **TypeScript:** prefer [viem](https://viem.sh) for the on-chain heuristic reads (typed ABI calls). For Solana, `@solana/web3.js`.
- **Golang:** `go-ethereum` bound contracts for EVM heuristics; community Solana libs for SPL Token program reads. Follow [Effective Go](https://go.dev/doc/effective_go).

### Infra patterns
- **Postgres:** persist `(chain_id, token_address, scan_timestamp, vendor_signals, diy_heuristics, verdict)`.
- **Redis:** verdict cache.
- **AWS SQS / GCP Pub/Sub / Kafka:** queue re-scan jobs; don't block user requests on a fresh scan.

## Decision tree

1. **You curate listings manually** → use vendors for triage, your team makes final call.
2. **Open swap UI, any token allowed** → vendor + DIY heuristics on every first-time interaction.
3. **Solana memecoin app** → [[vendor-birdeye]] required; layer DIY mint/freeze authority checks.
4. **EVM L1 / L2 focused** → GoPlus + DIY heuristics + Honeypot.is for known honeypots.
5. **Showing arbitrary incoming token balances** → tag suspected scams visually; don't block by default (false positives erode trust).

## Cross-references

- Vendor: [[vendor-birdeye]]
- Related categories: [[category-swap-integration]] (pre-trade check), [[category-rpc-and-indexer]] (DIY heuristics need RPC), [[category-smart-contract-testing]] (simulate honeypot suspicion via [[vendor-tenderly]])
- Reviewed by: [[web3-backend-reviewer]] (§7 vendor integrations)
- Security-reviewed by: [[wallet-security-auditor]] (user-protection logic)
