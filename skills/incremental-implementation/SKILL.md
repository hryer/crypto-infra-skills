---
name: incremental-implementation
description: Delivers changes incrementally. Use when implementing any feature or change that touches more than one file. Use when you're about to write a large amount of code at once, or when a task feels too big to land in one step.
---

# Incremental Implementation

## Overview

Build in thin vertical slices — implement one piece, test it, verify it, then expand. Avoid implementing an entire feature in one pass. Each increment should leave the system in a working, testable state. This is the execution discipline that makes large features manageable.

## When to Use

- Implementing any multi-file change
- Building a new feature from a task breakdown
- Refactoring existing code
- Any time you're tempted to write more than ~100 lines before testing

**When NOT to use:** Single-file, single-function changes where the scope is already minimal.

## The Increment Cycle

```
┌──────────────────────────────────────┐
│                                      │
│   Implement ──→ Test ──→ Verify ──┐  │
│       ▲                           │  │
│       └───── Commit ◄─────────────┘  │
│              │                       │
│              ▼                       │
│          Next slice                  │
│                                      │
└──────────────────────────────────────┘
```

For each slice:

1. **Implement** the smallest complete piece of functionality
2. **Test** — run the test suite (or write a test if none exists)
3. **Verify** — confirm the slice works as expected (tests pass, build succeeds, manual check)
4. **Commit** -- save your progress with a descriptive message (see `git-workflow-and-versioning` for atomic commit guidance)
5. **Move to the next slice** — carry forward, don't restart

## Slicing Strategies

### Vertical Slices (Preferred)

Build one complete path through the stack:

```
Slice 1: Create a task (DB + API + basic UI)
    → Tests pass, user can create a task via the UI

Slice 2: List tasks (query + API + UI)
    → Tests pass, user can see their tasks

Slice 3: Edit a task (update + API + UI)
    → Tests pass, user can modify tasks

Slice 4: Delete a task (delete + API + UI + confirmation)
    → Tests pass, full CRUD complete
```

Each slice delivers working end-to-end functionality.

### Contract-First Slicing

When backend and frontend need to develop in parallel:

```
Slice 0: Define the API contract (types, interfaces, OpenAPI spec)
Slice 1a: Implement backend against the contract + API tests
Slice 1b: Implement frontend against mock data matching the contract
Slice 2: Integrate and test end-to-end
```

### Risk-First Slicing

Tackle the riskiest or most uncertain piece first:

```
Slice 1: Prove the WebSocket connection works (highest risk)
Slice 2: Build real-time task updates on the proven connection
Slice 3: Add offline support and reconnection
```

If Slice 1 fails, you discover it before investing in Slices 2 and 3.

## Implementation Rules

### Rule 0: Simplicity First

Before writing any code, ask: "What is the simplest thing that could work?"

After writing code, review it against these checks:
- Can this be done in fewer lines?
- Are these abstractions earning their complexity?
- Would a staff engineer look at this and say "why didn't you just..."?
- Am I building for hypothetical future requirements, or the current task?

```
SIMPLICITY CHECK:
✗ Generic EventBus with middleware pipeline for one notification
✓ Simple function call

✗ Abstract factory pattern for two similar components
✓ Two straightforward components with shared utilities

✗ Config-driven form builder for three forms
✓ Three form components
```

Three similar lines of code is better than a premature abstraction. Implement the naive, obviously-correct version first. Optimize only after correctness is proven with tests.

### Rule 0.5: Scope Discipline

Touch only what the task requires.

Do NOT:
- "Clean up" code adjacent to your change
- Refactor imports in files you're not modifying
- Remove comments you don't fully understand
- Add features not in the spec because they "seem useful"
- Modernize syntax in files you're only reading

If you notice something worth improving outside your task scope, note it — don't fix it:

```
NOTICED BUT NOT TOUCHING:
- src/utils/format.ts has an unused import (unrelated to this task)
- The auth middleware could use better error messages (separate task)
→ Want me to create tasks for these?
```

### Rule 1: One Thing at a Time

Each increment changes one logical thing. Don't mix concerns:

**Bad:** One commit that adds a new component, refactors an existing one, and updates the build config.

**Good:** Three separate commits — one for each change.

### Rule 2: Keep It Compilable

After each increment, the project must build and existing tests must pass. Don't leave the codebase in a broken state between slices.

### Rule 3: Feature Flags for Incomplete Features

If a feature isn't ready for users but you need to merge increments:

```typescript
// Feature flag for work-in-progress
const ENABLE_TASK_SHARING = process.env.FEATURE_TASK_SHARING === 'true';

if (ENABLE_TASK_SHARING) {
  // New sharing UI
}
```

This lets you merge small increments to the main branch without exposing incomplete work.

### Rule 4: Safe Defaults

New code should default to safe, conservative behavior:

```typescript
// Safe: disabled by default, opt-in
export function createTask(data: TaskInput, options?: { notify?: boolean }) {
  const shouldNotify = options?.notify ?? false;
  // ...
}
```

### Rule 5: Rollback-Friendly

Each increment should be independently revertable:

- Additive changes (new files, new functions) are easy to revert
- Modifications to existing code should be minimal and focused
- Database migrations should have corresponding rollback migrations
- Avoid deleting something in one commit and replacing it in the same commit — separate them

## Language & Domain Best Practices (Go / TypeScript / Web3)

This is the **canonical home** for how to write code well in this stack. Other skills (`test-driven-development`, `shipping-and-launch`, `code-simplification`, `ci-cd-and-automation`) reference this section rather than repeating it. Write to these standards as you implement each slice — they're not a post-hoc review step.

### Golang authoring

Conform to [Effective Go](https://go.dev/doc/effective_go). The non-negotiables:

- **Context first.** Every blocking call (RPC, DB, vendor API) takes `ctx context.Context` as its first parameter, and cancellation propagates. No `context.Background()` deep inside business logic.
- **Wrap errors, never swallow.** `fmt.Errorf("createTx: %w", err)` preserves the chain. A bare `return err` loses the call site; `_ = err` is a red flag.
- **Goroutine lifetimes are owned.** Every `go func()` has a clear completion signal (WaitGroup, errgroup, or context cancellation). A goroutine with no exit path is a leak.
- **`defer` for cleanup** (close, unlock, cancel) — co-located with acquisition.
- **Small, consumer-side interfaces.** Define the interface where it's consumed, not where it's implemented. Accept interfaces, return concrete structs.
- **Hot-path allocation awareness.** In matching-engine / order-router code, profile with `pprof`; avoid per-request allocations and reflection. See [[category-matching-engine]].
- **No naked `string` for money or secrets.** Use decimal/fixed-point for amounts; `[]byte` (zeroable) for key material.

### TypeScript authoring

Align to **Effective TypeScript** (Dan Vanderkam). The high-leverage items:

- **`strict: true`**, no implicit `any`. An `any` on a signing/money path is a bug waiting to happen.
- **Narrow, don't assert.** Prefer type guards over `as` casts; reserve assertions for genuine escape hatches.
- **Model state with discriminated unions** (`{ status: 'pending' } | { status: 'confirmed', txHash: Hex }`) so impossible states are unrepresentable.
- **Branded types** for IDs and addresses (`type Address = string & { __brand: 'Address' }`) to stop mixing a `userId` with a `walletAddress`.
- **Typed domain errors.** Don't throw raw `Error` across module boundaries; map vendor errors to your domain at the boundary.
- **Propagate `AbortSignal`** through async call chains; no orphan promises (`void` or `.catch()` every fire-and-forget).
- **Structured logging** (pino) — never `console.log` in production code, and never log secrets/addresses/PII.
- **EVM clients:** prefer [viem](https://viem.sh) (typed ABIs catch encoding mistakes at compile time); treat web3.js / ethers as legacy.

### Rust authoring

Align to [The Rust Book](https://doc.rust-lang.org/book/) and the [Rust API Guidelines](https://rust-lang.github.io/api-guidelines/). Common in this stack for Solana programs (Anchor), high-performance matching engines, and node/simulation tooling (reth, revm). The non-negotiables:

- **`Result<T, E>` + `?`, not panics.** `unwrap()` / `expect()` / `panic!` belong in tests and truly-unreachable branches only — never on a request or signing path. In Solana programs a panic aborts the transaction, but silent `unwrap` on untrusted input is still a bug.
- **Errors:** `thiserror` for library/typed errors, `anyhow` for application boundaries. Preserve context with `.context("...")`.
- **Don't fight the borrow checker with `.clone()`.** Pass `&T` / `&str` / `&[T]`; clone deliberately, not to silence an error. Excessive `.clone()` on a hot path is a review flag (see [[category-matching-engine]]).
- **Model with the type system.** Newtype pattern for IDs/addresses (`struct Address(String);`) — the Rust analogue of TS branded types and Go defined types; stops mixing a `UserId` with a `WalletAddress`. `Option<T>` over nullable; exhaustive `match`.
- **Async:** `tokio` runtime; never block (`std::fs`, blocking locks) inside `async`; mind `Send + Sync` bounds; cancel-safety on `select!`. Propagate cancellation like Go's context.
- **Money:** `rust_decimal` (or integer base units), never `f64`.
- **Checked math on-chain:** in Solana programs use `checked_add` / `checked_mul` (or `overflow-checks = true`); silent overflow = lost or minted funds.
- **Clippy is the baseline.** `cargo clippy -- -D warnings` clean; `cargo fmt` clean.

### Web3: build it right the first time

These cost almost nothing if designed in, and are painful to retrofit. The reviewer ([[web3-backend-reviewer]]) checks for them after the fact — bake them in now so they share one definition of "good":

- **Decimal / fixed-point for money and amounts** — never `float64` / JS `number`. (§1)
- **Normalize addresses at the write boundary** — EIP-55 or consistent lowercase for EVM, base58 for Solana — so DB lookups don't miss on case. (§8)
- **Idempotency keys from v1** — every external-input handler (webhook, consumer) dedupes on a stable key. Retrofitting this after a double-spend incident is the hard way. (§1, §13)
- **Never log secrets** — keys, mnemonics, full vendor payloads, signing requests. (§6)
- **Reorg-aware reads** — don't treat 1 confirmation as final; thresholds per chain. (§1)

See [[web3-backend-reviewer]] (§1 correctness, §6 key management, §11 language idioms, §13 distributed systems) and [[wallet-security-auditor]] for the full review lens.

## Working with Agents

When directing an agent to implement incrementally:

```
"Let's implement Task 3 from the plan.

Start with just the database schema change and the API endpoint.
Don't touch the UI yet — we'll do that in the next increment.

After implementing, run `npm test` and `npm run build` to verify
nothing is broken."
```

Be explicit about what's in scope and what's NOT in scope for each increment.

## Increment Checklist

After each increment, verify. Run the toolchain that matches the package you touched — don't run `npm` checks on a Go change or vice versa:

- [ ] The change does one thing and does it completely
- [ ] All existing tests still pass — TS: `npm test` · Go: `go test ./...` · Rust: `cargo test` (or `cargo nextest run`)
- [ ] The build succeeds — TS: `npm run build` · Go: `go build ./...` · Rust: `cargo build`
- [ ] Static checks pass — TS: `npx tsc --noEmit` + `npm run lint` · Go: `go vet ./...` + `golangci-lint run` + `gofmt -l` (clean) · Rust: `cargo clippy -- -D warnings` + `cargo fmt --check`
- [ ] The new functionality works as expected
- [ ] The change is committed with a descriptive message

**Note:** Run each verification command after a change that could affect it. After a successful run, don't repeat the same command unless the code has changed since — re-running on unchanged code adds no information.

## Common Rationalizations

| Rationalization | Reality |
|---|---|
| "I'll test it all at the end" | Bugs compound. A bug in Slice 1 makes Slices 2-5 wrong. Test each slice. |
| "It's faster to do it all at once" | It *feels* faster until something breaks and you can't find which of 500 changed lines caused it. |
| "These changes are too small to commit separately" | Small commits are free. Large commits hide bugs and make rollbacks painful. |
| "I'll add the feature flag later" | If the feature isn't complete, it shouldn't be user-visible. Add the flag now. |
| "This refactor is small enough to include" | Refactors mixed with features make both harder to review and debug. Separate them. |
| "Let me run the build command again just to be sure" | After a successful run, repeating the same command adds nothing unless the code has changed since. Run it again after subsequent edits, not as reassurance. |

## Red Flags

- More than 100 lines of code written without running tests
- Multiple unrelated changes in a single increment
- "Let me just quickly add this too" scope expansion
- Skipping the test/verify step to move faster
- Build or tests broken between increments
- Large uncommitted changes accumulating
- Building abstractions before the third use case demands it
- Touching files outside the task scope "while I'm here"
- Creating new utility files for one-time operations
- Running the same build/test command twice in a row without any intervening code change

## Verification

After completing all increments for a task:

- [ ] Each increment was individually tested and committed
- [ ] The full test suite passes
- [ ] The build is clean
- [ ] The feature works end-to-end as specified
- [ ] No uncommitted changes remain
