---
title: "Deployment"
---

# Deployment

Deploy JumpSaaS to production on a standard Node.js server.

## Deployment Options

JumpSaaS deploys as a Node.js server application. The recommended path is a Docker container managed by **Dokploy**, but any Node.js-capable host works — a raw VPS, a PaaS like Railway or Render, or your own Docker setup.

The production entry point is `server.mjs`. The build command is `vite build`.

## Prerequisites

- A PostgreSQL database (managed or self-hosted)
- A server or PaaS that can run Node.js 20+
- All [required environment variables](#environment-variables) set in the target environment

---

## Dokploy + Docker (Recommended)

This is the path the CI/CD pipeline is built around. Dokploy is a self-hostable deployment platform — think a lightweight Heroku you run on your own VPS.

### 1. Set up Dokploy

Follow the [Dokploy quickstart](https://dokploy.com/docs) to install it on a VPS (a $6/mo Hetzner or DigitalOcean instance works fine).

### 2. Create a new application in Dokploy

- **Source:** connect your GitHub repo
- **Build type:** Dockerfile
- **Branch:** `main`

### 3. Configure environment variables

In the Dokploy application settings, add all [runtime environment variables](#environment-variables). See the section below for the full list.

Build-time variables (`VITE_PUBLIC_*`) must be present **at build time** — set them in `.env.production` (tracked in the repo) or inject them into the Docker build via Dokploy's build args. See [Build-time vs runtime variables](#build-time-vs-runtime-variables).

### 4. Set up the domain and SSL

Add your domain in Dokploy and enable the built-in Traefik reverse proxy with Let's Encrypt. Point your DNS A record at the VPS IP.

### 5. Deploy

Push to `main`. GitHub Actions runs the CI pipeline (lint, type-check, tests), then triggers a Dokploy deploy via webhook. See `.github/workflows/ci.yml` for the full pipeline.

### 6. First-deploy checklist

After the first successful deploy:

1. **Run migrations:** `pnpm db:push` against your production database (or exec into the container)
2. **Seed plans:** `pnpm db:seed:plans` — populates the Stripe plan data
3. **Verify Stripe webhooks:** the webhook endpoint is `https://www.yourdomain.com/api/webhooks/stripe`
4. **Test email:** trigger a sign-up and confirm the verification email arrives

---

## Environment Variables

### Build-time vs runtime variables

JumpSaaS has two categories of env vars:

**Build-time (`VITE_PUBLIC_*`)** — baked into the client bundle at build time by Vite. These must be available when `vite build` runs. Add them to `.env.production` (which is tracked in the repo — don't put secrets here) or inject them as Docker build args.

```bash
VITE_PUBLIC_APP_NAME=YourApp
VITE_PUBLIC_APP_URL=https://www.yourdomain.com
VITE_PUBLIC_STRIPE_PUBLISHABLE_KEY=pk_live_...
```

Access them in code via `import.meta.env.VITE_PUBLIC_*`.

**Runtime** — read by the Node.js server process at startup. Set these in Dokploy's environment variable UI (or your host's equivalent) — never commit secrets to the repo.

```bash
DATABASE_URL=postgresql://user:password@host:5432/dbname
BETTER_AUTH_SECRET=<random-32-char-string>
BETTER_AUTH_URL=https://www.yourdomain.com
STRIPE_SECRET_KEY=sk_live_...
STRIPE_WEBHOOK_SECRET=whsec_...
RESEND_API_KEY=re_...
FROM_EMAIL=hello@yourdomain.com
NODE_ENV=production
```

> **Note:** `BETTER_AUTH_URL` must match `VITE_PUBLIC_APP_URL` exactly (same protocol, same domain, no trailing slash).

### Full variable reference

See `.env.example` in the repo root for the complete list with inline comments. That file is the source of truth — if a variable exists in the codebase, it's documented there.

---

## Docker

The repo ships a `Dockerfile`. To build and run manually:

```bash
# Build
docker build -t my-app .

# Run
docker run -p 3000:3000 \
  -e DATABASE_URL=postgresql://... \
  -e BETTER_AUTH_SECRET=... \
  -e BETTER_AUTH_URL=https://www.yourdomain.com \
  -e NODE_ENV=production \
  my-app
```

For local Docker Compose (with a Postgres container bundled), see `docker-compose.yml`.

---

## CI/CD Pipeline

The pipeline is in `.github/workflows/ci.yml`. On push to `main` it:

1. Installs dependencies (`pnpm install`)
2. Runs type-check (`pnpm typecheck`)
3. Runs lint (`pnpm lint`)
4. Runs tests (`pnpm test`)
5. Triggers a Dokploy deploy via webhook (set `DOKPLOY_WEBHOOK_URL` in GitHub Secrets)

Deploys only happen when all checks pass.

---

## Alternative Hosts

If you're not using Dokploy, the key facts are:

- **Build command:** `vite build`
- **Start command:** `node server.mjs`
- **Node.js version:** 20+
- **Port:** `3000` by default (set `PORT` env var to override)

Railway, Render, Fly.io, and similar PaaS providers all support this pattern out of the box via their Dockerfile or Node.js buildpack detection.
