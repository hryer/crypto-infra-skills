---
name: shipping-and-launch
description: Prepares production launches. Use when preparing to deploy to production. Use when you need a pre-launch checklist, when setting up monitoring, when planning a staged rollout, or when you need a rollback strategy.
---

# Shipping and Launch

## Overview

Ship with confidence. The goal is not just to deploy — it's to deploy safely, with monitoring in place, a rollback plan ready, and a clear understanding of what success looks like. Every launch should be reversible, observable, and incremental.

## When to Use

- Deploying a feature to production for the first time
- Releasing a significant change to users
- Migrating data or infrastructure
- Opening a beta or early access program
- Any deployment that carries risk (all of them)

## The Pre-Launch Checklist

### Code Quality

- [ ] All tests pass (unit, integration, e2e)
- [ ] Build succeeds with no warnings
- [ ] Lint and type checking pass
- [ ] Code reviewed and approved
- [ ] No TODO comments that should be resolved before launch
- [ ] No `console.log` debugging statements in production code
- [ ] Error handling covers expected failure modes

### Security

- [ ] No secrets in code or version control
- [ ] `npm audit` shows no critical or high vulnerabilities
- [ ] Input validation on all user-facing endpoints
- [ ] Authentication and authorization checks in place
- [ ] Security headers configured (CSP, HSTS, etc.)
- [ ] Rate limiting on authentication endpoints
- [ ] CORS configured to specific origins (not wildcard)

### Performance

- [ ] Core Web Vitals within "Good" thresholds
- [ ] No N+1 queries in critical paths
- [ ] Images optimized (compression, responsive sizes, lazy loading)
- [ ] Bundle size within budget
- [ ] Database queries have appropriate indexes
- [ ] Caching configured for static assets and repeated queries

### Accessibility

- [ ] Keyboard navigation works for all interactive elements
- [ ] Screen reader can convey page content and structure
- [ ] Color contrast meets WCAG 2.1 AA (4.5:1 for text)
- [ ] Focus management correct for modals and dynamic content
- [ ] Error messages are descriptive and associated with form fields
- [ ] No accessibility warnings in axe-core or Lighthouse

### Infrastructure

- [ ] Environment variables set in production
- [ ] Database migrations applied (or ready to apply)
- [ ] DNS and SSL configured
- [ ] CDN configured for static assets
- [ ] Logging and error reporting configured
- [ ] Health check endpoint exists and responds

### Web3 / On-chain readiness

For any crypto backend, these block launch. The review lens for each lives in [[web3-backend-reviewer]] — this is the pre-ship checklist version.

- [ ] **RPC failover** configured — primary + fallback vendor, not a single endpoint (see [[category-rpc-and-indexer]]; e.g. [[vendor-alchemy]] ↔ [[vendor-quicknode]])
- [ ] **Kill switch / pause** — can halt outbound signing or processing without restarting the process (§4)
- [ ] **Reorg-safe confirmation thresholds** set per chain — no action finalized on 1 confirmation (§1)
- [ ] **Vendor webhook reconciliation job** live — REST-poll backstop for missed webhooks (§9)
- [ ] **Key rotation runbook** exists and has been dry-run (see [[wallet-security-auditor]]; secrets in KMS, never plaintext)
- [ ] **Gas / fee + nonce monitoring** — alert on stuck txs, fee spikes, nonce gaps
- [ ] **Contract / program addresses pinned + verified** against a registry, not hard-coded unverified strings (§7)
- [ ] **Idempotency** verified on every external-input handler under retry / double-delivery (§1, §13)

### Language-specific build

**Go:**
- [ ] Static binary where applicable (`CGO_ENABLED=0`), reproducible build
- [ ] Graceful shutdown on `SIGTERM` via `context` cancellation (drain in-flight work, close DB/queue cleanly)
- [ ] `pprof` endpoint exposed only on an internal port, never public
- [ ] Version / commit stamped via `-ldflags "-X main.version=..."`
- [ ] `go vet` + `golangci-lint` clean in CI (see [[ci-cd-and-automation]])

**TypeScript:**
- [ ] Production build excludes dev dependencies
- [ ] Source maps generated and handled (uploaded to error reporter, not served publicly)
- [ ] Structured logging on (pino), no `console.log`, no secrets/addresses in logs
- [ ] `tsc --noEmit` + lint clean in CI

**Rust:**
- [ ] `cargo build --release` with a tuned release profile (`lto = true`, `codegen-units = 1`, `strip = true`; `panic = "abort"` only if no unwinding is relied on)
- [ ] Graceful shutdown via `tokio::signal` (drain in-flight work, close DB/queue cleanly)
- [ ] `cargo clippy -- -D warnings` + `cargo fmt --check` + `cargo audit` (or `cargo deny`) clean in CI
- [ ] Panics don't leak across the API boundary (catch / convert at handlers; `unwrap` only where truly unreachable)
- [ ] **Solana programs:** verifiable build (`anchor build` / `solana-verify`); program **upgrade authority** is a multisig/governance key, not a hot EOA (security-critical — see [[wallet-security-auditor]]); deploy + upgrade runbook documented

### Documentation

- [ ] README updated with any new setup requirements
- [ ] API documentation current
- [ ] ADRs written for any architectural decisions
- [ ] Changelog updated
- [ ] User-facing documentation updated (if applicable)

## Feature Flag Strategy

Ship behind feature flags to decouple deployment from release:

```typescript
// Feature flag check
const flags = await getFeatureFlags(userId);

if (flags.taskSharing) {
  // New feature: task sharing
  return <TaskSharingPanel task={task} />;
}

// Default: existing behavior
return null;
```

**Feature flag lifecycle:**

```
1. DEPLOY with flag OFF     → Code is in production but inactive
2. ENABLE for team/beta     → Internal testing in production environment
3. GRADUAL ROLLOUT          → 5% → 25% → 50% → 100% of users
4. MONITOR at each stage    → Watch error rates, performance, user feedback
5. CLEAN UP                 → Remove flag and dead code path after full rollout
```

**Rules:**
- Every feature flag has an owner and an expiration date
- Clean up flags within 2 weeks of full rollout
- Don't nest feature flags (creates exponential combinations)
- Test both flag states (on and off) in CI

## Staged Rollout

### The Rollout Sequence

```
1. DEPLOY to staging
   └── Full test suite in staging environment
   └── Manual smoke test of critical flows

2. DEPLOY to production (feature flag OFF)
   └── Verify deployment succeeded (health check)
   └── Check error monitoring (no new errors)

3. ENABLE for team (flag ON for internal users)
   └── Team uses the feature in production
   └── 24-hour monitoring window

4. CANARY rollout (flag ON for 5% of users)
   └── Monitor error rates, latency, user behavior
   └── Compare metrics: canary vs. baseline
   └── 24-48 hour monitoring window
   └── Advance only if all thresholds pass (see table below)

5. GRADUAL increase (25% -> 50% -> 100%)
   └── Same monitoring at each step
   └── Ability to roll back to previous percentage at any point

6. FULL rollout (flag ON for all users)
   └── Monitor for 1 week
   └── Clean up feature flag
```

### Rollout Decision Thresholds

Use these thresholds to decide whether to advance, hold, or roll back at each stage:

| Metric | Advance (green) | Hold and investigate (yellow) | Roll back (red) |
|--------|-----------------|-------------------------------|-----------------|
| Error rate | Within 10% of baseline | 10-100% above baseline | >2x baseline |
| P95 latency | Within 20% of baseline | 20-50% above baseline | >50% above baseline |
| Client JS errors | No new error types | New errors at <0.1% of sessions | New errors at >0.1% of sessions |
| Business metrics | Neutral or positive | Decline <5% (may be noise) | Decline >5% |

### When to Roll Back

Roll back immediately if:
- Error rate increases by more than 2x baseline
- P95 latency increases by more than 50%
- User-reported issues spike
- Data integrity issues detected
- Security vulnerability discovered

## Monitoring and Observability

### What to Monitor

```
Application metrics:
├── Error rate (total and by endpoint)
├── Response time (p50, p95, p99)
├── Request volume
├── Active users
└── Key business metrics (conversion, engagement)

Infrastructure metrics:
├── CPU and memory utilization
├── Database connection pool usage
├── Disk space
├── Network latency
└── Queue depth (if applicable)

Client metrics:
├── Core Web Vitals (LCP, INP, CLS)
├── JavaScript errors
├── API error rates from client perspective
└── Page load time
```

### Error Reporting

```typescript
// Set up error boundary with reporting
class ErrorBoundary extends React.Component {
  componentDidCatch(error: Error, info: React.ErrorInfo) {
    // Report to error tracking service
    reportError(error, {
      componentStack: info.componentStack,
      userId: getCurrentUser()?.id,
      page: window.location.pathname,
    });
  }

  render() {
    if (this.state.hasError) {
      return <ErrorFallback onRetry={() => this.setState({ hasError: false })} />;
    }
    return this.props.children;
  }
}

// Server-side error reporting
app.use((err: Error, req: Request, res: Response, next: NextFunction) => {
  reportError(err, {
    method: req.method,
    url: req.url,
    userId: req.user?.id,
  });

  // Don't expose internals to users
  res.status(500).json({
    error: { code: 'INTERNAL_ERROR', message: 'Something went wrong' },
  });
});
```

### Post-Launch Verification

In the first hour after launch:

```
1. Check health endpoint returns 200
2. Check error monitoring dashboard (no new error types)
3. Check latency dashboard (no regression)
4. Test the critical user flow manually
5. Verify logs are flowing and readable
6. Confirm rollback mechanism works (dry run if possible)
```

## Rollback Strategy

Every deployment needs a rollback plan before it happens:

```markdown
## Rollback Plan for [Feature/Release]

### Trigger Conditions
- Error rate > 2x baseline
- P95 latency > [X]ms
- User reports of [specific issue]

### Rollback Steps
1. Disable feature flag (if applicable)
   OR
1. Deploy previous version: `git revert <commit> && git push`
2. Verify rollback: health check, error monitoring
3. Communicate: notify team of rollback

### Database Considerations
- Migration [X] has a rollback: `npx prisma migrate rollback`
- Data inserted by new feature: [preserved / cleaned up]

### Time to Rollback
- Feature flag: < 1 minute
- Redeploy previous version: < 5 minutes
- Database rollback: < 15 minutes
```
## See Also

- For security pre-launch checks, see `references/security-checklist.md`
- For performance pre-launch checklist, see `references/performance-checklist.md`
- For accessibility verification before launch, see `references/accessibility-checklist.md`
- [[web3-backend-reviewer]] — the review lens behind the Web3 readiness checklist (§4 operational maturity, §9 error handling / fallbacks)
- [[wallet-security-auditor]] — key rotation, KMS, signing-path pre-launch review
- [[category-wallet-custody]] — custody/operational runbooks (rotation, ceremony, incident response)
- [[category-rpc-and-indexer]] — RPC failover + indexer fallback design
- [[ci-cd-and-automation]] — the Go/TS/Foundry pipeline that enforces these gates pre-deploy

## Common Rationalizations

| Rationalization | Reality |
|---|---|
| "It works in staging, it'll work in production" | Production has different data, traffic patterns, and edge cases. Monitor after deploy. |
| "We don't need feature flags for this" | Every feature benefits from a kill switch. Even "simple" changes can break things. |
| "Monitoring is overhead" | Not having monitoring means you discover problems from user complaints instead of dashboards. |
| "We'll add monitoring later" | Add it before launch. You can't debug what you can't see. |
| "Rolling back is admitting failure" | Rolling back is responsible engineering. Shipping a broken feature is the failure. |

## Red Flags

- Deploying without a rollback plan
- No monitoring or error reporting in production
- Big-bang releases (everything at once, no staging)
- Feature flags with no expiration or owner
- No one monitoring the deploy for the first hour
- Production environment configuration done by memory, not code
- "It's Friday afternoon, let's ship it"

## Verification

Before deploying:

- [ ] Pre-launch checklist completed (all sections green)
- [ ] Feature flag configured (if applicable)
- [ ] Rollback plan documented
- [ ] Monitoring dashboards set up
- [ ] Team notified of deployment

After deploying:

- [ ] Health check returns 200
- [ ] Error rate is normal
- [ ] Latency is normal
- [ ] Critical user flow works
- [ ] Logs are flowing
- [ ] Rollback tested or verified ready
