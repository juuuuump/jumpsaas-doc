---
title: "Cloudflare Workers Deployment"
---

# Cloudflare Workers Deployment

> **Next.js only.** This guide applies to the `jumpsaas-nextjs` template. The TanStack variant deploys as a standard Node.js server — see its [Deployment](../getting-started/tanstack/deployment) guide.

This branch (`cloudflare`) deploys JumpSaaS to Cloudflare Workers using OpenNext and Neon PostgreSQL.

## How it differs from the main branch

| | Main branch | Cloudflare branch |
|---|---|---|
| Deployment | Docker / VPS (Dokploy) | Cloudflare Workers |
| Database driver | `pg` (node-postgres) | `@neondatabase/serverless` (HTTP) |
| Database host | Any PostgreSQL (self-hosted or cloud) | Neon (required) |
| Build tool | `next build` | `opennextjs-cloudflare build` |
| Local dev | `pnpm dev` | `pnpm dev` or `pnpm cf:preview` |

The database schema, auth, billing, and all application code are identical.

## Prerequisites

- [Cloudflare account](https://cloudflare.com) (free tier works)
- [Neon account](https://neon.tech) with a PostgreSQL database (free tier works)
- [Stripe account](https://stripe.com) for billing
- [Resend account](https://resend.com) for transactional email

## Local Development

> **This branch requires a Neon cloud database in all environments.** The `neon()` HTTP client cannot connect to a local PostgreSQL socket — `DATABASE_URL` must always point to a Neon instance. `docker-compose.yml` is not used on this branch.

For day-to-day development, use the standard Next.js dev server (pointing to your Neon DB):

```bash
# .env.local must contain a Neon DATABASE_URL
pnpm dev
```

To test in the actual Workers runtime locally:

```bash
cp .dev.vars.example .dev.vars
# Fill in .dev.vars with your credentials
pnpm cf:preview
# Visit http://localhost:8787
```

## Setting Up Neon

1. Create a project at [neon.tech](https://neon.tech)
2. Copy the connection string (PostgreSQL format):
   `postgresql://user:password@ep-xxx.region.aws.neon.tech/neondb?sslmode=require`
3. Add to `.env.local` as `DATABASE_URL`
4. Run migrations: `pnpm db:push`

## Deploying to Cloudflare

```bash
# 1. Authenticate
pnpm wrangler login

# 2. Set secrets
pnpm wrangler secret put DATABASE_URL
pnpm wrangler secret put BETTER_AUTH_SECRET
# ... (see wrangler.toml comments for full list)

# 3. Build and deploy
pnpm cf:build && pnpm cf:deploy
```

## Keeping in Sync with the Main Branch

To pull in updates from the main template:

```bash
git fetch origin
git merge origin/main
```

Files that differ from main and will accumulate merge conflicts: `src/db/index.ts`, `next.config.ts`, `open-next.config.ts`, `wrangler.toml`, `package.json`, `.gitignore`. Files added only on this branch (`open-next.config.ts`, `wrangler.toml`, `.dev.vars.example`, `docs/integrations/cloudflare.md`) will never conflict.
