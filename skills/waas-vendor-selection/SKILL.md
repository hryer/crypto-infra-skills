---
name: waas-vendor-selection
description: Use when selecting Wallet-as-a-Service (WaaS) vendors for a crypto product. Covers the two-tier model (consumer wallet vs institutional signer), specific vendor trade-offs (Privy, Dynamic, Web3Auth, Fireblocks, Fordefi, Turnkey, BitGo), build-vs-buy decision, and migration paths. Use whenever the user mentions WaaS, wallet vendor, Privy, Dynamic, Fireblocks, Fordefi, Turnkey, or embedded wallet.
---

# WaaS Vendor Selection

## When to use

Trigger this skill when the user asks to:
- Pick between Privy, Dynamic, Web3Auth for embedded user wallets
- Pick between Fireblocks, Fordefi, Turnkey for institutional signing
- Decide whether to layer consumer + institutional WaaS
- Plan migration from one WaaS to another
- Evaluate build-in-house vs buy

## The fundamental split: two tiers, two markets

WaaS vendors are NOT a single market. They're two adjacent markets that some treat as one:

### Tier 1: Consumer wallet infrastructure
- Embedded wallets for end-users
- Email/social/passkey login
- Mobile + web SDKs
- Often MPC-based to keep keys "non-custodial-ish"
- **Vendors:** Privy, Dynamic, Web3Auth, Magic, Para (formerly Capsule)

### Tier 2: Institutional / treasury signing
- Custody platform for company funds, hot wallets, settlement
- Policy engines, multi-approver workflows
- MPC or multisig + HSM
- Compliance and audit features
- **Vendors:** Fireblocks, Fordefi, Turnkey, BitGo, Anchorage

**Confusing these is the #1 mistake.** A team picking "Privy for everything" finds out they can't satisfy treasury policy requirements. A team picking "Fireblocks for everything" finds out per-user costs make consumer wallets uneconomical.

## Process

### 1. Map your wallet needs

For each wallet role in your product, classify it:

| Wallet role | Tier | Examples |
|---|---|---|
| User's holding wallet | Tier 1 | Where users keep their balance |
| User's transient signing wallet | Tier 1 | For session signing |
| Platform float wallet | Tier 2 | For paying network fees, gas |
| Treasury / cold storage | Tier 2 | Long-term holdings |
| Settlement / sweep wallet | Tier 2 | Where user funds aggregate |
| Operations wallet | Tier 2 | For paying contractors, vendors |

Most products need BOTH tiers. Some only need one.

### 2. Tier 1 selection (consumer wallets)

| Vendor | Strengths | Weaknesses | Best for |
|---|---|---|---|
| **Privy** | Best DX, mature embedded wallet, broad auth methods, MPC-backed, smart-account add-on | Pricing scales fast at high MAU | Default for most consumer apps |
| **Dynamic** | Multi-chain native (EVM + Solana strong), good for crypto-native users | Less polished email/social UX than Privy | Multi-chain apps, DeFi front-ends |
| **Web3Auth (tKey)** | Open-core, more control over MPC, customizable | More integration work | Teams that want to own more of the stack |
| **Magic** | Email-link auth, older incumbent | Less crypto-native, lagging on new features | Web2-leaning crypto apps |
| **Para (Capsule)** | MPC + passkey, good for cross-app wallets | Smaller ecosystem | Specific niches |
| **Turnkey (consumer mode)** | API-first, HSM-backed, very flexible | More integration work, less "embedded UX" out of box | Teams comfortable building UX |

**Decision criteria:**
- Speed-to-market matters most → **Privy**
- Multi-chain (Solana/EVM/Bitcoin) matters most → **Dynamic** or **Web3Auth**
- Cost matters most at scale → **Web3Auth (self-hosted)** or **Turnkey**
- You need passkey-first → **Privy** or **Para**

### 3. Tier 2 selection (institutional)

| Vendor | Strengths | Weaknesses | Best for |
|---|---|---|---|
| **Fireblocks** | Mature, broad chain support, comprehensive policy engine | Expensive, slow to support new chains/L2s, opaque pricing | Compliance-focused, enterprise |
| **Fordefi** | Modern UX, faster on new chains, better DeFi UX | Smaller team, less institutional brand recognition | DeFi-native teams |
| **Turnkey** | API-first, cheap at scale, HSM-backed | Less DeFi-tooling, more DIY | High-volume programmatic signing |
| **BitGo** | Multisig heritage, strong custodial brand | UX is dated | Regulated/conservative institutions |
| **Anchorage** | US-regulated crypto bank, qualified custodian | Slower onboarding, US-focused | US institutions needing qualified custody |
| **Cobo** | Asia presence, mixed custody/MPC | Less western traction | APAC operations |

**Decision criteria:**
- Need regulated qualified custody → **Anchorage** or **BitGo**
- Need policy + audit at enterprise scale → **Fireblocks**
- Need DeFi-native UX (signing through MetaMask-like patterns) → **Fordefi**
- Need programmatic signing at high volume with custom logic → **Turnkey**
- Need APAC coverage → **Cobo** or **Fordefi**

### 4. Hidden costs to ask about

Vendor pricing is usually "talk to sales." Pin them down on:

**Tier 1:**
- Per-MAU (monthly active user with a wallet)
- Per-signed-transaction (some charge for signing, not just MAU)
- Per-chain support (some charge extra for non-default chains)
- White-labeling cost
- Sub-account / multi-tenant cost

**Tier 2:**
- Per-workspace fee (typically $1k-$10k/mo)
- Per-tx fee on top of workspace
- Per-asset enablement fee (some charge to add a new chain or token)
- Setup / onboarding fees
- API call costs
- Custom policy engine fees

**Common gotcha:** "$X per MAU" but MAU is defined as anyone who logged in once that month, not anyone who signed a tx. Padding can 2-5x your bill.

### 5. Lock-in considerations

WaaS lock-in is real:
- Tier 1: user wallet keys are split between vendor and user. Switching vendors means re-onboarding users.
- Tier 2: shares are vendor-specific. Moving from Fireblocks to Fordefi means re-keying everything.

**Mitigation:**
- For Tier 1: design your product to be wallet-vendor-agnostic at the data layer. Wallet address is the user identifier in your DB, not vendor-specific IDs.
- For Tier 2: avoid baking vendor-specific concepts into your domain. Treat the vendor as a signer adapter.

### 6. Build vs buy

**Build:** justified only when:
- Your scale makes per-tx vendor fees > engineering cost
- You have specific compliance needs no vendor offers (rare)
- You're building the WaaS itself

**Buy:** the default for 95%+ of products. The cryptography of MPC is hard, the policy engine is harder, and the operational reliability is hardest. You don't want to be answering "why did our HSM lose a share at 3am" for the rest of your career.

**Hybrid:** common pattern is buy Tier 1, partial-build Tier 2:
- Use Privy/Dynamic for users (buy)
- Use a managed HSM (AWS CloudHSM / GCP Cloud HSM) + custom policy layer for treasury (partial-build)
- Falls back to a Tier-2 WaaS only for high-value cold storage (buy)

### 7. Migration paths

Migrating WaaS is one of the most painful operations in crypto product land.

**Tier 1 migration (e.g., Magic → Privy):**
1. Run both in parallel for a transition period
2. New users go to new vendor
3. Old users prompted to "re-secure" their wallet, which generates a new MPC share on new vendor
4. Old vendor's data exported and stored (in case rollback needed)
5. After 90% migration, drop old vendor

**Tier 2 migration (e.g., Fireblocks → Fordefi):**
1. Onboard new vendor, configure policies
2. Generate new addresses
3. Move funds incrementally (signed by old vendor)
4. Update systems to point to new vendor
5. Decommission old vendor

Both take 3-12 months for production systems. Plan accordingly.

## Rationalizations to reject

| Excuse | Reality |
|---|---|
| "We'll just use Privy for everything" | Privy is Tier 1. Treasury policy and approver workflows are not its job. |
| "Fireblocks for users is just Privy with extra features" | Fireblocks per-user pricing is 10-100x Privy's. You'll burn money. |
| "Vendor lock-in isn't a real concern" | It is, and migrations are 6-12 month projects. Design with switchability in mind. |
| "We can build our own MPC, it's just math" | MPC implementations have subtle bugs that lose funds. Don't roll your own. |
| "We don't need a Tier 2, our team approves manually" | "Manually approve" with no policy engine is a single point of failure (the approver). |

## Verification

Before committing to a vendor:
- [ ] Pricing modeled at 6-month, 12-month, 24-month projected scale
- [ ] All chains you support are confirmed (not "on roadmap")
- [ ] Migration plan documented (how would you switch in 12 months?)
- [ ] Vendor's SLA reviewed (uptime guarantees, support response)
- [ ] Compliance certifications match your needs (SOC 2, ISO 27001, MAS, NYDFS BitLicense, etc.)
- [ ] Pilot integration completed before signing
- [ ] Reference customers contacted (vendor will provide; ask about pain points)
- [ ] Contractual terms reviewed (data export rights, termination, price increase caps)

## Cheat sheet: typical stacks

**Consumer DEX / wallet app:**
- Tier 1: Privy or Dynamic
- Tier 2: not needed (non-custodial), or minimal HSM for operational gas wallet

**Crypto card product:**
- Tier 1: Privy (user wallet) or custodial (your own infra + Tier 2)
- Tier 2: Fireblocks or Fordefi (treasury, settlement, fee payment)

**DeFi protocol team:**
- Tier 1: not needed (users connect their own wallet)
- Tier 2: Safe multisig (governance) + Fireblocks/Fordefi (treasury) + optional HSM for relayer

**CEX / brokerage:**
- Tier 1: own custodial wallet system
- Tier 2: BitGo or Anchorage (regulated custody) + Fireblocks (operational)

**B2B crypto SaaS:**
- Tier 1: Turnkey or Dynamic (programmatic per-customer wallets)
- Tier 2: Fireblocks (own treasury)
