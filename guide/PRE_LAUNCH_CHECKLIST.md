---
title: "Pre-Launch Checklist"
---

# Pre-Launch Checklist

Complete these steps before deploying to production.

## 1. Environment Variables

> **Framework note — build-time variable prefixes differ:**
> - **Next.js**: public client-side variables use the `NEXT_PUBLIC_` prefix and are baked into the bundle at build time. Set them in `.env.production` before building.
> - **TanStack (Vite)**: public client-side variables use the `VITE_` prefix and are also baked in at build time. Set them in `.env.production` before building.

### Build-Time (must be set in `.env.production` before building)
- [ ] App name — `NEXT_PUBLIC_APP_NAME` (Next.js) / `VITE_APP_NAME` (TanStack)
- [ ] App URL — `NEXT_PUBLIC_APP_URL` (Next.js) / `VITE_APP_URL` (TanStack) — your production domain (e.g. `https://www.myapp.com`)
- [ ] Stripe publishable key — `NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY` (Next.js) / `VITE_STRIPE_PUBLISHABLE_KEY` (TanStack) — **use `pk_live_...`**, NOT `pk_test_...`

### Runtime (set in Dokploy / docker run -e / your hosting platform)
- [ ] `DATABASE_URL` — production PostgreSQL connection string
- [ ] `BETTER_AUTH_SECRET` — random 32+ char string (`openssl rand -base64 32`)
- [ ] `BETTER_AUTH_URL` — your production domain (same value as the app URL above)
- [ ] `GITHUB_CLIENT_ID` / `GITHUB_CLIENT_SECRET` — from your GitHub OAuth App
- [ ] `GOOGLE_CLIENT_ID` / `GOOGLE_CLIENT_SECRET` — from Google Cloud Console
- [ ] `RESEND_API_KEY` — from Resend dashboard
- [ ] `FROM_EMAIL` — a verified sender address in your Resend account
- [ ] `STRIPE_SECRET_KEY` — `sk_live_...` from Stripe dashboard
- [ ] `STRIPE_WEBHOOK_SECRET` — from Stripe webhook endpoint config
- [ ] `STRIPE_PRO_PRICE_ID` / `STRIPE_PREMIUM_PRICE_ID` — your live Stripe price IDs
- [ ] Storage variables (`STORAGE_BUCKET`, `STORAGE_ACCESS_KEY_ID`, etc.)

## 2. Stripe Setup
- [ ] Create products and prices in Stripe live mode
- [ ] Set up webhook endpoint pointing to `https://yourdomain.com/api/webhooks/stripe`
- [ ] Subscribe to: `customer.subscription.created`, `customer.subscription.updated`, `customer.subscription.deleted`, `invoice.payment_succeeded`, `invoice.payment_failed`
- [ ] Run `pnpm db:seed:plans` against production DB after first deploy

See [Stripe Setup](integrations/stripe) for full details.

## 3. Authentication
- [ ] GitHub OAuth App: update callback URL to `https://yourdomain.com/api/auth/callback/github`
- [ ] Google OAuth App: update authorized redirect URI to `https://yourdomain.com/api/auth/callback/google`
- [ ] Verify `BETTER_AUTH_URL` matches your production domain exactly

## 4. Legal Pages
- [ ] Replace all `[PLACEHOLDER]` values in `messages/en/legal.json` and `messages/de/legal.json`
- [ ] Update `LAST_UPDATED` dates in privacy, terms, and cookies pages
- [ ] Have a lawyer review the content (the template is a starting point, not legal advice)

## 5. Branding
- [ ] Update app name env var in `.env.production` — `NEXT_PUBLIC_APP_NAME` (Next.js) or `VITE_APP_NAME` (TanStack)
- [ ] Update app name, logo, and colors in `src/components/` as needed
- [ ] Update `messages/en.json` and `messages/de.json` with your product's copy

## 6. Email
- [ ] Verify your sender domain in Resend
- [ ] Set `FROM_EMAIL` to a verified sender address
- [ ] Send a test email using the forgot-password flow

## 7. Deployment
- [ ] Set `DOKPLOY_DEPLOY_URL` as a GitHub Secret (for CI/CD auto-deploy)
- [ ] Run database migrations: `pnpm db:push` on first deploy
- [ ] Test sign-up, sign-in, and billing flow end-to-end on production

> **Deploying to Cloudflare instead?** The deployment steps in this section are Docker/Dokploy-specific. Switch to the `cloudflare` branch (`git checkout cloudflare`) and follow its deployment guide instead. The rest of this checklist (auth, legal, branding, email) still applies. See [Cloudflare Deployment](integrations/cloudflare-deployment) for an overview.

See [Deployment (Next.js)](getting-started/nextjs/deployment) or [Deployment (TanStack)](getting-started/tanstack/deployment) for full details.

## 8. Optional Integrations
- [ ] Slack: Set `SLACK_CLIENT_ID`, `SLACK_CLIENT_SECRET`, `SLACK_SIGNING_SECRET` (or remove the Slack integration if unused)
- [ ] Storage: Configure Cloudflare R2 (`CLOUDFLARE_ACCOUNT_ID`, `STORAGE_BUCKET`, `STORAGE_ACCESS_KEY_ID`, `STORAGE_SECRET_ACCESS_KEY`, and either `STORAGE_PUBLIC_URL` or `R2_PUBLIC_URL_DOMAIN`)
