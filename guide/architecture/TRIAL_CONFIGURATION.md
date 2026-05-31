---
title: Trial Configuration Architecture
---

# Trial Configuration Architecture

## Decision: Product Metadata over Price Field

**Date:** 2026-01-24

**Status:** Implemented

## Context

Stripe subscriptions support trial periods, but the configuration location is not obvious:

1. **Price.recurring.trial_period_days** - Appears in API responses but:
   - Not documented in official Stripe Price API docs
   - Often null or unavailable
   - Requires configuring each price separately (monthly, yearly, etc.)
   - Unclear if it's deprecated or just undocumented

2. **Product.metadata.trial_period_days** - Custom metadata field:
   - Documented, stable metadata API
   - One configuration for all prices of a product
   - Easy to manage via Stripe Dashboard
   - Explicit control over trial logic

3. **Checkout session trial_period_days** - Direct parameter:
   - Most explicit and reliable
   - Requires value at checkout time
   - Needs source of truth (we use Product metadata)

## Decision

Store trial configuration in **Product metadata** (`trial_period_days`), sync to database, pass to checkout sessions.

### Flow

```
Stripe Product Metadata (trial_period_days: "14")
          ↓
   pnpm db:seed:plans
          ↓
Database (billing_plan.trial_period_days = 14)
          ↓
createCheckoutSession()
          ↓
Stripe Checkout (subscription_data.trial_period_days = 14)
          ↓
Subscription created with 14-day trial
```

## Consequences

**Positive:**
- [YOURS] Centralized trial config (one metadata field per product)
- [YOURS] Reliable, documented API field
- [YOURS] Easy to update via Stripe Dashboard
- [YOURS] Database caching provides snapshot consistency
- [YOURS] Works even if Price field is null/unavailable

**Negative:**
- [NO] Requires manual metadata setup for each product
- [NO] Changes require re-sync (`pnpm db:seed:plans`)
- [NO] Not obvious to new developers (needs documentation)

## Alternatives Considered

### Option A: Read from Price.recurring.trial_period_days

**Rejected because:**
- Field is undocumented and unreliable
- Found to be null in testing
- Would require fallback logic anyway

### Option B: Hardcode in database

**Rejected because:**
- No single source of truth in Stripe
- Requires manual SQL updates
- Can't use Stripe Dashboard

### Option C: Environment variables

**Rejected because:**
- Can't vary by product
- Requires redeployment to change
- Not scalable for multiple plans

## Implementation

See commit: `fix(billing): read trial config from product metadata`

**Files changed:**
- `src/lib/billing/providers/stripe.ts` - Read from product.metadata
- `src/lib/billing/providers/base.ts` - Remove from ProviderPrice type
- `docs/BILLING.md` - Update setup instructions

## Testing

Verified with automated checks:
1. TypeScript compilation passes
2. Production build succeeds
3. Only expected files modified

Manual testing steps documented in `docs/BILLING.md`:
1. Set metadata `trial_period_days = 7`
2. Run `pnpm db:seed:plans`
3. Create subscription via checkout
4. Confirm subscription has 7-day trial
5. UI displays trial correctly

## References

- [Stripe Trial Subscriptions](https://docs.stripe.com/billing/subscriptions/trials)
- [Stripe Product Metadata](https://docs.stripe.com/api/metadata)
- [Stripe Checkout Sessions](https://docs.stripe.com/api/checkout/sessions/create#create_checkout_session-subscription_data-trial_period_days)
