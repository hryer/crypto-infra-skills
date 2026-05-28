---
name: vendor-fireblocks
description: Backend integration guide for Fireblocks — institutional MPC custody platform. Covers vault accounts, transaction creation API, webhook handling, policy engine, AWS Co-Signer, and the Fireblocks MCP server. Use whenever the user is integrating Fireblocks for treasury, settlement, fee payment, or institutional signing at the services / backend level.
---

# Fireblocks

Institutional-tier (Tier 2) MPC custody platform. For decision-level context — when to pick Fireblocks vs Fordefi / Turnkey / BitGo — start at [[category-wallet-custody]].

## What this vendor is for

Fireblocks is the incumbent institutional crypto-custody platform. It runs MPC under the hood, exposes a policy engine for multi-approver workflows, and supports a broad chain matrix. It is the default pick for compliance-focused / enterprise treasury and operational wallets.

## Custody / data / pricing model

- **Custody model:** Custodial-with-MPC. Fireblocks holds key shares; you control policy.
- **What Fireblocks guarantees:** SOC 2 Type II, insurance (Lloyd's), policy enforcement, audit trail.
- **What it does NOT guarantee:** Non-custodial sovereignty — Fireblocks can freeze if compelled.
- **Pricing:** Per-workspace base + per-tx + per-asset enablement + setup. Pin all of these down — see [[category-wallet-custody]] "Hidden vendor costs."

## Auth & API setup

- API key + API secret (RSA private key). **Never** persist the RSA private key in plaintext.
- Source secrets from AWS KMS / GCP KMS / Vault. The signing key for API requests is just as sensitive as the user funds.
- Use separate API users per environment (sandbox / mainnet) with separate policy scopes.
- Co-Signer deployment (AWS Co-Signer or on-prem) is required for automated outbound transactions.

<!-- TODO: fill in concrete auth steps from https://developers.fireblocks.com/reference/api-overview -->

## SDK usage

### TypeScript

Official SDK: `@fireblocks/ts-sdk` (replaces the older `fireblocks-sdk`).

<!-- TODO: paste minimal createTransaction + getTransaction example using @fireblocks/ts-sdk -->

Strict mode on. Wrap every Fireblocks call in a typed error mapper — Fireblocks error codes are stable; map them to your domain errors at the boundary.

### Golang

Official SDK: `github.com/fireblocks/fireblocks-sdk-go`.

<!-- TODO: paste minimal createTransaction + getTransaction example -->

Follow [Effective Go](https://go.dev/doc/effective_go) — `ctx context.Context` first param on every Fireblocks call, wrap errors with `fmt.Errorf("fireblocks createTx: %w", err)`, no goroutine leaks on polling loops.

## Webhook / callback handling

Fireblocks pushes webhooks for tx status changes (`TRANSACTION_CREATED`, `TRANSACTION_STATUS_UPDATED`, `TRANSACTION_APPROVAL_STATUS_UPDATED`, etc.).

- **Signature verification:** Fireblocks signs every webhook with their RSA public key. **Always verify** before processing — do not trust the body.
- **Idempotency key:** use the Fireblocks `txId` as the idempotency key in your consumer.
- **Retry semantics:** Fireblocks retries on non-2xx; consumer must be idempotent.

**Infra wiring:**
- Receive webhook → verify signature → enqueue to AWS SQS / GCP Pub/Sub / Kafka → ACK back to Fireblocks immediately.
- Worker consumes from queue → updates DB transactionally.
- **Reconciliation job:** poll `GET /v1/transactions` every 5 minutes for txs in non-terminal status as a backstop for missed webhooks.

<!-- TODO: paste signature verification snippet (Node + Go) -->

## Common integration mistakes

- Holding the API RSA private key in env vars or DB plaintext → use KMS.
- Skipping webhook signature verification "because it's behind a VPN" → still verify; defense in depth.
- Treating `TRANSACTION_STATUS_UPDATED` as final without checking the actual `status` field → status flips through many intermediate values; only `COMPLETED` / `FAILED` / `CANCELLED` are terminal.
- Forgetting that vault accounts are workspace-scoped — sandbox and mainnet are separate workspaces with separate IDs.
- Polling getTransactions in a `time.Sleep(1 * time.Second)` loop → use webhooks as primary, polling only as backstop.

<!-- TODO: extend with vendor-specific gotchas as encountered -->

## MCP integration

Fireblocks publishes an MCP server (`@fireblocks/mcp-server`) that exposes the API as agent tools. **The skill stands alone — you only need this if you want the agent to query a live workspace** (e.g. "check my vault balances", "what's the active policy", "show me recent transactions").

### Setup

The user supplies credentials and the agent gets the tools automatically — no code changes. Register the server in the MCP client config (Claude Desktop / Cursor / `.mcp.json`):

```json
{
  "mcpServers": {
    "fireblocks": {
      "command": "npx",
      "args": ["@fireblocks/mcp-server"],
      "env": {
        "FIREBLOCKS_API_KEY": "<api-key>",
        "FIREBLOCKS_PRIVATE_KEY_PATH": "/abs/path/to/fireblocks_secret.key",
        "ENABLE_WRITE_OPERATIONS": "false",
        "FIREBLOCKS_API_BASE_URL": "https://api.fireblocks.io/v1"
      }
    }
  }
}
```

- Node 18+, stdio transport.
- `FIREBLOCKS_PRIVATE_KEY_PATH` is a **path to the RSA secret file**, not the key contents — keep it `chmod 600` and never inline the key into the config JSON.
- For sandbox, point `FIREBLOCKS_API_BASE_URL` at the sandbox host and use a separate sandbox API user (vault IDs differ per workspace — see Common integration mistakes).

### Safety posture (read-only by default)

- **Keep `ENABLE_WRITE_OPERATIONS=false`** and use a **Viewer-role** API user. With writes off, only the read tools below are exposed — safe for "check my vault" / review workflows.
- The only write tool is `create_transaction`, gated behind that flag. **Do not enable it for agent use.** An agent that can move funds on a custody platform is a foot-gun; create transactions through the SDK with the policy engine and human approval instead. See [[category-wallet-custody]] safety guidance.

### Tools

Read tools (available with `ENABLE_WRITE_OPERATIONS=false`):

| When the user wants to… | Tool(s) |
|---|---|
| Check vault balances / holdings | `get_vault_accounts`, `get_vault_account_by_id`, `get_vault_account_asset`, `get_vault_balance_by_asset`, `get_vault_assets` |
| Inspect transactions | `get_transactions` (filter by status / date / source / destination) |
| Review governance | `get_active_policy`, `get_whitelist_ip_addresses`, `get_users` |
| List wallets & venues | `get_external_wallets`, `get_internal_wallets`, `get_exchange_accounts`, `get_network_connections` |
| Look up chain / asset metadata | `get_blockchains`, `get_blockchain_asset`, `get_assets` |

Write tool (exposed only when `ENABLE_WRITE_OPERATIONS=true`; avoid for agent flows): `create_transaction`.

## Latest docs reference

- Official docs: https://developers.fireblocks.com/
- Webhook reference: https://developers.fireblocks.com/reference/webhooks-overview
- TypeScript SDK: https://github.com/fireblocks/ts-sdk
- Go SDK: https://github.com/fireblocks/fireblocks-sdk-go
- MCP server: https://github.com/fireblocks/fireblocks-mcp — npm `@fireblocks/mcp-server`
- Last-verified: `2026-05-29` <!-- TODO: bump on each re-check -->

## When NOT to use this vendor

- **Consumer-facing user wallet** → wrong tier; use [[vendor-privy]] or [[vendor-dynamic]] (Tier 1).
- **DeFi-native flows on newest L2s** → Fireblocks chain support lags; consider [[vendor-fordefi]].
- **High-volume programmatic signing where per-tx cost dominates** → consider [[vendor-turnkey]].
- **You need on-chain transparency of approvals** → use a Safe multisig instead.

## Cross-references

- Parent category: [[category-wallet-custody]]
- Reviewed by: [[web3-backend-reviewer]] (§7 vendor integrations, §9 webhook handling)
- Security-reviewed by: [[wallet-security-auditor]] (MPC ceremony, key share storage, policy engine review)
- Alternatives in same tier: [[vendor-fordefi]], [[vendor-turnkey]]
