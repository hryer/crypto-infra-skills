# ERC-4337 Stack — Component Reference

## The five components

```
User signs ──> UserOperation ──> Bundler ──> EntryPoint ──> Account contract
                                                    │
                                                    └──> Paymaster (optional)
```

### 1. UserOperation
The meta-transaction. Signed by the smart account owner. Contains:
- `sender` — the smart account address
- `nonce` — replay protection (managed by EntryPoint, not the chain)
- `callData` — what the account should execute
- `callGasLimit`, `verificationGasLimit`, `preVerificationGas` — gas params
- `maxFeePerGas`, `maxPriorityFeePerGas` — EIP-1559 style
- `paymasterAndData` — if using a paymaster
- `signature` — owner's signature

### 2. Bundler
Off-chain service that:
- Receives UserOps from clients (typically via JSON-RPC `eth_sendUserOperation`)
- Validates them (simulates `validateUserOp` on the account)
- Bundles N UserOps into one EVM transaction
- Submits to the EntryPoint contract

**Bundler options:**
| Bundler | Hosted | Self-host | Notes |
|---|---|---|---|
| Pimlico | ✓ | ✓ | Most mature, good docs, supports many chains |
| Stackup | ✓ | ✓ | Open source, "infinitas" stack |
| Alchemy | ✓ | ✗ | Integrated with their RPC |
| Biconomy | ✓ | ✗ | Bundled with their account SDK |
| Self-hosted | — | ✓ | Use Stackup or Pimlico's reference impl |

**When to self-host:** when bundler costs exceed engineering cost, typically > 100k UserOps/month. Before that, just use Pimlico.

### 3. EntryPoint
The singleton contract at `0x0000000071727De22E5E9d8BAf0edAc6f37da032` (v0.7).
- Validates the UserOp (calls `validateUserOp` on the account)
- Validates the paymaster (calls `validatePaymasterUserOp`)
- Executes the call
- Handles refunds and prefund accounting

**You don't deploy this. It's already there. Just point at the address.**

### 4. Account contract
The actual smart wallet. Holds funds, implements `validateUserOp`.

| Implementation | Best for | Tradeoff |
|---|---|---|
| **Kernel (ZeroDev)** | Production. Plugin system (validators, hooks, executors). Modular. | Most complex |
| **Safe + 4337 module** | Existing Safe users who want gasless | Heavier gas cost |
| **SimpleAccount** | Reference / learning only | Not production-ready |
| **Biconomy SCW** | Tied to Biconomy SDK | Vendor lock-in |
| **Coinbase Smart Wallet** | Coinbase ecosystem | Passkey-only |

**Default pick: Kernel.** Plugin system means you can add features (session keys, social recovery, spending limits) without redeploying the wallet.

### 5. Paymaster (optional)
Sponsors gas for users. Three modes:

**Verifying paymaster** (most common)
- Off-chain service signs an attestation that "this UserOp can use my gas"
- Account-abstraction backend decides who to sponsor based on business rules
- Use for: app-funded gas (you eat the cost to onboard users)

**Token paymaster**
- User pays gas in an ERC-20 (USDC, DAI)
- Paymaster swaps to ETH to pay actual gas
- Use for: users who don't want to hold native gas tokens

**Sponsored paymaster (free for everyone)**
- Anyone can use it, no validation
- Use for: testnets only. On mainnet this is an instant drain attack.

## Cost model

Per UserOp on mainnet (rough, post-Dencun):
- Base EOA tx: ~21k gas
- 4337 UserOp through Kernel: ~80-150k gas (depends on validator)
- 4337 UserOp through Safe: ~150-250k gas
- Bundler fee on top: typically 10-30% markup on gas

**Implication:** 4337 is 4-10x more expensive than EOA per tx. You're buying UX (gasless, batching, session keys) with that overhead. Make sure the UX win justifies it.

## Operational pitfalls

1. **Bundler failover** — what happens if your bundler goes down? Have a fallback (e.g., Pimlico primary, self-hosted secondary).
2. **Paymaster abuse** — verifying paymasters can be drained by attackers spamming UserOps. Rate-limit by sender, by IP, by user account.
3. **Validation simulation drift** — `validateUserOp` is simulated off-chain by the bundler. If on-chain state changes between simulation and execution, the UserOp can fail. Bundlers handle this but it costs gas to discover.
4. **Nonce gaps** — 4337 nonces are 2D (key + sequence). Sequential nonces per key. If a UserOp at nonce 5 fails, nonce 6 won't execute. Design for this in your UX.
5. **Storage access rules (ERC-7562)** — the validation phase can only access certain storage slots. Custom validators that touch arbitrary storage will be rejected by bundlers. Read the storage rules before writing a custom validator.
