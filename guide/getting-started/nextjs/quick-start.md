---
title: "Quick Start"
---

# Quick Start

Get JumpSaaS running locally in 5 minutes.

## Prerequisites

- Node.js 20+
- pnpm (`npm install -g pnpm`)
- Docker (for the default local PostgreSQL workflow)

> **Deployment paths:** the `main` branch supports standard server deployment, either with containers such as Docker/Dokploy or as a direct Node.js deployment. JumpSaaS also supports edge deployment on Cloudflare Workers via the `cloudflare` branch. See [Deployment Options](deployment#deployment-options) to compare before you choose a production path.

## 0. Clone the repository

```bash
git clone git@github.com:juuuuump/jumpsaas.git my-app
cd my-app
```

Rename the template remote and add your own project repo as `origin`:

```bash
git remote rename origin template
git remote add origin git@github.com:your-org/my-app.git
```

This keeps the template as a named remote so you can pull future updates with `git fetch template`. See [Updating](../../customizing/updating) for the full update workflow.

## 1. Install dependencies

```bash
pnpm install
```

## 2. Start the database

```bash
docker compose up -d
```

## 3. Set up environment

The repo ships a committed `.env.example` that lists every supported variable with placeholder values and inline comments. It's the source of truth for which env vars exist — copy it to create your local `.env`:

```bash
cp .env.example .env
```

Your `.env` is git-ignored; only `.env.example` and `.env.production` are tracked. When you add a new env var to the codebase, also add it to `.env.example` (with a safe placeholder, never a real secret) so other contributors and deploys pick it up.

Edit `.env` — minimum required values:

```bash
DATABASE_URL=postgresql://user:password@localhost:5432/jumpsaas
BETTER_AUTH_SECRET=any-random-32-char-string-here
BETTER_AUTH_URL=http://localhost:3000
RESEND_API_KEY=re_your_key       # Required — get from resend.com
FROM_EMAIL=noreply@yourdomain.com
```

> **Note:** `FROM_EMAIL` is required — the app will throw at startup if missing.

## 4. Run migrations

```bash
pnpm db:push
```

## 5. Start the dev server

```bash
pnpm dev
# → http://localhost:3000
```

## What You'll See

- Landing page with pricing at `/`
- Sign up / sign in (email + OAuth) at `/sign-in`
- User settings at `/settings`
- Billing dashboard at `/settings/billing`
- Admin dashboard at `/admin`

## Next Steps

- [Template Boundaries](template-boundaries) — understand what's safe to modify
- [Adding Features](../../customizing/adding-features/nextjs) — start building your product
- [Stripe Setup](../../integrations/stripe) — wire up payments (required for billing to work)
- [Pre-Launch Checklist](../../PRE_LAUNCH_CHECKLIST) — before you go live
