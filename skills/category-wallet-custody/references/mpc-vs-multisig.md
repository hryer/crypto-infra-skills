# MPC vs Multisig — Decision Reference

## TL;DR

- **Multisig** = N on-chain signatures required. Visible, slow, gas-costly, auditable.
- **MPC/TSS** = N off-chain key shares produce 1 on-chain signature. Invisible, fast, gas-cheap, opaque.

They are not interchangeable. They solve different problems.

## Decision tree

```
Do you need on-chain auditability of the approval?
├── YES → Multisig (Safe / Gnosis)
└── NO  → Continue
        │
        Is signing latency critical (< 1s)?
        ├── YES → MPC
        └── NO  → Continue
                │
                Is on-chain footprint a cost issue?
                ├── YES (high tx volume) → MPC
                └── NO  → Either works; pick on operational preference
```

## Multisig (e.g., Safe)

### Properties
- Each approver has their own private key
- Approvals are on-chain transactions (or off-chain sigs aggregated then submitted)
- Threshold is enforced by the smart contract
- Smart contract address ≠ any individual signer address

### Strengths
- On-chain audit trail of who approved what
- Compatible with hardware wallets (Ledger, Trezor)
- Open ecosystem (Safe is widely supported by DeFi protocols)
- Threshold and signer set are governance-controllable on-chain

### Weaknesses
- Higher gas cost per transaction (multi-sig validation is more expensive than EOA)
- Slower (requires multiple parties to act)
- Not supported on non-EVM chains in the same form (Solana has multisig but it's different)
- Approval txns must wait for block confirmation before final execution

## MPC / TSS (e.g., Fireblocks, Fordefi)

### Properties
- The "private key" never exists in one place — it exists only as N shares
- A threshold (e.g., 2-of-3) of shares jointly produces a valid signature
- The signature is indistinguishable on-chain from a normal EOA signature
- Re-sharing changes the share set without changing the public key/address

### Strengths
- Single signature on-chain = lower gas, same cost as EOA
- Fast: signing happens in milliseconds once shares cooperate
- Works on any chain (the chain doesn't know it's MPC)
- Share rotation without changing addresses (huge ops win)

### Weaknesses
- Off-chain trust: you must trust the MPC implementation's cryptography
- No on-chain visibility of approvals (whoever holds the share controls signing)
- Vendor lock-in: shares from Vendor A don't work with Vendor B
- Recovery from lost shares depends entirely on vendor's procedure

## Common confusions

**"MPC is just multisig with extra steps"** — No. Multisig is N keys. MPC is 1 key split into N shares. The end result on-chain looks completely different.

**"MPC is more secure than multisig"** — Depends on threat model. Multisig protects against single-key compromise via *governance*. MPC protects against single-key compromise via *cryptography*. Both can fail.

**"We use MPC so we don't need a policy engine"** — Wrong. The shares can sign anything by default. The policy layer (Fireblocks' policy engine, Fordefi's rules) is what makes MPC safe in practice.

## When to combine

Some products use **both**:
- MPC for hot operational wallet (fast, cheap signing)
- Multisig for cold treasury and governance (transparent, slow approval)

This is the standard pattern for serious institutional setups.
