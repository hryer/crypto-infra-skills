---
name: multi-chain-tx-management
description: Use when designing transaction submission, monitoring, and recovery for a multi-chain backend (EVM + Solana + TRON + others). Covers nonce management, gas/fee estimation per chain, RBF/cancellation patterns, confirmation strategies, reorg handling, and unified transaction status modeling. Use whenever the user mentions transaction submission, nonce management, multi-chain wallet, gas estimation, transaction stuck, or RBF.
---

# Multi-Chain Transaction Management

## When to use

Trigger this skill when the user asks to:
- Design a transaction submission service for multiple chains
- Handle nonce gaps, stuck transactions, or replace-by-fee
- Unify transaction status tracking across EVM/Solana/TRON
- Estimate gas/fees reliably across chains
- Recover from reorgs in a tx submission pipeline
- Design retry logic that doesn't double-spend

## The cross-chain complexity tax

Each chain has its own:
- **Fee mechanism** — EVM has gas, Solana has lamports per compute unit, Bitcoin has sats per vByte, TRON has bandwidth + energy
- **Transaction lifecycle** — pending → confirmed (some chains have "preconfirmed" in between)
- **Finality model** — probabilistic (PoW, PoS chains) vs deterministic (Solana finalized commitment)
- **Replace/cancel semantics** — EVM RBF via same-nonce-higher-gas, Solana doesn't really have RBF, Bitcoin has BIP-125
- **Nonce / sequence model** — EVM sequential, Solana recent_blockhash, TRON uses ref_block

A "multi-chain tx service" that pretends these are all the same will fail in production. The job is to find what CAN be unified (status model, retries) and isolate what CAN'T (fee mechanics, signing).

## Process

### 1. Pick a unified status model

What every chain has in common:

```
States:
  PENDING_SIGN           # constructed but not signed
  PENDING_BROADCAST      # signed, not yet sent
  BROADCAST              # sent to network, no confirmation
  PRECONFIRMED           # seen by mempool / first confirmation
  CONFIRMED              # N confirmations (chain-specific N)
  FINALIZED              # safe from reorg
  FAILED                 # rejected, expired, or reverted
  REPLACED               # superseded by another tx (RBF)
  CANCELED               # explicitly canceled (rare)
```

The thresholds for CONFIRMED vs FINALIZED differ per chain. The state machine doesn't.

### 2. Nonce/sequence management per chain

**EVM (sequential nonces):**
- Account has a nonce N. Next tx must use N.
- Send tx with N+1 before N is mined → it sits in mempool waiting for N
- Tracking: maintain a per-account "next nonce" counter, sync with chain on startup
- Concurrency: lock or queue per account to avoid two services taking the same nonce
- Edge case: smart accounts (ERC-4337) use 2D nonces (key + sequence). Mostly hidden by bundler but exposed when you build your own.

**Solana (recent_blockhash):**
- No sequential nonces. Each tx references a recent_blockhash from the last ~150 slots.
- After ~2 minutes, the blockhash expires and the tx is dropped.
- Replacements: you can't replace by nonce. You send a new tx with a new blockhash.
- This means Solana "stuck transactions" expire on their own — much simpler than EVM.

**TRON:**
- Uses `ref_block_bytes` and `ref_block_hash` from recent blocks. Like Solana, txs expire.
- Default expiration is 60 seconds. Configurable up to days.
- No nonce conflict issue.

**Bitcoin:**
- UTXO-based. No nonce. Inputs ARE the "nonce" — once a UTXO is spent, you can't double-spend it.
- RBF via BIP-125: a tx can replace itself if it spends the same inputs at a higher fee.

**Strategic implication:** EVM nonce management is the hardest. Solana/TRON/Bitcoin are simpler once you stop trying to use EVM mental models.

### 3. Fee estimation per chain

Fee mechanisms differ fundamentally:

| Chain | Fee components | Estimation source |
|---|---|---|
| EVM (EIP-1559) | baseFee (network-set) + priorityFee (user-set) | eth_feeHistory + your own analysis |
| EVM (legacy) | gasPrice (user-set) | eth_gasPrice + buffer |
| Solana | priority fee (microlamports per CU) + base fee | getPrioritizationFees |
| TRON | bandwidth + energy (frozen TRX or burned) | Estimate from contract execution |
| Bitcoin | sats per vByte | mempool.space API or own mempool analysis |

**Buffer for safety:**
- EVM: +20% on priority fee for "next block" inclusion
- Solana: +50% on priority fee during congestion (it spikes hard)
- Bitcoin: +25% on fee rate for "next 3 blocks"

**Dynamic re-estimation:** for time-sensitive txs (liquidations, MEV), re-estimate just before signing. Cached estimates go stale fast.

### 4. RBF (replace-by-fee) and cancellation

**EVM:**
- Replace: same nonce, higher gas price (typically +10% min on both maxFeePerGas and maxPriorityFee)
- Cancel: same nonce, 0-value self-send with higher gas
- Caveats: if original is mined just before your replacement, you've sent a redundant tx; nonce now advances

**Solana:**
- Can't RBF. Just send a new tx with a fresh blockhash.
- "Cancel" doesn't exist as a concept; old tx will simply expire.

**TRON:**
- Can't directly RBF. Wait for expiration and retry.

**Bitcoin:**
- BIP-125 RBF: original tx must signal RBF in its sequence number; replacement spends same inputs at higher fee.

**Service-level pattern:** track tx by `intent_id` (your concept), not by hash. An intent can have multiple "attempts" (each with a different hash). Status is on the intent, not the attempts.

### 5. Confirmation strategy

How long do you wait before declaring a tx "confirmed enough"?

| Chain | Practical confirmation | True finality |
|---|---|---|
| Ethereum mainnet | 12 blocks (~2.5 min) | 64-128 blocks (~30 min) for finality |
| Polygon PoS | 32 blocks (~1 min) | ~256 blocks for checkpoint finality |
| BSC | 15 blocks (~45s) | ~30 blocks |
| Arbitrum/Optimism | Soft confirm via sequencer (sec) | L1 finalization (~7 days for fraud-proof) |
| Base | Same as Optimism | Same |
| Solana | `confirmed` commitment (~13s) | `finalized` (~32 slots, ~30s) |
| TRON | 20 blocks (~60s) | After SR voting (~3 min) |
| Bitcoin | 6 blocks (~60 min) | Effectively final at 6+ |

**Two-tier confirmation:**
- Optimistic: show the user "confirmed" at chain's normal threshold
- Pessimistic: hold reversible business actions (e.g., dispense fiat for off-ramp) until finalization

The gap between these matters most on L2s where soft-confirms are fast but rollup finalization is slow.

### 6. Reorg handling for outbound txs

A tx you sent gets reorged out:
- For EVM: track via tx receipt's blockHash; if blockHash for the same blockNumber changes, the tx was reorged
- For Solana: a confirmed tx can theoretically reorg before finalization (rare but possible)
- For Bitcoin: a tx at 1 conf can disappear; rare at 3+ confs

**Recovery:**
- Mark the tx state back to BROADCAST
- Wait for re-inclusion (the original tx is likely still in some node's mempool)
- If after 5-10 minutes it doesn't re-confirm, treat as failed and retry from intent

### 7. Idempotency at the intent level

Submit-tx APIs must be idempotent. Use `intent_id` as the idempotency key.

```go
type SubmitIntent struct {
    IntentID        string   // client-supplied, idempotency key
    Chain           string
    From            string
    To              string
    Amount          decimal.Decimal
    AssetID         string
    MaxFee          decimal.Decimal
    SubmittedBy     string
    SubmittedAt     time.Time
}

type Intent struct {
    SubmitIntent
    Status          string  // PENDING_SIGN | BROADCAST | CONFIRMED | ...
    Attempts        []Attempt
}

type Attempt struct {
    Hash            string
    BroadcastedAt   time.Time
    FeeEstimate     decimal.Decimal
    Status          string
    ReplacedBy      string  // hash of replacement attempt, if any
}
```

If the same `intent_id` is submitted twice: return the existing intent's state, do not start a new attempt.

### 8. Service architecture

```
                   ┌────────────────────┐
   API ──intent──> │  Intent Manager    │
                   └────────────────────┘
                            │
                            ▼
                   ┌────────────────────┐
                   │  Chain Submitter   │  (one per chain)
                   │    - signer        │
                   │    - nonce mgr     │
                   │    - fee estimator │
                   │    - broadcaster   │
                   └────────────────────┘
                            │
                            ▼
                       Chain RPC
                            │
                            ▼
                   ┌────────────────────┐
                   │  Status Watcher    │  (subscribes to chain events)
                   └────────────────────┘
                            │
                            ▼
                   Updates Intent state
```

**Key design decisions:**
- Intent Manager is chain-agnostic
- Chain Submitter is chain-specific (separate service or package per chain)
- Status Watcher is the eyes — separate from Submitter so they can scale independently
- All state flows through the Intent Manager (single writer)

## Rationalizations to reject

| Excuse | Reality |
|---|---|
| "We'll use the same nonce management for all chains" | EVM nonces are unique. Don't pretend Solana has nonces. |
| "We don't need RBF, we'll wait for txs to confirm" | A stuck tx with low gas can block a queue of 100 user actions for hours. RBF is required. |
| "Fee estimation is the RPC's job" | Public RPC fee estimators are conservative and often wrong. Build your own using historical data. |
| "We don't need finality, confirmation is enough" | For high-value off-ramps, you need finality. A reorg after dispensing fiat = loss. |
| "Multi-chain means we can use one library for everything" | Wrapper libraries that claim "ethers but for all chains" mostly lie. Use the native SDK per chain. |

## Verification

Before launch:
- [ ] Nonce conflict test: submit two txs concurrently from the same EVM account; both should land or one should be rejected cleanly
- [ ] Stuck-tx test: send EVM tx with deliberately low gas, verify RBF kicks in after timeout
- [ ] Solana blockhash expiry test: verify expired txs are detected and retried
- [ ] Reorg test (testnet): force a reorg, verify status reverts correctly
- [ ] Finality test: verify business actions are gated on true finality where needed
- [ ] Idempotency test: submit same intent_id twice; same result, only one attempt
- [ ] Monitor: open intents, stuck intents (> 5 min in BROADCAST), failed intents, finality lag per chain
