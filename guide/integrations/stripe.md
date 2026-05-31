---
title: "Stripe Setup Guide"
---

# Stripe Setup Guide

Quick setup guide for configuring Stripe as a payment provider in JumpSaaS.

> **See also**: [Billing Domain Documentation](../reference/billing) for architecture overview

## Prerequisites

- Stripe account ([sign up here](https://dashboard.stripe.com/register))
- PostgreSQL database running
- Development environment configured

## Recommended Initialization Flow

Because this project uses a **Stripe → DB sync** strategy (products live in Stripe and are synced to our product database via webhooks), the simplest way to initialize Stripe from scratch is:

1. **Deploy the server** so the webhook endpoint URL exists (e.g. `https://yourdomain.com/api/webhooks/stripe`).
2. **Create the Stripe account** and copy the secret key into your environment (`STRIPE_SECRET_KEY`).
3. **Create the webhook endpoint** by running `pnpm stripe:webhook https://yourdomain.com/api/webhooks/stripe`. The script registers all 26 events and prints the `STRIPE_WEBHOOK_SECRET` — add it to your env.
4. **Push the product catalog** with `pnpm stripe:push`. This creates the products/prices in Stripe, which fires `product.*` and `price.*` webhooks, which our sync logic uses to populate the product DB automatically.

This avoids manually clicking 26 events in the Stripe Dashboard and avoids needing a separate "seed local DB" step — Stripe is the source of truth and the webhook does the rest.

## Step 1: Deploy the Server

The webhook endpoint URL must exist before Stripe can deliver events to it.

- **Production**: deploy the app so `https://yourdomain.com/api/webhooks/stripe` is reachable.
- **Local development**: run `pnpm dev` and expose it with a tunnel (e.g. Cloudflare Tunnel — see [cloudflare-tunnel.md](./cloudflare-tunnel)) or use the Stripe CLI forwarding flow described in [Local Webhooks](#local-webhooks-stripe-cli) below.

## Step 2: Get API Keys

1. Go to [Stripe Dashboard](https://dashboard.stripe.com/)
2. Ensure you're in **Test Mode** (toggle in top right)
3. Navigate to **Developers** > **API keys**
4. Copy your keys:
   - **Publishable key** (starts with `pk_test_`)
   - **Secret key** (starts with `sk_test_`)

Add to `.env`:

**Next.js:**
```bash
STRIPE_SECRET_KEY=sk_test_your_secret_key_here
NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY=pk_test_your_publishable_key_here
NEXT_PUBLIC_APP_URL=https://yourdomain.com
```

**TanStack (Vite):**
```bash
STRIPE_SECRET_KEY=sk_test_your_secret_key_here
VITE_STRIPE_PUBLISHABLE_KEY=pk_test_your_publishable_key_here
VITE_APP_URL=https://yourdomain.com
```

## Step 3: Create the Webhook Endpoint

Run the setup script — it registers all 26 events in one call:

```bash
pnpm stripe:webhook https://yourdomain.com/api/webhooks/stripe
# or, if NEXT_PUBLIC_APP_URL is set:
pnpm stripe:webhook
```

The script:
- Creates a webhook endpoint pointing at `/api/webhooks/stripe`
- Subscribes all events the handler supports (checkout, customer, subscription, invoice, payment intent, payment method, product, price)
- Prints the signing secret on first run
- Is idempotent — re-running against an existing endpoint with the same URL updates its events instead of creating a duplicate

Copy the printed signing secret into `.env`:
```bash
STRIPE_WEBHOOK_SECRET=whsec_your_webhook_secret_here
```

> The signing secret is only shown on **create**. If you re-run the script against an existing endpoint, retrieve the secret from the Stripe Dashboard.

See the [Webhook Events Reference](#webhook-events-reference) below for the full event list and what each one does.

## Step 4: Push the Product Catalog

Define your plans and products in `scripts/stripe/config/`, then push them to Stripe:

```bash
pnpm stripe:push
```

This creates/updates products and prices in Stripe. Because the webhook endpoint from Step 3 is already subscribed to `product.*` and `price.*` events, Stripe immediately calls back into your server and the sync logic populates the `billing_plan` / `billing_product` tables — **no separate DB seed step is required**.

The catalog config is the source of truth for what exists in Stripe. Edit the files under `scripts/stripe/config/` and re-run `pnpm stripe:push` whenever you want to change plans or products.

### Catalog metadata reference

Plans and products are differentiated and identified entirely by Stripe metadata. The script sets these for you, but you'll see them in the Stripe Dashboard:

**Plan products** (subscription tiers shown on the pricing page):
- `plan_slug` — REQUIRED, unique identifier (e.g. `pro`, `premium`)
- `tier` — Optional: `free`, `starter`, `pro`, `premium`, `enterprise`
- `sort_order` — Optional display order
- `is_public` — Optional, defaults to `true`

**Standalone products** (one-time purchase items shown below plan cards):
- `product_type: product` — REQUIRED, tells the sync this is a product, not a plan
- `product_slug` — REQUIRED, unique identifier
- `is_public: true` — REQUIRED to show on the pricing page
- `sort_order`, `category` — Optional

For products with **variants** (e.g. small/medium/large packs), each Stripe price gets a unique `lookup_key` used as the variant slug, and the `nickname` is shown as the variant chip label. Variants are sorted cheapest → most expensive on the pricing page.

**Lifetime pricing**: any one-time price attached to a plan product is automatically mapped to `interval: "lifetime"` and enables the Lifetime tab on `/pricing`.

### Manual edits in the Stripe Dashboard

You can also create or edit plans and products directly in the Stripe Dashboard if you prefer — as long as the metadata keys above are set, the webhook sync will pick up the changes the same way.

<a id="local-webhooks-stripe-cli"></a>
### Local Webhook Alternative (Stripe CLI)

For local development without a public URL, skip `pnpm stripe:webhook` and use the Stripe CLI to forward events instead:

```bash
brew install stripe/stripe-cli/stripe
stripe login
stripe listen --forward-to localhost:3000/api/webhooks/stripe
```

The CLI prints a `whsec_...` secret — add it as `STRIPE_WEBHOOK_SECRET` in `.env`. Keep `stripe listen` running while developing.

## Webhook Events Reference

The application handles 26 different Stripe webhook events, organized by category:

### Checkout Events
- **`checkout.session.completed`**: Syncs customer data after successful checkout. Ensures user sees their subscription immediately.

### Customer Events
- **`customer.created`**: No action taken (customer created during checkout).
- **`customer.updated`**: Updates customer email and metadata in database.
- **`customer.deleted`**: Removes customer record from database.

### Subscription Events
- **`customer.subscription.created`**: Syncs new subscription to database.
- **`customer.subscription.updated`**: Updates subscription status, period dates, cancellation info.
- **`customer.subscription.deleted`**: Marks subscription as canceled in database.
- **`customer.subscription.paused`**: Updates subscription status to paused.
- **`customer.subscription.resumed`**: Updates subscription status to active.

### Invoice Events
- **`invoice.paid`**: Syncs invoice and updates subscription (confirms payment received).
- **`invoice.payment_failed`**: Records failed payment and syncs subscription status.
- **`invoice.payment_action_required`**: Records that user action is needed (e.g., 3D Secure).
- **`invoice.created`**: Syncs invoice when created (before finalization).
- **`invoice.updated`**: Updates invoice details in database.
- **`invoice.finalized`**: Syncs finalized invoice ready for payment.

### Payment Intent Events
- **`payment_intent.succeeded`**: Syncs customer data after successful one-time payment.
- **`payment_intent.payment_failed`**: Syncs customer data to reflect failed payment.
- **`payment_intent.canceled`**: Syncs customer data when payment is canceled.

### Payment Method Events
- **`payment_method.attached`**: Adds payment method to database when attached to customer.
- **`payment_method.updated`**: Updates payment method details (e.g., expiration date).
- **`payment_method.detached`**: Removes payment method from database when detached.

### Product Events (Plan & Product Syncing)
- **`product.created`**: Routes by `product_type` metadata — `product` → creates/updates product in `billing_product`; otherwise syncs plan if `plan_slug` is set.
- **`product.updated`**: Same routing as above — updates plan or product details when product is modified in Stripe.
- **`product.deleted`**: Deactivates the plan or product in the database.

### Price Events (Plan & Product Syncing)
- **`price.created`**: Retrieves the parent product to determine routing — re-syncs the plan or product (adding the new variant) accordingly.
- **`price.updated`**: Same as above — updates pricing for the affected plan or product variant.

> **Implementation**: See `src/app/api/webhooks/stripe/route.ts` for the complete webhook handler.

## Step 5: Configure Stripe Settings (Recommended)

### Limit to One Subscription Per Customer

1. Go to **Settings** > **Billing** > **Subscriptions and emails**
2. Find **Multiple subscriptions**
3. Select **Limit customers to one subscription**
4. Save

**Why**: Prevents race conditions when users open multiple checkout sessions.

### Disable Cash App Pay (Optional)

1. Go to **Settings** > **Payment methods**
2. Find **Cash App Pay** and toggle **OFF**

**Why**: High cancellation rate per t3dotgg recommendations.

### Enable Email Receipts

1. Go to **Settings** > **Emails**
2. Enable **Successful payments**
3. Customize template (optional)

## Step 6: Test the Integration

### Test Cards

| Card Number         | Scenario           |
|--------------------|--------------------|
| 4242 4242 4242 4242 | Success            |
| 4000 0000 0000 0341 | Declined           |
| 4000 0000 0000 9995 | Insufficient funds |
| 4000 0082 6000 0000 | 3D Secure required |

Use any future expiry date (e.g., 12/34), any CVC (e.g., 123), any ZIP (e.g., 12345).

### Testing Workflow

1. Start your app:
   ```bash
   pnpm dev
   ```

2. Start webhook forwarding (separate terminal):
   ```bash
   stripe listen --forward-to localhost:3000/api/webhooks/stripe
   ```

3. Sign up for an account in your app

4. Go to pricing page and click "Subscribe" on Pro plan

5. Complete checkout with test card `4242 4242 4242 4242`

6. Verify:
   - Redirected to `/settings/billing`
   - Subscription shows as "Active"
   - Payment method is saved
   - Database `billing_subscription` table updated
   - Webhook logs appear in Stripe CLI

7. Test cancellation and resumption

## Step 7: Understand Plan Syncing

### How It Works

Plans sync from Stripe to your database in two ways:

**1. Manual Sync (Initial Setup)**
```bash
pnpm db:seed:plans
```
- Fetches all active products from Stripe
- Syncs products with `plan_slug` metadata
- Creates or updates plans in database
- Stores product/price IDs in `providerData` field

**2. Automatic Sync (Webhooks)**

After webhook setup, plans sync automatically when you:
- Create a new product in Stripe → `product.created` event → plan created in DB
- Update product metadata → `product.updated` event → plan updated in DB
- Change pricing → `price.updated` event → plan re-synced in DB
- Delete product → `product.deleted` event → plan marked inactive in DB

**Benefits**:
- [YOURS] Single source of truth (Stripe dashboard)
- [YOURS] No manual database updates needed
- [YOURS] Real-time plan changes
- [YOURS] No environment variables for price IDs

### When to Manually Sync

- Initial database setup
- After webhook configuration changes
- If automatic sync fails
- To force re-sync all plans

## Step 8: Go Live

When ready for production:

1. Switch Stripe Dashboard to **Live Mode**
2. Get live API keys (`pk_live_*`, `sk_live_*`)
3. **Create products/prices in live mode** with the same metadata structure:
   - Plans: `plan_slug` (REQUIRED), `tier`, `sort_order`, `is_public` (Optional) — features via Stripe marketing features list
   - Lifetime pricing: add a one-time price to any plan product
   - Products: `product_type: product`, `product_slug` (REQUIRED), `is_public: true` (REQUIRED) — plus one-time prices with **Lookup key** (variant slug) and **Price description** (variant name) set under Advanced
4. Set up production webhook endpoint (`https://yourdomain.com/api/webhooks/stripe`)
5. Update production environment variables with live keys
6. Seed plans to production database:
   ```bash
   DATABASE_URL="postgresql://..." pnpm db:seed:plans
   ```

## Troubleshooting

**Webhook not receiving events:**
- Verify signing secret in `.env` matches Stripe CLI output
- Check Stripe Dashboard > Developers > Webhooks for logs
- Use ngrok for testing with external URLs

**Plans not syncing automatically:**
- Ensure webhook is subscribed to `product.*` and `price.*` events
- Verify product has `plan_slug` metadata in Stripe
- Check application logs for sync errors
- Manually run `pnpm db:seed:plans` to force sync

**Subscription not syncing:**
- Check database connection and schema
- Verify `billing_customer` record exists
- Check application logs for sync errors

**Checkout fails:**
- Ensure product is active in Stripe
- Verify plan exists in database (`billing_plan` table)
- Check that product has valid monthly price
- Check browser console for errors

**Plan missing after creating product:**
- Verify product has `plan_slug` metadata
- Check that product is marked as active in Stripe
- Manually trigger sync with `pnpm db:seed:plans`
- Check webhook logs in Stripe dashboard

**Payment method not saving:**
- Verify webhook received `checkout.session.completed`
- Check `billing_payment_method` table
- Review sync function logs

**Product not appearing on pricing page:**
- Verify product has both `product_type: product` and `product_slug` (e.g. `extra-fixes`) metadata in Stripe
- Verify `is_public: true` is set in product metadata
- Confirm each variant price has a **Lookup key** and **Price description** set (under Advanced) and is a one-time price (not recurring)
- Check the `billing_product` table — the row should exist with `is_public = true`
- If the row is missing, trigger a sync by updating the product in Stripe (fires `product.updated`) or check webhook logs for errors

**Lifetime tab not showing on pricing page:**
- Verify the plan product has a one-time price in addition to its recurring price(s)
- Run `pnpm db:seed:plans` to force-sync the new price into the database
- Check that the `billing_plan.provider_data` JSONB field contains a `lifetime` key in the prices map

## Architecture Pattern

JumpSaaS uses the t3dotgg pattern to eliminate race conditions:

```
User completes payment
  ↓
Stripe redirects to /api/billing/checkout/success
  ↓
Success handler calls syncStripeDataToDB() ← Immediate sync
  ↓
User sees subscription
  ↓
Webhook fires (may arrive later)
  ↓
Webhook calls syncStripeDataToDB() again ← Redundant but ensures consistency
```

By syncing in both the success handler AND webhook, we guarantee users see their subscription immediately while maintaining eventual consistency.

## Metadata-Based Product Sync

JumpSaaS uses a **metadata-driven approach** to sync products from Stripe to your database:

### Why Metadata?

Instead of hardcoding Product/Price IDs in `.env`, all plan configuration lives in Stripe metadata. This provides:

- **Single source of truth** - Stripe is the authoritative source
- **Dynamic updates** - Change plan features/order without code deployment
- **Multi-environment** - Same code works for test/production Stripe accounts
- **No environment variable bloat** - No need for `STRIPE_PRO_PRICE_ID`, etc.

### Required Metadata Fields

#### Plan Products

| Field | Required | Type | Description | Example |
|-------|----------|------|-------------|---------|
| `plan_slug` | [YOURS] Yes | string | Unique identifier for the plan | `pro`, `premium` |
| `tier` | No | string | Plan tier (free, starter, pro, premium, enterprise) | `pro` |
| `sort_order` | No | number | Display order (lower = first) | `1`, `2`, `3` |
| `is_public` | No | boolean | Whether to show on pricing page | `true` |
| `trial_period_days` | No | number | Free trial duration in days | `14`, `7`, `30` |

Add a **one-time price** to any plan product to enable lifetime purchasing for that plan. No additional metadata needed.

Add features using Stripe's built-in **marketing features** list on the product — not metadata.

#### Products

Set these on the **Stripe product**:

| Field | Required | Type | Description | Example |
|-------|----------|------|-------------|---------|
| `product_type` | [YOURS] Yes | string | Must be `product` to identify as a product | `product` |
| `product_slug` | [YOURS] Yes | string | Unique identifier for the product | `extra-fixes` |
| `is_public` | [YOURS] Yes | boolean | Must be `true` to show on pricing page | `true` |
| `sort_order` | No | number | Display order (lower = first) | `1`, `2`, `3` |
| `category` | No | string | Groups products under a heading on the pricing page | `Add-ons` |

Add features using Stripe's built-in **marketing features** list on the product — not metadata.

Set these on each **Stripe price** (one per variant) — no metadata needed, use the standard UI fields:

| Price field | Required | Description | Example |
|-------------|----------|-------------|---------|
| **Lookup key** | [YOURS] Yes | Unique slug for this variant — used as the identifier in code | `small`, `medium`, `large` |
| **Price description** (nickname) | [YOURS] Yes | Display name shown on the variant selector chip | `Small Pack`, `Medium Pack`, `Large Pack` |
| **Amount** | [YOURS] Yes | Price in your currency — variants are sorted cheapest → most expensive automatically | `$9.00`, `$19.00`, `$39.00` |

Each product price must be a **one-time price** (not recurring). A product with a single variant shows a simple buy button; multiple variants show a chip selector.

> **Note:** For detailed trial configuration instructions, see [Trial Configuration](../reference/billing#trial-configuration) in the Billing documentation.

### How Sync Works

1. **Fetch products** - `pnpm db:seed:plans` calls Stripe API
2. **Filter by metadata** - Only products with `plan_slug` are synced
3. **Map to database** - Metadata fields map to plan table columns
4. **Upsert** - Creates new plans or updates existing ones by slug

See `scripts/seed-plans.ts` and `src/lib/billing/providers/stripe.ts` for implementation.

## Production Deployment

### Critical: Build-Time Environment Variables

**IMPORTANT**: Public client-side variables are **baked into the JavaScript bundle at build time** — they cannot be overridden at runtime without a full rebuild. This applies to both frameworks, but the variable prefix differs:

| Framework | Public variable prefix | Example |
|-----------|----------------------|---------|
| Next.js | `NEXT_PUBLIC_` | `NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY` |
| TanStack (Vite) | `VITE_` | `VITE_STRIPE_PUBLISHABLE_KEY` |

#### Build-Time vs Runtime Variables

| Variable Type | When Set | Can Override at Runtime? | Where to Set |
|---------------|----------|-------------------------|--------------|
| Public (`NEXT_PUBLIC_*` / `VITE_*`) | **Build time** | No | `.env.production` file |
| Regular env vars | Runtime | Yes | Dokploy UI / Docker env |

#### Required Steps for Production

1. **Update `.env.production` BEFORE building**:

   Next.js:
   ```bash
   NEXT_PUBLIC_APP_URL=https://www.yourdomain.com
   NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY=pk_live_51abc...
   ```

   TanStack:
   ```bash
   VITE_APP_URL=https://www.yourdomain.com
   VITE_STRIPE_PUBLISHABLE_KEY=pk_live_51abc...
   ```

2. **Commit and push changes**:
   ```bash
   git add .env.production
   git commit -m "chore: update production domain and Stripe key"
   git push
   ```

3. **Trigger FULL rebuild in Dokploy**:
   - Not just redeploy — must rebuild to bake new values into the bundle
   - Environment variables set in the Dokploy UI do NOT override build-time vars

4. **Verify after deployment**:
   - Check browser network tab for auth API calls
   - Ensure requests go to your actual domain (not a placeholder)

#### Common Mistakes

**Wrong**: Setting the app URL only in Dokploy UI
- Result: Old placeholder value `https://yourdomain.com` still in bundle
- Symptom: CORS errors, OAuth "State not found" errors

**Correct**: Update `.env.production`, commit, push, rebuild
- Result: Correct domain baked into client bundle

#### Google OAuth Configuration

After updating your domain, configure Google Cloud Console:

1. Go to [Google Cloud Console](https://console.cloud.google.com/)
2. Navigate to **APIs & Services** > **Credentials**
3. Select your OAuth 2.0 Client ID
4. Update **Authorized redirect URIs**:
   ```
   https://www.jumpsaas.com/api/auth/callback/google
   ```
5. Save changes

#### GitHub OAuth Configuration

Similarly for GitHub:

1. Go to [GitHub Settings](https://github.com/settings/developers) > OAuth Apps
2. Select your application
3. Update **Authorization callback URL**:
   ```
   https://www.jumpsaas.com/api/auth/callback/github
   ```
4. Save changes

### Deployment Checklist

- [ ] `.env.production` has correct app URL (`NEXT_PUBLIC_APP_URL` for Next.js, `VITE_APP_URL` for TanStack)
- [ ] `.env.production` has correct Stripe publishable key (`NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY` for Next.js, `VITE_STRIPE_PUBLISHABLE_KEY` for TanStack) — must be `pk_live_...`
- [ ] Changes committed and pushed to git
- [ ] Full rebuild triggered in Dokploy (not just redeploy)
- [ ] Google OAuth redirect URI updated in Google Cloud Console
- [ ] GitHub OAuth redirect URI updated in GitHub Settings
- [ ] Stripe webhook endpoint configured with production URL
- [ ] All runtime env vars (secrets) set in Dokploy UI
- [ ] Database migrations applied to production
- [ ] Plans seeded to production database

## Security Checklist

- [ ] Webhook signature verification enabled
- [ ] API keys in environment variables only
- [ ] HTTPS in production
- [ ] User authentication required for checkout
- [ ] Database backups configured

## Resources

- [Stripe Documentation](https://stripe.com/docs)
- [Stripe API Reference](https://stripe.com/docs/api)
- [t3dotgg's Recommendations](https://github.com/t3dotgg/stripe-recommendations)
- [Billing Domain Architecture](../reference/billing)
