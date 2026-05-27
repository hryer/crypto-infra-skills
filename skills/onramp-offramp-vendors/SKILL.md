---
name: onramp-offramp-vendors
description: Use when integrating fiat on/off-ramp providers for a crypto product. Covers vendor selection (MoonPay, Transak, Stripe, Ramp, Banxa, Onramper), aggregator vs direct integration, KYC re-use, settlement timing, FX spread economics, regional coverage, and failover. Use whenever the user mentions onramp, offramp, fiat deposit, fiat withdrawal, card-to-crypto, or buy crypto with USD/IDR/EUR.
---

# On/Off-Ramp Vendor Integration

## When to use

Trigger this skill when the user asks to:
- Choose between MoonPay, Transak, Ramp Network, Banxa, Stripe Crypto, Onramper, etc.
- Decide between direct vendor integration vs an aggregator
- Plan KYC reuse across ramp + main product
- Design fallback when primary ramp is unavailable in a region
- Optimize FX/spread economics on fiat conversion
- Add a new region or local payment method

## The landscape

### Direct vendor integration
You build a connection to one (or several) ramp providers:

| Vendor | Strengths | Weaknesses | Key regions |
|---|---|---|---|
| **MoonPay** | Largest, brand recognition, card + bank rails | High fees, slow new-region rollout | Global, US strong |
| **Transak** | Multi-chain, strong India/SE Asia, lower fees | Smaller US footprint | Asia, India, EMEA |
| **Ramp Network** | Clean UX, good widget, strong UK/EU | Smaller in US | UK, EU, broader global |
| **Banxa** | Asia-Pacific strength, multiple payment methods | Less polished SDK | APAC, AU |
| **Stripe Crypto** | Stripe ecosystem, modern API | US-only originally, limited assets, restrictive | US, expanding |
| **Mercuryo** | EU strong, card rails | Smaller global reach | EU |
| **Sardine** | Fraud + ramp combined | Newer | US |
| **Coinbase Onramp** | Coinbase user base, trust | Coinbase-locked-in users | Global where Coinbase operates |

### Aggregators
A single integration that routes to multiple underlying vendors:

| Aggregator | Notes |
|---|---|
| **Onramper** | Most widely used aggregator, 20+ providers |
| **Meld** | Newer, growing partner list |
| **Alchemy Pay** | Aggregator + own rails |

**Aggregator vs direct:**
- **Aggregator pros:** one integration covers many regions/methods; auto-failover; faster to launch
- **Aggregator cons:** thinner margins (aggregator takes a cut); less control over UX; harder to do KYC re-use
- **Direct pros:** better unit economics; KYC re-use possible; more control
- **Direct cons:** N integrations to maintain; you manage failover

**Recommendation:** start with aggregator for fast launch. Move highest-volume corridors to direct integration once you've proven demand.

## Process

### 1. Map your geography to regulatory reality

Not every vendor operates in every country. Indonesia is restricted on most US-based providers. Brazil has specific local rails (PIX). India has UPI-specific flows. Map your target users to vendors that can actually serve them.

**Example matrix for SE Asia:**

| Region | Best primary | Best backup | Avoid |
|---|---|---|---|
| Indonesia | Banxa (with local rails) or Transak | Onramper aggregator | MoonPay (limited IDR support) |
| Vietnam | Transak | Banxa | most US-based |
| Philippines | Coinbase Onramp, Transak | Banxa | — |
| Singapore | Most work (regulated market) | — | — |

### 2. Plan KYC handoff

Most ramp providers require their own KYC. If your product has already KYCed the user, you have three options:

**Option A: Re-KYC** (default for most vendors)
- User does KYC at your product + KYC at the ramp
- Bad UX, two checks
- But: simplest to integrate

**Option B: Pass-through KYC**
- Some vendors accept "KYC-shared" agreements where your KYC counts
- Requires legal agreement + tech integration (passing identity proofs)
- Better UX, lower drop-off
- MoonPay, Transak, Ramp have variants of this

**Option C: White-label / API-only ramp**
- The ramp is fully embedded; user never sees the vendor brand
- KYC happens once in your flow
- Requires premium tier with the vendor
- Best UX, costs the most

**Decision rule:** start with Option A (re-KYC). Move to B or C only when drop-off data justifies the investment.

### 3. Settlement timing model

The vendor takes fiat → delivers crypto. Timing varies wildly:

| Payment method | Typical settlement | Risk |
|---|---|---|
| Card (Visa/MC) | Crypto delivered within minutes; reversible for ~6 months via chargeback | High vendor risk |
| Bank wire (SEPA, ACH) | Hours to 1-2 days | Lower risk |
| Local rails (PIX, UPI, IDEAL) | Minutes | Medium risk |
| Apple/Google Pay | Minutes | Medium risk |

**Vendor risk:** card-based onramps eat chargeback risk. They charge more (5-7% all-in) to compensate. Bank-based is cheaper (~1-2%) but slower.

### 4. Where the money goes

The user pays the vendor in fiat. The vendor delivers crypto. There are two flow models:

**Direct-to-user wallet:**
- Vendor sends crypto to the user's wallet address you provide
- Cleanest, no custody risk to you
- Standard for non-custodial products

**Vendor-to-your-treasury:**
- Vendor sends crypto to your operational wallet
- You credit the user in your internal ledger
- Used for custodial products

Both are valid. Mixing them in the same product creates ledger complexity.

### 5. Off-ramp specifics

Off-ramp (crypto → fiat) is harder than on-ramp:
- Smaller universe of providers
- Country-specific banking requirements
- Tighter AML scrutiny
- KYC re-verification often required even if user was already KYCed at onramp

**Players:** MoonPay, Ramp Network, Transak, Coinbase, Banxa, Mercuryo all have offramp. Volume is heavily skewed to fewer vendors.

**Settlement times:**
- Card payouts (debit cards): minutes-hours (where supported)
- Bank wire: 1-3 business days
- Local rails (PIX, SEPA Instant): minutes

### 6. FX and fee economics

Vendor pricing is opaque on purpose. Typical all-in fee structures:

| Method | All-in user fee | Vendor share | Your share (if reseller) |
|---|---|---|---|
| Card on-ramp | 4-7% | 3-5% | 1-2% (if any) |
| Bank on-ramp | 1-2% | 0.5-1% | 0.5-1% |
| Off-ramp | 2-4% | 1.5-3% | 0.5-1% |

Your slice depends on volume tier negotiated with the vendor.

**Crypto-FX spread:** the displayed crypto price vs market mid is where some vendors quietly take an additional 50-150 bps. Audit this with synthetic test transactions.

### 7. Failover

Single-vendor dependency is dangerous. A vendor outage means no onramps for hours.

**Failover patterns:**
- **Health checks:** ping vendor's status API every minute. Pre-empt routing on red status.
- **Region-aware routing:** if primary fails for region X, try secondary for region X (not global).
- **Hot/warm vendors:** primary gets 90%+ of volume. Secondary gets 5-10% to keep the integration warm and tested.
- **Aggregator as backup:** even with direct integrations, keep an aggregator as third option.

### 8. Compliance hooks

Required regardless of vendor:
- Sanctions screening on the recipient wallet address (not just user identity)
- Transaction monitoring for layering, structuring
- Travel rule data for tx > threshold (depends on jurisdiction)
- SAR (Suspicious Activity Report) filing process

The vendor does much of this for you, but you remain accountable. Don't assume "the vendor handles compliance" — you handle compliance for your users.

## Rationalizations to reject

| Excuse | Reality |
|---|---|
| "MoonPay is the standard, just use them" | Region coverage matters more than brand. MoonPay struggles in Indonesia. Pick by region. |
| "We don't need failover, vendors are reliable" | Every major ramp has had multi-hour outages. Plan for it. |
| "Aggregators are just middlemen, cut them out" | The right aggregator covers 10x more regions than you can integrate directly in a year. |
| "Off-ramp can wait until we have users wanting to cash out" | Adding off-ramp requires new vendor contracts, new KYC, new banking. 3-6 month lead time. Plan early. |
| "The vendor will handle all compliance" | You're the regulated entity (or operating in their jurisdiction). Their compliance is necessary but not sufficient. |

## Verification

Before going live with a ramp:
- [ ] Test on-ramp in every supported region with a real card and a real bank
- [ ] Verify wallet address handoff (some chains use different address formats — vendor must support)
- [ ] Test failure modes (failed payment, KYC rejection, fraud decline)
- [ ] Document the user-facing fee disclosure
- [ ] Sanctions screening verified for both user identity and destination address
- [ ] Failover tested by simulating primary vendor 500
- [ ] Off-ramp tested for at least one region
- [ ] Reconciliation: vendor settlement reports match your internal records daily

## Competitive notes for product positioning

If competing with Crypto.com Pay, Bybit, Coinbase, Binance Pay, etc.:
- **Brand-name vendors win on trust** in some regions, lose on price
- **Local rails** beat international cards almost everywhere outside US (PIX in Brazil, UPI in India, IDEAL in NL, etc.)
- **Aggregator-powered products** can claim "20+ payment methods" without 20 integrations
- **Limit gaming:** higher tx limits than competitors is a real moat in markets with high inflation or capital control
