---
name: wallet-infrastructure
description: Use when designing, reviewing, or migrating wallet infrastructure for crypto products. Covers custody models (EOA, multisig, MPC/TSS, HSM), smart accounts (ERC-4337 stack, EIP-7702), WaaS vendor selection (Privy/Dynamic vs Fireblocks/Fordefi), gasless transactions (paymasters), and multi-chain key management (EVM/Solana/TRON). Use whenever the user mentions wallet architecture, account abstraction, MPC, multisig, key custody, signing infrastructure, or WaaS integration.
---

# Wallet Infrastructure Design

## When to use

Trigger this skill when the user asks to:
- Choose a custody model (EOA vs multisig vs MPC vs HSM)
- Design or review an ERC-4337 / EIP-7702 smart account stack
- Compare WaaS vendors (Privy, Dynamic, Fireblocks, Fordefi, Turnkey)
- Design a paymaster / gasless transaction system
- Architect multi-chain key management (EVM + Solana + TRON)
- Migrate an existing wallet system to account abstraction
- Plan a cold storage ceremony or key rotation

## Process

### 1. Classify the custody requirement

Ask before designing. The right architecture depends on:

- **Who controls the key?** End-user (non-custodial), platform (custodial), or shared (semi-custodial)?
- **What's the loss tolerance?** A retail wallet losing $50 is different from a treasury losing $50M.
- **What's the operational tempo?** Hot signing every second vs. cold signing once a week.
- **What's the recovery story?** Seed phrase, social recovery, MPC re-share, multisig threshold change?

### 2. Pick the custody primitive

| Primitive | Use when | Avoid when |
|---|---|---|
| **EOA + seed phrase** | Personal wallets, sovereignty matters, user is technical | Custodial product, recovery story needed |
| **Multisig (Safe / Gnosis)** | Treasury, DAO governance, 2-7 signers, on-chain transparency | Sub-second signing, mobile UX |
| **MPC / TSS** | Custodial product at scale, sub-second signing, no on-chain footprint | Need on-chain visibility of approvals |
| **HSM (cloud KMS / on-prem)** | Single hot wallet, regulated environment, audit trail required | Multi-party approval needed |
| **Smart account (ERC-4337)** | UX features (gasless, batching, session keys, social recovery) on EVM | Solana/non-EVM, simple custody only |
| **EIP-7702** | Existing EOAs need smart-account features without migration | Pre-Pectra chains, full smart-account-only flow |

### 3. Pick the WaaS vendor layer (if not building in-house)

Vendor selection has two tiers — don't conflate them:

**Tier 1: End-user wallet / auth (consumer-facing)**
- **Privy** — best DX, embedded wallets, email/social login, MPC-backed
- **Dynamic** — similar territory, stronger multi-chain story
- **Web3Auth (tKey)** — open-core, more control, more ops burden

**Tier 2: Institutional signing / treasury (back-office)**
- **Fireblocks** — incumbent, MPC + policy engine, expensive, slow to adopt new chains
- **Fordefi** — newer, MPC-based, better DeFi UX, supports more L2s out of the box
- **Turnkey** — API-first, HSM-backed, cheaper at scale, less DeFi-native
- **BitGo** — multisig heritage, strong institutional brand

**Don't pick one of each blindly.** Many products only need Tier 1 (consumer wallets) OR Tier 2 (treasury signer). Layering both is justified only when:
- Tier 1 holds user funds, Tier 2 holds platform float
- Or: Tier 1 = transient user signing, Tier 2 = settlement / sweep treasury

### 4. Design the ERC-4337 stack (if going smart-account)

Required components:
- **UserOperation** — the meta-tx, signed by the smart account owner
- **Bundler** — collects UserOps, submits them as a bundle (Pimlico, Stackup, Alchemy bundler, self-hosted)
- **EntryPoint** — singleton contract (0x...0789 v0.7) that processes the bundle
- **Account contract** — the actual smart wallet (Kernel, Safe, SimpleAccount, Biconomy)
- **Paymaster (optional)** — sponsors gas, runs validation logic

**Decisions to lock down:**
- Self-host bundler or buy? (Buy until volume > 100k UserOps/month)
- Which account implementation? (Kernel = best plugin system, Safe = best audit story, SimpleAccount = reference only)
- Paymaster type — verifying (off-chain sig), token (user pays in ERC-20), or sponsored (free)?

### 5. EIP-7702 migration path (post-Pectra)

If you already have EOAs and want smart-account features:
- Users sign an authorization granting their EOA the code of a smart-account implementation
- Tx looks like a normal EOA tx but executes smart-account logic
- **Critical pitfall:** the authorization is per-chain — re-authorize on every chain
- **Critical pitfall:** delegated code can be replaced — your security depends on what implementation you point to
- **Critical pitfall:** combining 7702 + 4337 is possible but the validation rules are subtle

### 6. Multi-chain key management

**EVM family (same curve secp256k1):** one private key works across all EVM chains. Multi-chain ≠ multi-key.

**Solana (ed25519):** different curve, different key entirely. Cannot derive from EVM key without an MPC scheme that supports both curves (most don't).

**TRON (secp256k1 but TRON-specific address derivation):** same key as EVM, different address format.

**Bitcoin (secp256k1, but UTXO model):** same key possible, but signing flow is fundamentally different.

Implication: a "multi-chain wallet" backend needs at minimum two key-derivation paths (EVM/secp256k1 and Solana/ed25519). Many MPC vendors only support secp256k1 — verify before committing.

### 7. Define the operational runbooks

A wallet system is incomplete without:
- **Key rotation procedure** — how do you rotate when an employee with HSM access leaves?
- **Cold storage ceremony** — air-gapped signing, witness sign-off, video recording
- **Incident response** — what if a key is suspected compromised? Pause first, investigate second.
- **Recovery testing** — quarterly restore-from-backup drills, not just on paper

## Rationalizations to reject

| Excuse | Reality |
|---|---|
| "MPC is just better multisig" | No. Multisig is on-chain and auditable. MPC is off-chain and faster but invisible to chain observers. They solve different problems. |
| "We'll just use Privy for everything" | Privy is consumer wallet infra. Putting platform treasury in Privy is a category error. |
| "ERC-4337 is the future, let's go 4337-only" | Solana exists. Non-EVM chains exist. 4337 isn't a universal answer. |
| "We don't need a paymaster, users have ETH" | If you're targeting non-crypto-native users, they don't. Gasless is a UX requirement, not a nice-to-have. |
| "EIP-7702 makes ERC-4337 obsolete" | They're complementary. 7702 upgrades EOAs, 4337 enables fresh smart accounts. Most production stacks will use both. |
| "Our HSM is secure, we don't need policies" | An HSM with no policy engine is just a key that signs anything. The policy layer is what makes it safe. |

## Verification

Before considering a wallet design complete:
- [ ] Custody model documented with explicit threat model (who can do what, under what conditions)
- [ ] Key derivation paths defined for every supported chain
- [ ] Recovery procedure tested end-to-end on a testnet
- [ ] Key rotation runbook exists and has been dry-run
- [ ] If using MPC: the threshold and re-share procedure are documented
- [ ] If using ERC-4337: the bundler failover plan exists (what if Pimlico goes down?)
- [ ] If using a paymaster: the abuse policy is in place (rate limits, allowlist, max gas per user)
- [ ] Vendor lock-in cost has been measured (how hard is it to leave this WaaS?)

## References

- `references/mpc-vs-multisig.md` — when to use which, with concrete decision tree
- `references/erc-4337-stack.md` — full component breakdown with bundler/paymaster choices
- `references/eip-7702-migration.md` — migrating EOAs to smart accounts without breaking users
