---
title: "Billing Domain"
---

# Billing Domain

This document describes the billing domain architecture in JumpSaaS. The billing system is **provider-agnostic** by design, with Stripe as the first payment provider implementation.

## Architecture Overview

```
┌─────────────────────────────────────────────────────────┐
│                    Application Layer                     │
│  (API Routes, Server Actions, UI Components)             │
└─────────────────┬───────────────────────────────────────┘
                  │
┌─────────────────▼───────────────────────────────────────┐
│              Billing Domain Services                     │
│  (Provider-agnostic business logic)                      │
│                                                           │
│  • Customer Service      • Subscription Service          │
│  • Payment Method        • Billing History               │
│  • Plan Configuration                                    │
└─────────────────┬───────────────────────────────────────┘
                  │
        ┌─────────┴─────────┬─────────────┐
        │                   │             │
┌───────▼────────┐  ┌──────▼──────┐  ┌──▼──────────┐
│ Stripe Provider│  │   Paddle    │  │  Other...   │
│  (Implemented) │  │  (Future)   │  │  (Future)   │
└────────────────┘  └─────────────┘  └─────────────┘
```

### Key Principles

1. **Provider Abstraction**: Core billing logic is independent of payment providers
2. **Single Source of Truth**: Database is the source of truth, synced from providers
3. **Service Layer Pattern**: All data transformations happen in service functions
4. **Type Safety**: Strong TypeScript types for domain models

## Core Concepts

### 1. Plans

Plans define subscription tiers (Free, Pro, Premium). They are:
- **Provider-agnostic**: Stored once in the database
- **Stripe-Synced**: Automatically fetched from Stripe API via metadata
- **Zero Configuration**: No environment variables needed for product/price IDs
- **Extensible**: Support multiple providers via `providerData` JSONB field

**Stripe Product Metadata:**
```json
{
  "plan_slug": "pro",
  "tier": "pro",
  "features": "[\"Priority support\", \"API access\", \"Advanced analytics\"]",
  "sort_order": "1",
  "is_public": "true"
}
```

**How It Works:**
1. Create products in Stripe Dashboard with metadata
2. Run `pnpm db:seed:plans` to sync from Stripe API
3. All plan data (name, price, features) comes from Stripe

**Database Schema:**
```typescript
// billing_plan table
{
  id: string;
  slug: string;              // "free", "pro", "premium" (from Stripe metadata)
  name: string;              // "Pro" (from Stripe product name)
  tier: PlanTier;            // "pro" (from Stripe metadata)
  price: decimal;            // 29.00 (from Stripe price)
  currency: string;          // "usd" (from Stripe price)
  interval: PlanInterval;    // "month" (from Stripe price)
  creditsPerMonth: number;   // Not used (set to 0)
  features: string[];        // Parsed from Stripe metadata JSON
  providerData: JSONB;       // { stripe: { priceId, productId } }
}
```

## Stripe-Based Plan Seeding

Plans are seeded from Stripe API as the **single source of truth**:

### Setup Process

1. **Create Products in Stripe Dashboard**
   - Go to Products → Create product
   - Add monthly recurring price (e.g., $29/month)
   - **Add metadata** (critical):
     - `plan_slug`: `"pro"` (our internal identifier)
     - `tier`: `"pro"` (plan tier)
     - `features`: `'["Priority support", "API access"]'` (JSON array)
     - `sort_order`: `"1"` (display order)
     - `is_public`: `"true"` (show on pricing page)

2. **Run Seed Script**
   ```bash
   pnpm db:seed:plans
   ```

   The script automatically:
   - Fetches all active products from Stripe
   - Parses metadata for plan configuration
   - Syncs pricing data from Stripe prices
   - Updates database with complete plan info

3. **Verify Database**
   - Check `billing_plan` table has plans synced
   - Verify `providerData` contains Stripe IDs
   - Confirm prices match Stripe Dashboard

### Benefits

- **Zero Configuration**: No product/price IDs in environment variables
- **Single Source of Truth**: Stripe Dashboard is authoritative
- **Metadata-Based**: All plan data stored in Stripe product metadata
- **Type-Safe**: Provider interface enforces consistent data structure
- **Extensible**: Easy to add Paddle, LemonSqueezy, etc.

### Metadata Fields Reference

| Field | Type | Required | Example | Description |
|-------|------|----------|---------|-------------|
| `plan_slug` | string | Yes | `"pro"` | Unique plan identifier |
| `tier` | string | Yes | `"pro"` | Plan tier (free, starter, pro, premium, enterprise) |
| `features` | JSON array | Yes | `'["Feature 1", "Feature 2"]'` | Plan features list |
| `sort_order` | string (number) | No | `"1"` | Display order (default: 0) |
| `is_public` | string (boolean) | No | `"true"` | Show on pricing page (default: true) |
| `trial_period_days` | string (number) | No | `"14"` | Free trial duration in days (default: 0) |

## Trial Configuration

JumpSaaS supports free trials for paid plans. Trials are configured via Stripe Product metadata and synced to your database.

### Setting Up Trials

Trials are configured via Stripe Product metadata and synced to your database:

**1. Add trial metadata to Stripe Product:**

Go to [Stripe Dashboard → Products](https://dashboard.stripe.com/products):

1. Click on your product (e.g., "Pro Plan")
2. Scroll to "Metadata" section
3. Click "Add metadata"
4. Key: `trial_period_days`
5. Value: `14` (for 14-day trial)
6. Click "Save product"

**2. Sync to database:**

```bash
pnpm db:seed:plans
```

This reads `product.metadata.trial_period_days` and stores it in your database `billing_plan.trial_period_days` field.

**3. Trial applies automatically at checkout:**

When users subscribe, the checkout service reads `trialPeriodDays` from the database plan and passes it to Stripe:

```typescript
// src/services/billing/checkout/create.service.ts (line 55-58)
subscription_data: {
  trial_period_days: planConfig.trialPeriodDays > 0
    ? planConfig.trialPeriodDays
    : undefined,
}
```

**Why Product metadata?**

- **Centralized**: One place to configure trials for all prices of a product
- **Reliable**: Metadata is a documented, stable API field
- **Dashboard-friendly**: Easy to update via Stripe Dashboard without code changes
- **Version control**: Changes tracked via database migrations, not hidden in Stripe Price objects

**Alternative (NOT recommended):**

Stripe Prices have an undocumented `recurring.trial_period_days` field, but it's:
- Not in official API documentation
- Often null/unavailable
- Per-price instead of per-product
- Requires configuring each price separately

We use Product metadata for reliability and simplicity.

### How Trials Work

```
┌─────────────────────────────────────────────────────────────────┐
│                        Trial Flow                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  1. Configure in Stripe Dashboard                               │
│     → Add product.metadata.trial_period_days = "14"             │
│                                                                  │
│  2. Run pnpm db:seed:plans                                      │
│     → Reads product metadata                                    │
│     → Stores in billing_plan.trial_period_days                  │
│                                                                  │
│  3. User clicks "Subscribe" on pricing page                     │
│     → createCheckoutSession() reads planConfig.trialPeriodDays  │
│     → Passes subscription_data.trial_period_days to Stripe      │
│                                                                  │
│  4. Stripe creates subscription with trial                      │
│     → status: "trialing"                                        │
│     → trial_start and trial_end timestamps set                  │
│                                                                  │
│  5. Webhook syncs subscription to database                      │
│     → trialStart, trialEnd populated from Stripe                │
│     → status = "trialing"                                       │
│                                                                  │
│  6. During trial period                                          │
│     → User has full access to plan features                     │
│     → No charges made                                           │
│     → UI shows trial countdown banner                           │
│                                                                  │
│  7. Trial ends                                                   │
│     → Stripe automatically charges payment method               │
│     → Webhook updates status to "active"                        │
│     → Normal billing cycle begins                               │
│                                                                  │
│  8. If payment fails at trial end                               │
│     → Status becomes "past_due" or "incomplete"                 │
│     → User loses access based on grace period config            │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Trial Display

**Plan Cards (Pricing Page):**
- Shows "{days}-day free trial" below price
- Only shown for paid plans with `trialPeriodDays > 0`

**Billing Page (During Trial):**
- Blue alert banner: "Free trial active"
- Countdown: "X days remaining"
- Trial end date
- Next billing date (when trial ends)

**Status Badge:**
- Shows "Trial" in blue with special styling
- Helps users identify trial subscriptions quickly

### Testing Trials

**1. Configure trial in Stripe Dashboard:**

1. Go to test mode Products
2. Select a product (or create test product)
3. Add metadata: `trial_period_days` = `3` (short for testing)
4. Save product

**2. Sync to database:**

```bash
pnpm db:seed:plans
```

Verify in database:
```sql
SELECT name, trial_period_days FROM billing_plan WHERE slug = 'test-plan';
```

Expected: `trial_period_days = 3`

**3. Subscribe using test card:**

Test card: `4242 4242 4242 4242`

**4. Verify in `/settings/billing`:**
- [YOURS] Trial banner appears with blue background
- [YOURS] Countdown shows "3 days remaining"
- [YOURS] Status badge shows "Trial" in blue
- [YOURS] Trial end date displayed
- [YOURS] Next billing date shown below trial end

**5. Verify in Stripe Dashboard:**

Check the subscription has:
- Status: `trialing`
- Trial end date set to 3 days from now

**Fast-forward trial (test mode only):**

Use [Stripe Test Clocks](https://stripe.com/docs/billing/testing/test-clocks) to simulate trial expiration.

### Trial Best Practices

**Trial Duration:**
- 14 days is standard for most SaaS products
- 7 days for simpler products
- 30 days for complex enterprise software

**Payment Method Collection:**
- This codebase requires payment method at checkout (default Stripe Checkout behavior)
- Reduces fraud and improves trial-to-paid conversion
- User is auto-charged when trial ends (no manual upgrade needed)

**Trial End Behavior:**
- Per Stripe docs, subscriptions auto-charge at trial end if payment method exists
- If payment fails: subscription becomes `past_due` or `incomplete`
- Consider implementing Stripe's `trial_will_end` webhook (fires 3 days before) for reminder emails

**User Experience:**
- Show clear trial countdown ([YOURS] implemented in UI)
- Display trial end date prominently ([YOURS] implemented)
- Provide full feature access during trial
- Add upgrade CTAs during last 3 days of trial

**Compliance:**
- Follow card network requirements for trial messaging
- Configure compliance settings in Stripe Dashboard → Settings → Billing

### Migration: Adding Trials to Existing Plans

To add trials to existing plans:

**1. Add metadata to Stripe Product:**

1. Go to Stripe Dashboard → Products
2. Click your product
3. Add metadata: `trial_period_days` = `14`
4. Save

**2. Sync changes:**

```bash
pnpm db:seed:plans
```

**3. Verify:**

```sql
SELECT name, trial_period_days FROM billing_plan;
```

**4. Deploy:**

New subscriptions will automatically include the 14-day trial. Existing active subscriptions are NOT affected.

**Important Notes:**
- Trial is read from product metadata at sync time
- Applied at checkout session creation (NOT from Stripe Price)
- Only new subscriptions get the trial
- To apply trials to existing users, cancel and recreate their subscriptions

### 2. Customers

A customer links a user to a payment provider. One user can have multiple customer records (one per provider).

**Service Functions:**
```typescript
// src/lib/billing/service/customer.service.ts
await getCustomerByUserId(userId);                    // Get customer
await createStripeCustomer(userId, email);            // Create new customer
```

**Database Schema:**
```typescript
// billing_customer table
{
  id: string;
  userId: string;              // References user table
  provider: PaymentProvider;   // "stripe", "paddle", etc.
  providerCustomerId: string;  // "cus_xxx" from provider
  email: string;
  metadata: JSONB;
}
```

### 3. Subscriptions

Subscriptions track active plans for users.

**Service Functions:**
```typescript
// src/lib/billing/service/subscription.service.ts
await getActiveSubscription(userId);                  // Get active subscription
await cancelSubscription(subscriptionId);             // Cancel at period end
await resumeSubscription(subscriptionId);             // Resume canceled subscription
```

**Database Schema:**
```typescript
// billing_subscription table
{
  id: string;
  userId: string;
  customerId: string;
  planId: string;
  provider: PaymentProvider;
  providerSubscriptionId: string;  // "sub_xxx" from provider
  status: SubscriptionStatus;      // "active", "trialing", "canceled", etc.
  currentPeriodStart: Date;
  currentPeriodEnd: Date;
  cancelAt: Date | null;
  cancelAtPeriodEnd: boolean;
  trialStart: Date | null;
  trialEnd: Date | null;
}
```

### 4. Payment Methods

Payment methods store card/bank information for customers.

**Service Functions:**
```typescript
// src/lib/billing/service/payment-method.service.ts
await getDefaultPaymentMethod(userId);               // Get default payment method
```

**Database Schema:**
```typescript
// billing_payment_method table
{
  id: string;
  userId: string;
  customerId: string;
  provider: PaymentProvider;
  providerPaymentMethodId: string;  // "pm_xxx" from provider
  type: string;                     // "card", "bank_account", etc.
  cardBrand: string;                // "visa", "mastercard"
  cardLast4: string;                // "4242"
  cardExpMonth: number;             // 12
  cardExpYear: number;              // 2025
  isDefault: boolean;
}
```

### 5. Billing History

Combines invoices and payments into a unified history.

**Service Functions:**
```typescript
// src/services/billing/invoice.service.ts
const invoicesResult = await getInvoices(userId, page, pageSize);
```

**Response Type:**
```typescript
type BillingHistoryItem = {
  id: string;
  type: "invoice" | "payment";
  amount: number;        // Converted from cents (1000 → 10.00)
  currency: string;
  status: string;
  date: Date;
  isPaid?: boolean;      // Computed for invoices
  isOverdue?: boolean;   // Computed for invoices
};
```

## Service Layer Pattern

All billing services follow this pattern:

### [YOURS] DO in Service Layer

1. **Transform data from cents to decimal**
   ```typescript
   amount: invoice.total / 100  // 2900 → 29.00
   ```

2. **Add computed business fields**
   ```typescript
   isPaid: invoice.status === "paid"
   isOverdue: dueDate < now && status === "open"
   ```

3. **Merge and normalize data**
   ```typescript
   const items = [...invoices.map(transform), ...payments.map(transform)]
     .sort((a, b) => b.date - a.date);
   ```

4. **Export clear TypeScript types**
   ```typescript
   export type BillingHistoryItem = { ... }
   ```

### [NO] DON'T in Service Layer

- Currency symbols (`$`, `€`) - belongs in UI
- Date formatting (`"Jan 15, 2024"`) - belongs in UI with i18n
- Translations - use next-intl in components
- UI-specific data shaping

## Adding a New Payment Provider

To add Paddle, Lemon Squeezy, or other providers:

### 1. Add Provider Enum Value
```typescript
// src/db/schema/billing.ts
export const paymentProviderEnum = pgEnum("payment_provider", [
  "stripe",
  "paddle",  // ← Add here
  // ...
]);
```

### 2. Update Plan Provider Data Type
```typescript
// src/db/schema/billing.ts
export type PlanProviderData = {
  stripe?: { priceId: string; productId: string };
  paddle?: { planId: string; productId?: string };  // ← Add here
};
```

### 3. Create Provider Directory
```bash
src/lib/paddle/
├── config.ts          # Initialize Paddle client
└── sync.ts            # syncPaddleDataToDB(customerId)
```

### 4. Implement Sync Function
```typescript
// src/lib/paddle/sync.ts
export async function syncPaddleDataToDB(paddleCustomerId: string) {
  // 1. Fetch subscription from Paddle API
  // 2. Find/create customer in database
  // 3. Upsert subscription record
  // 4. Upsert payment method
  // 5. Return normalized data
}
```

### 5. Update Service Functions
```typescript
// src/lib/billing/service/customer.service.ts
export async function createPaddleCustomer(userId: string, email: string) {
  // Create customer in Paddle
  // Save to database with provider="paddle"
}
```

### 6. Create Webhook Handler
```typescript
// src/app/api/webhooks/paddle/route.ts
export async function POST(request: Request) {
  // Verify webhook signature
  // Call syncPaddleDataToDB(customerId)
}
```

## Credit System Integration

Subscriptions automatically sync credits to users:

```typescript
// In syncStripeDataToDB or other provider sync functions
if (subscription.status === "active") {
  await updateUserCredits(userId, plan.creditsPerMonth);
}
```

## File Structure

```
src/
├── services/
│   └── billing/
│       ├── customer.service.ts         # Customer operations
│       ├── subscription.service.ts     # Subscription operations
│       ├── payment-method.service.ts   # Payment method operations
│       ├── invoice.service.ts          # Invoice operations
│       ├── plan.service.ts             # Plan operations
│       └── index.ts                    # Barrel exports
│
├── lib/
│   ├── billing/
│   │   ├── providers/
│   │   │   ├── base.ts                     # Payment provider interfaces
│   │   │   ├── stripe.ts                   # Stripe provider implementation
│   │   │   └── index.ts                    # Provider registry
│   │   └── id.ts                           # ID generation
│   │
│   └── stripe/                             # Stripe-specific implementation
│       ├── config.ts                       # Stripe client initialization
│       └── sync.ts                         # syncStripeDataToDB()
│
├── scripts/
│   └── seed-plans.ts                       # Seed plans from Stripe API
│
├── app/
│   └── api/
│       ├── billing/
│       │   ├── plans/route.ts              # Get public plans
│       │   ├── checkout/route.ts           # Create checkout session
│       │   ├── checkout/success/route.ts   # Handle success redirect
│       │   ├── subscription/
│       │   │   ├── cancel/route.ts         # Cancel subscription
│       │   │   └── resume/route.ts         # Resume subscription
│       │   └── data/route.ts               # Get billing data
│       │
│       └── webhooks/
│           └── stripe/route.ts             # Stripe webhook handler
│
└── components/
    └── billing/
        ├── provider.tsx                              # BillingProvider context
        ├── helpers.ts                                # Formatting utilities
        ├── index.ts                                  # Exports (BillingProvider, hooks)
        └── ui/
            ├── billing-subscription/
            │   ├── index.ts                          # Public exports
            │   ├── billing-subscription-card.tsx     # Main subscription card
            │   ├── subscription-plan-details.tsx     # Plan name, status, and pricing
            │   ├── subscription-billing-info.tsx     # Next billing and cancellation info
            │   ├── subscription-actions.tsx          # Cancel/resume buttons
            │   └── subscription-status-badge.tsx     # Status badge component
            ├── billing-payment-method/
            │   ├── index.ts                          # Public exports
            │   ├── billing-payment-method-card.tsx   # Main payment method card
            │   ├── payment-method-display.tsx        # Payment method display
            │   └── empty-payment-method.tsx          # Empty state component
            └── billing-history/
                ├── index.ts                          # Public exports
                ├── billing-history-table.tsx         # Main billing history table
                ├── invoice-row.tsx                   # Individual invoice row
                ├── invoice-status-badge.tsx          # Status badge component
                └── empty-billing-history.tsx         # Empty state component
```

## Client-Side Billing Context

The `BillingProvider` provides global access to billing data with **lazy loading** for optimal performance.

### Setup

The provider is included in the root layout:

```tsx
// src/app/[locale]/layout.tsx
import { BillingProvider } from "@/components/billing";

<BillingProvider>
  {children}
</BillingProvider>
```

### Available Hooks

#### `useBilling()` - Base Context
Access raw billing context (no automatic fetching):

```tsx
import { useBilling } from "@/components/billing";

const { 
  userId,           // string | null - from auth session
  plans,            // PlanInfo[] - empty until fetched
  currentPlan,      // string | null - e.g., "pro"
  subscription,     // SubscriptionInfo | null
  paymentMethod,    // PaymentMethodInfo | null
  fetchPlans,       // () => Promise<PlanInfo[]>
  fetchBilling,     // () => Promise<void>
} = useBilling();
```

#### `usePlans()` - Plans with Auto-Fetch
Fetches plans automatically when component mounts:

```tsx
import { usePlans } from "@/components/billing";

const { plans, isLoading } = usePlans();
// Plans are fetched on first use, cached thereafter
```

#### `useSubscription()` - User Billing with Auto-Fetch
Fetches user's subscription data (only for authenticated users):

```tsx
import { useSubscription } from "@/components/billing";

const { 
  subscription,     // Full subscription details
  currentPlan,      // Plan slug (e.g., "pro")
  paymentMethod,    // Payment method info
  isLoading,        // Loading state
  isLoaded,         // Whether fetch has been attempted
  refetch,          // Force refresh
} = useSubscription();
```

### Lazy Loading Benefits

| Data | Fetched When | Benefits |
|------|--------------|----------|
| Plans | Component uses `usePlans()` | Anonymous users don't trigger API calls |
| User Billing | Component uses `useSubscription()` | Only authenticated users on billing-related pages |

### Example Usage

```tsx
// Pricing page - shows plans to everyone
function PricingPage() {
  const { plans, isLoading } = usePlans();
  const { currentPlan } = useSubscription(); // Only fetches if logged in
  
  return plans.map(plan => (
    <PricingCard 
      plan={plan} 
      isCurrentPlan={plan.slug === currentPlan} 
    />
  ));
}

// Settings page - only for authenticated users
function BillingSettings() {
  const { subscription, paymentMethod, isLoading } = useSubscription();
  // Billing data fetched automatically
}
```

## API Routes

### Plans

```typescript
// GET /api/billing/plans
// Returns public plans (used by BillingProvider)
// Response: { plans: PlanInfo[] }
```

### Checkout Flow

```typescript
// POST /api/billing/checkout
// Body: { planSlug: string }
// Creates checkout session and returns URL

// GET /api/billing/checkout/success?session_id=xxx
// Syncs data immediately after successful payment
```

### Subscription Management

```typescript
// POST /api/billing/subscription/cancel
// Body: { subscriptionId: string }
// Cancels subscription at period end

// POST /api/billing/subscription/resume
// Body: { subscriptionId: string }
// Resumes a canceled subscription
```

### Data Fetching

```typescript
// GET /api/billing/data
// Returns current subscription, payment method, and invoices
// Response: { subscription, paymentMethod, invoices, pagination }
```

## Testing

### Test Payment Flow

1. Start development server:
   ```bash
   pnpm dev
   ```

2. Use test card:
   - Card: `4242 4242 4242 4242`
   - Expiry: Any future date
   - CVC: Any 3 digits

3. Verify data synced to database

### Test Webhooks Locally

```bash
# Install Stripe CLI
brew install stripe/stripe-cli/stripe

# Forward webhooks to local server
stripe listen --forward-to localhost:3000/api/webhooks/stripe
```

### Testing Trial Emails

**Stripe fires `customer.subscription.trial_will_end` event 3 days before trial expires.**

To test locally:
```bash
stripe listen --forward-to localhost:3000/api/webhooks/stripe
stripe trigger customer.subscription.trial_will_end
```

This sends a trial expiring email to the test user.

## Plan Switching

Users with an active subscription can switch between plans (upgrade or downgrade). The system uses Stripe Billing Portal for a secure, hosted payment experience.

### How It Works

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         Plan Switch Flow                                 │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  1. User clicks "Upgrade" or "Switch Plan"                              │
│                    │                                                     │
│                    ▼                                                     │
│  2. App calls switchPlan() server action                                │
│                    │                                                     │
│                    ▼                                                     │
│  3. Server creates Stripe Billing Portal session                        │
│     with subscription_update_confirm flow                               │
│                    │                                                     │
│                    ▼                                                     │
│  4. User redirected to Stripe's hosted page                             │
│     - Sees proration breakdown                                          │
│     - Reviews charges/credits                                           │
│     - Confirms the change                                               │
│                    │                                                     │
│                    ▼                                                     │
│  5. Stripe processes payment (if upgrade) or issues credit              │
│                    │                                                     │
│                    ▼                                                     │
│  6. User redirected back to /settings/billing                           │
│                    │                                                     │
│                    ▼                                                     │
│  7. Webhook updates subscription in database                            │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### Implementation

**Server Action** (`src/components/plan/actions.ts`):

```typescript
export async function switchPlan(
  planSlug: string,
  interval: RecurringInterval = "month"
): Promise<{ url?: string; error?: string }> {
  // ... validation ...

  // Create Billing Portal session with subscription_update_confirm flow
  const portalSession = await stripe.billingPortal.sessions.create({
    customer: customer.stripeCustomerId,
    return_url: `${baseUrl}/${locale}/settings/billing`,
    flow_data: {
      type: "subscription_update_confirm",
      subscription_update_confirm: {
        subscription: currentSubscription.providerSubscriptionId,
        items: [
          {
            id: stripeSubscription.items.data[0].id,
            price: stripePriceId,
          },
        ],
      },
    },
  });

  return { url: portalSession.url };
}
```

### Required: Stripe Billing Portal Configuration

[CAUTION] **The Billing Portal must be configured to allow subscription updates.** Without this, you'll get:

```
Error: This subscription cannot be updated because the subscription
update feature in the portal configuration is disabled.
```

**Configuration Steps:**

1. Go to [Stripe Dashboard → Settings → Billing → Customer portal](https://dashboard.stripe.com/settings/billing/portal)

2. Under **Subscriptions**, enable **"Customers can switch plans"**

3. Configure **Products** - Add all your products/prices that customers can switch between

4. Configure **Proration behavior** (recommended settings):
   - **When customers change plans**: "Prorate charges and credits"
   - **Charge timing**: "Invoice prorations immediately at the time of the update"
   - **Downgrades**: "Update immediately" (or "At end of billing period" if preferred)

5. Save the configuration

### Proration Explained

Stripe calculates prorated charges/credits based on unused time:

**Example: Upgrading mid-cycle**
```
Current: Pro plan ($19/month), paid on Jan 1
Action:  Upgrade to Premium ($100/month) on Jan 15

Calculation:
- Unused Pro time: 16 days × ($19/30) = $10.13 credit
- Remaining Premium time: 16 days × ($100/30) = $53.33 charge
- Amount due: $53.33 - $10.13 = $43.20

Next month: Full $100 Premium charge
```

**Example: Downgrading mid-cycle**
```
Current: Premium plan ($100/month), paid on Jan 1
Action:  Downgrade to Pro ($19/month) on Jan 15

Calculation:
- Unused Premium time: 16 days × ($100/30) = $53.33 credit
- Remaining Pro time: 16 days × ($19/30) = $10.13 charge
- Credit issued: $53.33 - $10.13 = $43.20

Credit is applied to future invoices automatically.
```

### Customer Credits

When a customer downgrades, Stripe issues a credit:

| Concept | How It Works |
|---------|--------------|
| **Storage** | Credit stored as negative `customer.balance` |
| **Application** | Automatically applied to future invoices |
| **Visibility** | Shown on invoices in Stripe Dashboard |
| **User action** | None required - fully automatic |

**Credit flow example:**
```
Downgrade credit: $43.20

Month 1: $19 invoice - $43.20 credit = $0 due (remaining: $24.20)
Month 2: $19 invoice - $24.20 credit = $0 due (remaining: $5.20)
Month 3: $19 invoice - $5.20 credit  = $13.80 due (remaining: $0)
```

### Proration Options

Configure in Stripe Billing Portal settings:

| Option | Behavior |
|--------|----------|
| **No charges or credits** | New price applies at next billing cycle only |
| **Prorate charges and credits** | Calculate exact unused time (recommended) |
| **Charge or credit full difference** | Full price difference, regardless of timing |

### Error Handling

Common errors and solutions:

| Error | Cause | Solution |
|-------|-------|----------|
| `PLAN_SWITCH_FAILED` | Generic failure | Check server logs |
| `NO_ACTIVE_SUBSCRIPTION` | User has no subscription | Redirect to checkout instead |
| `SAME_PLAN` | Switching to current plan | Show "Current Plan" badge |
| `PLAN_NOT_AVAILABLE` | Price not found for interval | Check Stripe product setup |
| Portal disabled error | Billing Portal not configured | Configure portal in Stripe Dashboard |

### Testing Plan Switches

1. Create a subscription using test card `4242 4242 4242 4242`
2. Click upgrade/switch on a different plan
3. Confirm on Stripe Billing Portal
4. Verify:
   - Invoice created in Stripe Dashboard
   - Subscription updated in database
   - Webhook processed correctly

## Security Considerations

- [YOURS] Always verify webhook signatures
- [YOURS] Use HTTPS in production
- [YOURS] Authenticate users before checkout
- [YOURS] Store API keys in environment variables
- [YOURS] Never expose provider IDs to frontend
- [YOURS] Validate subscription ownership before operations

## Provider Documentation

- [Stripe Setup Guide](../integrations/stripe) - How to configure Stripe

## Next Steps

1. **Add more providers**: Paddle, Lemon Squeezy, PayPal
2. **Implement usage tracking**: Track credit consumption per feature
3. **Add feature gating**: Middleware to check plan access
4. **Customer portal**: Self-service subscription management
5. **Analytics**: Track MRR, churn, conversion rates
