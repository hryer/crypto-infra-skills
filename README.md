# crypto-infra-skills

Production-grade engineering skills, agents, and commands for AI coding agents — built for **crypto / Web3 backend** work and covering the full lifecycle from spec to ship.

This is a [Claude Code](https://claude.com/claude-code) plugin. It teaches your agent how to **write**, **test**, **review**, and **ship** wallet infrastructure, indexers, matching engines, swap integrations, crypto-card products, and the vendor integrations behind them — with the rigor of a staff engineer who has shipped these systems to production.

Language guidance is pinned to canonical references so authoring and review share one definition of "good":

- **Go** → [Effective Go](https://go.dev/doc/effective_go)
- **TypeScript** → Effective TypeScript (Dan Vanderkam)
- **Rust** → [The Rust Book](https://doc.rust-lang.org/book/) + [Rust API Guidelines](https://rust-lang.github.io/api-guidelines/)

---

## What's inside

| Layer | Count | What it is |
|---|---|---|
| **Commands** | 7 | User-facing entry points (`/build`, `/ship`, …) that compose personas + skills |
| **Agents (personas)** | 5 | Single-role reviewers/auditors that produce a structured report |
| **Skills** | 55 | Workflows with steps + exit criteria, in three families (below) |
| **Guidelines** | 5 | Shared checklists (security, performance, testing, a11y, orchestration) |

The three layers compose like this (see [agents/README.md](agents/README.md) for detail):

| Layer | Job | Example |
|---|---|---|
| **Skill** | The *how* — a workflow, invoked from inside a persona or command | `incremental-implementation` |
| **Persona** | The *who* — adopts a viewpoint, produces a report | `web3-backend-reviewer` |
| **Command** | The *when* — composes personas + skills | `/review`, `/ship` |

---

## Install

In Claude Code:

```
# Add this repo as a plugin marketplace
/plugin marketplace add hryer/crypto-infra-skills

# Install the plugin
/plugin install crypto-infra-skills@crypto-infra-skills
```

**Local development** (from a clone of this repo):

```
/plugin marketplace add .
/plugin install crypto-infra-skills@crypto-infra-skills
```

Once installed, the commands (`/build`, `/ship`, …) become available, agents can be invoked, and skills auto-trigger based on what you're doing.

---

## How to use

### 1. Commands — the main entry points

Type a slash command to run a workflow. Each composes the relevant skills and personas.

| Command | What it does |
|---|---|
| [`/spec`](.claude/commands/spec.md) | Write a structured specification before writing code |
| [`/plan`](.claude/commands/plan.md) | Break work into small verifiable tasks with acceptance criteria + dependency order |
| [`/build`](.claude/commands/build.md) | Implement the next task incrementally — build, test, verify, commit |
| [`/test`](.claude/commands/test.md) | TDD workflow — write failing tests, implement, verify (Prove-It pattern for bugs) |
| [`/review`](.claude/commands/review.md) | Five-axis code review — correctness, readability, architecture, security, performance |
| [`/code-simplify`](.claude/commands/code-simplify.md) | Reduce complexity without changing behavior |
| [`/ship`](.claude/commands/ship.md) | Pre-launch checklist via parallel fan-out to specialist personas → go/no-go |

A typical flow: `/spec` → `/plan` → `/build` (repeat per task) → `/review` → `/ship`.

### 2. Skills — auto-triggered or invoked

Skills load automatically when your request matches their description (e.g., asking about a Fireblocks integration pulls in `vendor-fireblocks`). You can also invoke one explicitly by name. Skills are the knowledge layer — you rarely call them directly; commands and agents pull them in.

### 3. Agents — when you want one expert's perspective

Invoke a persona for a focused review. They produce a structured report and **do not call each other** (you or a command orchestrate them).

| Agent | Role | Use for |
|---|---|---|
| [code-reviewer](agents/code-reviewer.md) | Senior Staff Engineer | Five-axis review before merge |
| [security-auditor](agents/security-auditor.md) | Security Engineer | OWASP-style vulnerability audit |
| [test-engineer](agents/test-engineer.md) | QA Engineer | Test strategy + coverage |
| [web3-backend-reviewer](agents/web3-backend-reviewer.md) | Web3 Staff Engineer | Crypto backend review: reorg, idempotency, ledger conservation, vendor integrations, key mgmt, infra, distributed systems, debugging |
| [wallet-security-auditor](agents/wallet-security-auditor.md) | Wallet Security Engineer | Adversarial review of custody, signing, MPC/multisig, ERC-4337, Solana programs |

---

## The skill taxonomy

55 skills in three families, distinguished by name prefix.

### `category-*` — decision frameworks (13)

The **entry point** for a problem area. Each frames the use case, lays out vendor options, gives a build-it-yourself path, includes inline Go/TS/Rust + infra best practices, and branches to `vendor-*` skills.

| Skill | Covers |
|---|---|
| [category-wallet-custody](skills/category-wallet-custody/SKILL.md) | Custody models + WaaS vendor selection (two-tier) |
| [category-swap-integration](skills/category-swap-integration/SKILL.md) | DEX aggregators vs direct routers |
| [category-token-security-screening](skills/category-token-security-screening/SKILL.md) | Honeypot / rug detection |
| [category-price-feeds](skills/category-price-feeds/SKILL.md) | Off-chain APIs vs on-chain oracles |
| [category-onchain-analytics](skills/category-onchain-analytics/SKILL.md) | Labeled-wallet data, smart-money signals |
| [category-rpc-and-indexer](skills/category-rpc-and-indexer/SKILL.md) | RPC + event indexing + fallback |
| [category-smart-contract-testing](skills/category-smart-contract-testing/SKILL.md) | Simulation, fork tests, Anchor/Foundry |
| [category-multichain-tx-management](skills/category-multichain-tx-management/SKILL.md) | Nonce/gas/RBF across chains |
| [category-internal-ledger](skills/category-internal-ledger/SKILL.md) | Double-entry, conservation, idempotency |
| [category-matching-engine](skills/category-matching-engine/SKILL.md) | Order book, matching, HFT hot path |
| [category-mev-and-keepers](skills/category-mev-and-keepers/SKILL.md) | Searcher/keeper bots, revm, Flashbots |
| [category-crypto-card](skills/category-crypto-card/SKILL.md) | Card issuing, auth/settlement, ledger |
| [category-onramp-offramp](skills/category-onramp-offramp/SKILL.md) | Fiat on/off-ramp vendors |

### `vendor-*` — integration guides (19)

How to integrate a specific vendor **at the backend level**: auth, KMS-managed secrets, SDK usage (TS + Go), webhook handling + idempotency, infra wiring, common mistakes, MCP integration where available, and a "when NOT to use this" section.

- **Wallet/custody:** [fireblocks](skills/vendor-fireblocks/SKILL.md) · [fordefi](skills/vendor-fordefi/SKILL.md) · [privy](skills/vendor-privy/SKILL.md) · [dynamic](skills/vendor-dynamic/SKILL.md) · [turnkey](skills/vendor-turnkey/SKILL.md)
- **RPC/indexer:** [alchemy](skills/vendor-alchemy/SKILL.md) · [quicknode](skills/vendor-quicknode/SKILL.md) · [helius](skills/vendor-helius/SKILL.md)
- **Swap:** [lifi](skills/vendor-lifi/SKILL.md)
- **Token security:** [birdeye](skills/vendor-birdeye/SKILL.md)
- **Contract testing/debug:** [tenderly](skills/vendor-tenderly/SKILL.md)
- **Prices/oracles:** [coingecko](skills/vendor-coingecko/SKILL.md) · [coinmarketcap](skills/vendor-coinmarketcap/SKILL.md) · [coinapi](skills/vendor-coinapi/SKILL.md) · [chainlink](skills/vendor-chainlink/SKILL.md)
- **Analytics:** [nansen](skills/vendor-nansen/SKILL.md)
- **Card issuers:** [reap](skills/vendor-reap/SKILL.md) · [rain](skills/vendor-rain/SKILL.md) · [dcs](skills/vendor-dcs/SKILL.md)

> Vendor skills carry a `last-verified` date and a `<!-- TODO: fill in from latest vendor docs -->` marker — refresh them against current vendor docs before relying on the integration detail.

### Generic engineering skills (23)

Language-agnostic workflow + quality skills used across any task: spec/plan/build/test/review/ship discipline, debugging, git, CI/CD, API design, documentation, and the meta-skills. The authoring/testing/shipping skills carry inline **Go + TypeScript + Rust + Web3** best practices:

- [incremental-implementation](skills/incremental-implementation/SKILL.md) — canonical home for language authoring idioms
- [test-driven-development](skills/test-driven-development/SKILL.md) — canonical home for testing idioms
- [shipping-and-launch](skills/shipping-and-launch/SKILL.md) — canonical home for the pre-launch checklist (incl. Web3 readiness)
- [ci-cd-and-automation](skills/ci-cd-and-automation/SKILL.md) — Go/TS/Rust + Foundry/Anchor pipelines
- [code-simplification](skills/code-simplification/SKILL.md) — idiomatic refactors per language

Others: `api-and-interface-design`, `browser-testing-with-devtools`, `code-review-and-quality`, `context-engineering`, `debugging-and-error-recovery`, `deprecation-and-migration`, `documentation-and-adrs`, `doubt-driven-development`, `frontend-ui-engineering`, `git-workflow-and-versioning`, `idea-refine`, `interview-me`, `performance-optimization`, `planning-and-task-breakdown`, `security-and-hardening`, `source-driven-development`, `spec-driven-development`, `using-agent-skills`.

---

## Example workflows

**Integrate a custodial wallet vendor**
> "I'm adding Fireblocks for treasury signing."

Auto-loads [category-wallet-custody](skills/category-wallet-custody/SKILL.md) (custody decision) → [vendor-fireblocks](skills/vendor-fireblocks/SKILL.md) (backend integration). Build with `/build`, then review with the [web3-backend-reviewer](agents/web3-backend-reviewer.md) and [wallet-security-auditor](agents/wallet-security-auditor.md) before `/ship`.

**Build a Solana indexer with a fallback**
> "Index our program's events on Solana with a backup RPC."

Loads [category-rpc-and-indexer](skills/category-rpc-and-indexer/SKILL.md) → [vendor-helius](skills/vendor-helius/SKILL.md) (primary) + [vendor-quicknode](skills/vendor-quicknode/SKILL.md) (fallback). Reviewer checks reorg/finality handling and the reconciliation backstop.

**Ship a matching engine change**
> `/ship`

Fans out to the reviewer personas, runs the pre-launch checklist (including Web3 readiness: kill switch, RPC failover, reorg thresholds), and returns a go/no-go.

---

## Repository layout

```
.
├── .claude/commands/      # 7 slash commands
├── .claude-plugin/        # plugin.json + marketplace.json
├── agents/                # 5 personas + README
├── skills/                # 55 skills (category-*, vendor-*, generic)
├── guidelines/            # 5 shared checklists
├── CONTRIBUTING.md
└── LICENSE
```

---

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md). The bar: every skill/agent/command must read like it was written by someone who has shipped that kind of system to production at a serious crypto company.

## License

MIT — see [LICENSE](LICENSE).
