# Contributing to crypto-infra-skills

Thanks for the interest. This marketplace is opinionated, so contributions are evaluated against a specific bar.

## The bar

Every skill, agent, and command must read like it was written by someone who has shipped that kind of system to production at a serious crypto company. If a senior engineer who has run an indexer or built a wallet system in production would roll their eyes at the content, it doesn't ship.

## What to contribute

### High-value contributions

- **New skills** in underserved backend areas (e.g., stablecoin issuance backend, custodial exchange settlement, RWA tokenization backend, cross-chain message bus design)
- **Reference deep-dives** that expand on existing skills with production-grade detail
- **Bug fixes** to existing skill content (incorrect facts, outdated vendor information, dead URLs)
- **New slash commands** that orchestrate existing skills for a specific workflow

### Lower-value contributions

- Rewording existing content for stylistic preference
- Adding content that duplicates what's already in another skill
- "Beginner-friendly" rewrites that water down the assumed expertise level

## How to add a new skill

1. **Open an issue first** describing the proposed skill. We'll discuss whether it fits the marketplace's scope before you write content.

2. **Create the skill directory:**
   ```
   plugins/web3-backend/skills/<your-skill-name>/SKILL.md
   plugins/web3-backend/skills/<your-skill-name>/references/  # optional
   ```

3. **Follow the existing structure** (see `CLAUDE.md` for the template). Reference `wallet-infrastructure/SKILL.md` as the canonical example.

4. **Frontmatter description matters most.** Spend time making the description specific — it determines whether Claude auto-loads your skill. Bad descriptions = skills that never trigger.

5. **Test it locally:**
   ```
   /plugin marketplace add .
   /plugin install web3-backend@crypto-infra-skills
   ```
   Restart Claude Code, then prompt with the kinds of questions your skill should handle. If it doesn't auto-trigger, the description needs work.

6. **Submit a PR** with:
   - The new skill files
   - An updated `marketplace.json` and `plugin.json` if you bump the version
   - A note in the PR description about which prompts you tested

## Style guide

### Tone
- Direct. State conclusions, then justify.
- Opinionated where there's a real best answer.
- Honest about trade-offs without hiding behind "it depends."

### Format
- Use tables for vendor comparisons and decision matrices.
- Use code blocks for concrete examples (not pseudocode unless explicitly labeled).
- Use the "Rationalizations to reject" pattern for common pitfalls.
- End with a "Verification" checklist.

### What to avoid
- Generic software-engineering advice ("write tests", "use version control")
- Marketing copy from vendor websites
- Speculation about unreleased features
- "It depends" without concrete decision rules

## Code in skills

Code samples should:
- Actually run (or be marked as illustrative)
- Be in the language the target audience uses (Go for backend infra, Solidity for contract-adjacent stuff if any)
- Show real-world patterns, not toy examples

## Versioning

We use semver:
- **Patch** (`0.1.x`) — typo fixes, small content additions, no breaking changes
- **Minor** (`0.x.0`) — new skills, new commands, new agents
- **Major** (`x.0.0`) — breaking changes to skill names, removed skills, restructured commands

Bump both `marketplace.json` and `plugin.json` versions for any user-facing change.

## License

By contributing you agree your contributions will be licensed under MIT.

## Questions

Open an issue. Discussions happen there before PRs.
