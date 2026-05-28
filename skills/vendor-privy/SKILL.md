---
name: vendor-privy
description: Backend integration guide for Privy — Tier 1 embedded wallet provider with email/social/passkey auth and MPC-backed keys. Covers backend SDK usage, JWT verification, wallet creation, server-side signing, webhook handling, and smart-account add-on. Use whenever the user is integrating Privy for consumer wallet UX, embedded wallets, or server-side actions on behalf of a Privy-authed user.
---

# Privy

Tier-1 (consumer) embedded wallet platform. Best default for consumer apps. For decision context — Privy vs Dynamic vs Web3Auth — start at [[category-wallet-custody]].

## What this vendor is for

Privy provides email / social / passkey auth + an embedded MPC wallet that lives in the user's browser session, with one share held by Privy. From the backend, you verify Privy JWTs to authenticate users and (optionally) perform server-side actions on the user's wallet via Privy's server SDK.

## Custody / data / pricing model

- **Custody model:** Non-custodial-ish — MPC with one share held by user (browser), one share by Privy. User holds keys in the legal sense.
- **What Privy guarantees:** SOC 2 Type II, embedded wallet UX, recovery flows (passkey / email).
- **Pricing:** Per-MAU. Pin down the MAU definition (logged-in vs signed-tx). See [[category-wallet-custody]] "Hidden vendor costs."

## Auth & API setup

- App ID + App Secret (server-side). Source from KMS / Vault, **never** in client-side code or env files committed to git.
- Frontend uses the public App ID; backend uses App ID + Secret.
- JWT verification: Privy issues JWTs to the client; the backend verifies them against Privy's JWKS endpoint.

<!-- TODO: fill in concrete auth steps from https://docs.privy.io -->

## SDK usage

### TypeScript

Official SDK: `@privy-io/server-auth` (server-side), `@privy-io/react-auth` (client-side).

<!-- TODO: paste minimal `verifyAuthToken` + `getUser` example using @privy-io/server-auth -->

Strict mode on. Cache JWKS responses (Privy SDK handles this but verify timeouts).

### Golang

<!-- TODO: confirm Go SDK availability. If REST-only, build a thin verifier using github.com/golang-jwt/jwt/v5 against Privy's JWKS. -->

Follow [Effective Go](https://go.dev/doc/effective_go) — context-first, error wrapping.

## Webhook / callback handling

Privy emits webhooks for user lifecycle events (user created, wallet created, etc.).

- **Signature verification:** Privy signs webhooks; verify HMAC before processing.
- **Idempotency key:** use the Privy event ID.
- **Retry semantics:** non-2xx triggers retry.

**Infra wiring:**
- Webhook → verify → enqueue (SQS / Pub/Sub / Kafka) → worker.
- Reconciliation: `GET /v1/users` periodic sweep if user creation events are missed.

<!-- TODO: paste HMAC verification snippet -->

## Common integration mistakes

- Storing Privy App Secret in client code or in repo `.env` files committed to git.
- Trusting JWT `sub` claim without verifying the signature.
- Treating Privy embedded wallet as fully custodial — it's MPC; you cannot sign for the user without their session.
- Forgetting that disconnecting Privy means re-onboarding the user (lock-in).
- Using Privy MAU pricing assumptions that ignore inactive but logged-in users.

<!-- TODO: extend -->

## MCP integration

<!-- TODO: check if Privy offers an MCP server. If not, omit this section. -->

## Latest docs reference

- Official docs: https://docs.privy.io/
- Server SDK: https://docs.privy.io/guide/server/
- Webhook reference: https://docs.privy.io/guide/server/webhooks
- Last-verified: `<YYYY-MM-DD>` <!-- TODO: bump on each re-check -->

## When NOT to use this vendor

- **Treasury or platform float** → wrong tier; use [[vendor-fireblocks]] / [[vendor-fordefi]] (Tier 2).
- **Multi-chain Solana-heavy product** → [[vendor-dynamic]] has stronger Solana story.
- **You want to own the MPC stack** → Web3Auth (tKey) or Para.
- **Pure programmatic signing without UX** → [[vendor-turnkey]].

## Cross-references

- Parent category: [[category-wallet-custody]]
- Reviewed by: [[web3-backend-reviewer]] (§7 vendor integrations)
- Security-reviewed by: [[wallet-security-auditor]] (JWT verification, MPC share handling)
- Alternatives in same tier: [[vendor-dynamic]], [[vendor-turnkey]]
