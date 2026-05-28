---
name: category-wallet-custody
description: Entry point for any wallet/custody decision in a crypto product. Covers the custody model choice (EOA, multisig, MPC/TSS, HSM, ERC-4337, EIP-7702), the WaaS vendor landscape across two tiers (consumer-facing Tier 1 — Privy, Dynamic, Web3Auth, Magic, Para — and institutional Tier 2 — Fireblocks, Fordefi, Turnkey, BitGo, Anchorage), build-vs-buy decisions, multi-chain key management (EVM/Solana/TRON/Bitcoin), and operational runbooks. Branch to vendor-* skills for backend integration details. Use whenever the user mentions wallet architecture, custody, MPC, multisig, embedded wallet, WaaS, Privy, Dynamic, Fireblocks, Fordefi, Turnkey, ERC-4337, EIP-7702, account abstraction, paymaster, or key management.
---

# Wallet Custody — Category

This skill is the **decision entry point** for any wallet/custody work. Once a custody model and (optionally) a vendor are chosen, jump to the matching `vendor-*` skill for the backend implementation details.

## When to use this skill

- Picking a custody model (EOA vs multisig vs MPC vs HSM vs smart account)
- Comparing WaaS vendors (Privy, Dynamic, Fireblocks, Fordefi, Turnkey, BitGo, etc.)
- Deciding whether to layer consumer wallets (Tier 1) and institutional signing (Tier 2)
- Planning a migration from one WaaS to another, or from in-house to vendor (or vice versa)
- Designing an ERC-4337 / EIP-7702 smart-account stack
- Architecting multi-chain key management (EVM + Solana + TRON + Bitcoin)
- Planning key rotation, cold-storage ceremony, or recovery procedures

## The use cases (sub-problems within wallet custody)

| Use case | Typical answer |
|---|---|
| End-user wallet for a consumer app | Tier 1 embedded wallet (Privy / Dynamic) |
| Treasury or hot signer for a platform | Tier 2 institutional signer (Fireblocks / Fordefi / Turnkey) |
| User signs hundreds of txs/day with great UX | Smart account (ERC-4337) + paymaster |
| Existing EOAs need smart-account features | EIP-7702 delegation |
| Multi-party approval for governance | Multisig (Safe) |
| Single hot wallet in a regulated environment | HSM + policy engine |
| Power user managing own funds | EOA with seed phrase (sovereignty) |

## The fundamental split: two tiers, two markets

WaaS vendors are NOT a single market. Confusing the two tiers is the #1 mistake.

### Tier 1 — Consumer wallet infrastructure
- Embedded wallets for end-users
- Email / social / passkey login
- Mobile + web SDKs
- Usually MPC-backed for "non-custodial-ish" UX
- **Vendors:** [[vendor-privy]], [[vendor-dynamic]], Web3Auth, Magic, Para (formerly Capsule)

### Tier 2 — Institutional / treasury signing
- Custody platform for platform funds, hot wallets, settlement
- Policy engines, multi-approver workflows
- MPC or multisig + HSM
- Compliance, audit, regulated custody options
- **Vendors:** [[vendor-fireblocks]], [[vendor-fordefi]], [[vendor-turnkey]], BitGo, Anchorage, Cobo

Most products need BOTH tiers. Some only need one (consumer DEX = Tier 1 only; DeFi protocol team = Tier 2 only).

## Vendor options

### Tier 1 (consumer wallets)

| Vendor | Strengths | Weaknesses | Best for | Backend skill |
|---|---|---|---|---|
| **Privy** | Best DX, mature embedded wallet, broad auth, MPC-backed, smart-account add-on | Pricing scales fast at high MAU | Default for most consumer apps | [[vendor-privy]] |
| **Dynamic** | Multi-chain native (EVM + Solana strong), good for crypto-native users | Less polished email/social UX than Privy | Multi-chain apps, DeFi front-ends | [[vendor-dynamic]] |
| **Web3Auth (tKey)** | Open-core, more control over MPC, customizable | More integration work | Teams owning more of the stack | *(not in scaffold)* |
| **Magic** | Email-link auth, older incumbent | Less crypto-native, lagging on new features | Web2-leaning crypto apps | *(not in scaffold)* |
| **Para (Capsule)** | MPC + passkey, good for cross-app wallets | Smaller ecosystem | Specific niches | *(not in scaffold)* |
| **Turnkey** (consumer mode) | API-first, HSM-backed, very flexible | More integration work | Teams building custom UX | [[vendor-turnkey]] |

### Tier 2 (institutional)

| Vendor | Strengths | Weaknesses | Best for | Backend skill |
|---|---|---|---|---|
| **Fireblocks** | Mature, broad chain support, comprehensive policy engine, MCP server available | Expensive, slow to support new chains/L2s | Compliance-focused, enterprise | [[vendor-fireblocks]] |
| **Fordefi** | Modern UX, faster on new chains, better DeFi UX | Smaller team, less brand recognition | DeFi-native teams | [[vendor-fordefi]] |
| **Turnkey** | API-first, cheap at scale, HSM-backed | Less DeFi tooling, more DIY | High-volume programmatic signing | [[vendor-turnkey]] |
| **BitGo** | Multisig heritage, strong custodial brand | UX is dated | Regulated/conservative institutions | *(not in scaffold)* |
| **Anchorage** | US-regulated crypto bank, qualified custodian | Slower onboarding, US-focused | US institutions needing qualified custody | *(not in scaffold)* |
| **Cobo** | Asia presence, mixed custody/MPC | Less western traction | APAC operations | *(not in scaffold)* |

## Build-it-yourself option

When you're not using a vendor, you're choosing one of these custody primitives directly. Each has a place — none is universally right.

| Primitive | Use when | Avoid when |
|---|---|---|
| **EOA + seed phrase** | Personal wallets, sovereignty matters, user is technical | Custodial product, recovery needed |
| **Multisig (Safe / Gnosis)** | Treasury, DAO governance, 2-of-7 signers, on-chain transparency | Sub-second signing, mobile UX |
| **MPC / TSS (in-house)** | High scale where vendor pricing dominates, you have crypto-engineering depth | You can't afford one full-time cryptographer on call |
| **HSM (CloudHSM / on-prem)** | Single hot wallet, regulated environment, audit trail required | Multi-party approval needed |
| **Smart account (ERC-4337)** | UX features (gasless, batching, session keys, social recovery) on EVM | Solana/non-EVM, simple custody only |
| **EIP-7702** | Existing EOAs need smart-account features without migration | Pre-Pectra chains, full smart-account-only flow |

### DIY: ERC-4337 stack

Required components:
- **UserOperation** — meta-tx signed by smart-account owner
- **Bundler** — Pimlico, Stackup, Alchemy bundler, or self-hosted
- **EntryPoint** — singleton contract (0x...0789 v0.7)
- **Account contract** — Kernel (best plugins), Safe (best audit), SimpleAccount (reference)
- **Paymaster (optional)** — verifying (off-chain sig), token (ERC-20 pay), or sponsored (free)

Decisions to lock down:
- Self-host bundler or buy? **Buy** until volume > 100k UserOps/month
- Account implementation? Kernel for plugin flexibility, Safe for audit story
- Paymaster type — pick based on abuse model and UX target

### DIY: EIP-7702 migration path (post-Pectra)

- Users sign an authorization granting their EOA the code of a smart-account implementation
- Tx looks like a normal EOA tx but executes smart-account logic
- **Critical pitfalls:** authorization is per-chain (re-authorize per chain); delegated code can be replaced (security depends on what implementation you point to); combining 7702 + 4337 has subtle validation rules

### DIY: multi-chain key management

| Chain family | Curve | Same key as EVM? | Notes |
|---|---|---|---|
| EVM (all chains) | secp256k1 | yes | One key works across all EVM chains |
| Solana | ed25519 | **no** | Different curve, different key entirely |
| TRON | secp256k1 | yes | Same key as EVM, different address format |
| Bitcoin | secp256k1 | yes (key), no (signing flow) | UTXO model is fundamentally different |

Implication: a multi-chain wallet backend needs at minimum two key-derivation paths (secp256k1 and ed25519). Many MPC vendors only support secp256k1 — verify before committing.

## Backend best practices (inline)

These cut across every wallet integration. Detailed coverage lives in [[web3-backend-reviewer]] (agent) and the relevant `vendor-*` skill.

### Key storage
- **Never** persist `private_key`, `mnemonic`, `seed_phrase`, `keystore_json`, `.perm` in plaintext — DB, logs, env files, or images. See [[web3-backend-reviewer]] §6 and §8.
- Secrets sourced from KMS (AWS / GCP) / Vault / vendor-managed key IDs — not raw env vars.
- Testnet and mainnet keys provably isolated (separate KMS namespace, separate IAM).

### SDK conventions
- **TypeScript clients** — prefer [viem](https://viem.sh) for new code; treat `web3.js` and `ethers.js` as legacy. Strict mode, typed ABIs, public/wallet client split.
- **Golang backends** — conform to [Effective Go](https://go.dev/doc/effective_go). Context propagation through every signing path. Goroutine hygiene (no leaks). Error wrapping with `fmt.Errorf("...: %w", err)`.

### Webhook / callback handling
- Verify signatures (HMAC or vendor-specific).
- Idempotency key on every inbound event.
- Buffer through AWS SQS / GCP Pub/Sub / Kafka so a downstream DB outage doesn't drop deliveries.
- Reconciliation job (REST poll backstop) for when webhooks fail silently.

### Hidden vendor costs to pin down
Vendor pricing is usually "talk to sales." Ask:

**Tier 1:** per-MAU (and how is MAU defined?), per-signed-tx, per-chain enablement, white-labeling, sub-account/multi-tenant, contractual price-increase caps.

**Tier 2:** per-workspace fee ($1k-$10k/mo typical), per-tx fee on top, per-asset enablement fee, setup/onboarding, API call costs, custom policy engine fees.

Common gotcha: "$X per MAU" where MAU = anyone who logged in once (not anyone who signed). Can 2-5x the bill.

### Lock-in mitigation
- **Tier 1:** wallet address is the user identifier in your DB, not vendor-specific IDs.
- **Tier 2:** treat the vendor as a signer adapter — no vendor-specific concepts in your domain model.

## Decision tree (use case → recommendation)

1. **Consumer DEX / wallet app**
   - Tier 1: [[vendor-privy]] or [[vendor-dynamic]]
   - Tier 2: not needed, or minimal HSM for operational gas wallet

2. **Crypto card product**
   - Tier 1: [[vendor-privy]] (user wallet) — or custodial via Tier 2
   - Tier 2: [[vendor-fireblocks]] or [[vendor-fordefi]] (treasury, settlement, fee payment)
   - Card-issuer integration: see [[category-crypto-card]]

3. **DeFi protocol team**
   - Tier 1: not needed (users connect their own wallet)
   - Tier 2: Safe multisig (governance) + [[vendor-fireblocks]] / [[vendor-fordefi]] (treasury) + HSM for relayer

4. **CEX / brokerage**
   - Tier 1: own custodial wallet system
   - Tier 2: BitGo or Anchorage (regulated) + [[vendor-fireblocks]] (operational)

5. **B2B crypto SaaS**
   - Tier 1: [[vendor-turnkey]] or [[vendor-dynamic]] (programmatic per-customer wallets)
   - Tier 2: [[vendor-fireblocks]] (own treasury)

6. **High-UX consumer app needing gasless / batched txs**
   - Layer ERC-4337 on top of Tier 1 (Privy's smart-account add-on, or roll your own with Pimlico bundler)

## Rationalizations to reject

| Excuse | Reality |
|---|---|
| "MPC is just better multisig" | No. Multisig is on-chain and auditable. MPC is off-chain and faster but invisible to chain observers. Different problems. |
| "We'll just use Privy for everything" | Privy is Tier 1. Treasury policy + approver workflows are not its job. |
| "Fireblocks for users is just Privy with extra features" | Fireblocks per-user pricing is 10-100x Privy's. You'll burn money. |
| "ERC-4337 is the future, let's go 4337-only" | Solana exists. Non-EVM chains exist. 4337 isn't universal. |
| "EIP-7702 makes ERC-4337 obsolete" | Complementary. 7702 upgrades EOAs, 4337 enables fresh smart accounts. |
| "We can build our own MPC, it's just math" | MPC implementations have subtle bugs that lose funds. Don't roll your own without a real cryptographer. |
| "Our HSM is secure, we don't need policies" | An HSM with no policy engine signs anything. The policy layer is what makes it safe. |
| "Vendor lock-in isn't a real concern" | Migrations are 6-12 month projects. Design with switchability in mind. |

## Operational runbooks (required before launch)

- **Key rotation procedure** — how do you rotate when an employee with HSM access leaves?
- **Cold storage ceremony** — air-gapped signing, witness sign-off, video recording
- **Incident response** — pause first, investigate second
- **Recovery testing** — quarterly restore-from-backup drills, not just on paper
- **Migration plan** — how would you switch vendor in 12 months?

## Verification

- [ ] Custody model documented with explicit threat model (who can do what, under what conditions)
- [ ] Key derivation paths defined for every supported chain
- [ ] Recovery procedure tested end-to-end on testnet
- [ ] Key rotation runbook exists and has been dry-run
- [ ] If using MPC: threshold and re-share procedure documented
- [ ] If using ERC-4337: bundler failover plan exists
- [ ] If using a paymaster: abuse policy in place (rate limits, allowlist, max gas per user)
- [ ] Vendor lock-in cost measured
- [ ] Pricing modeled at 6, 12, 24-month projected scale
- [ ] All chains supported confirmed (not "on roadmap")
- [ ] Pilot integration completed before signing

## Migration paths

**Tier 1 migration (e.g., Magic → Privy):**
1. Run both in parallel for a transition period
2. New users go to new vendor
3. Old users prompted to re-secure (generates new MPC share on new vendor)
4. Old vendor data exported and retained (rollback contingency)
5. After 90% migration, drop old vendor

**Tier 2 migration (e.g., Fireblocks → Fordefi):**
1. Onboard new vendor, configure policies
2. Generate new addresses
3. Move funds incrementally (signed by old vendor)
4. Update systems to point to new vendor
5. Decommission old vendor

Both take 3-12 months. Plan accordingly.

## Cross-references

- **Backend integration per vendor:** [[vendor-fireblocks]], [[vendor-fordefi]], [[vendor-privy]], [[vendor-dynamic]], [[vendor-turnkey]]
- **Review agent:** [[web3-backend-reviewer]] — reviews wallet backends for the dimensions above (custody classification §7, key management §6, schema §8, error handling §9)
- **Security agent:** [[wallet-security-auditor]] — cryptographic-level review (HSM, MPC ceremony, key generation entropy, recovery flows)
- **Crypto card product:** [[category-crypto-card]] — when wallet sits behind a card
- **Multi-chain tx submission:** [[category-multichain-tx-management]]

## References

- `references/mpc-vs-multisig.md` — when to use which, with concrete decision tree
- `references/erc-4337-stack.md` — full component breakdown with bundler/paymaster choices
- `references/eip-7702-migration.md` — migrating EOAs to smart accounts without breaking users
