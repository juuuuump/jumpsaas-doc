---
title: "Cloudflare Deployment"
---

# Cloudflare Deployment

> **Next.js only.** This guide applies to the `jumpsaas-nextjs` template. The TanStack variant deploys as a standard Node.js server — see its [Deployment](../getting-started/tanstack/deployment) guide.

JumpSaaS supports deploying to Cloudflare Workers as the edge deployment path. This option lives on the `cloudflare` branch and uses OpenNext with Neon PostgreSQL.

## Server vs Edge

| | Server deployment | Edge deployment |
|---|---|---|
| Branch | `main` | `cloudflare` |
| Runtime | Node.js | Cloudflare Workers |
| Typical deploy targets | Docker, Dokploy, VPS, direct Node hosting | Cloudflare Workers |
| Database | Any PostgreSQL (self-hosted or cloud) | Neon PostgreSQL (required) |
| Database driver | `pg` (node-postgres) | `@neondatabase/serverless` (HTTP) |
| Build tool | `next build` | `opennextjs-cloudflare build` |
| Local dev database | Docker Compose by default | Neon cloud instance (always remote) |

The database schema, auth, billing, and all application code are identical between both branches.

## Prerequisites

Required before switching to the Cloudflare branch:

- [Cloudflare account](https://cloudflare.com) (free tier works)
- [Neon account](https://neon.tech) with a PostgreSQL database (free tier works)

Optional (same as the server deployment path):
- [Stripe account](https://stripe.com) — required only if you use billing
- [Resend account](https://resend.com) — required only if you use transactional email

> **Note:** The Cloudflare branch requires a Neon cloud database in all environments, including local development. Docker Compose and direct local PostgreSQL workflows are not used on that branch.

## Getting Started

1. Switch to the `cloudflare` branch: `git checkout cloudflare`
2. Follow the deployment guide at `docs/integrations/cloudflare.md` on that branch

## Keeping in Sync

The `cloudflare` branch periodically merges `main` to stay up to date:

```bash
git fetch origin
git merge origin/main
```
