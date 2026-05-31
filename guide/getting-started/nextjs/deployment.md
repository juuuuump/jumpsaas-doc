---
title: "Deployment"
---

# Deployment

## Deployment Options

JumpSaaS supports two production models:

- **Server deployment** on the `main` branch. You can run this in containers such as Docker/Dokploy, or deploy it directly as a Node.js server on your VPS or preferred platform.
- **Edge deployment** on the `cloudflare` branch. This runs on Cloudflare Workers with the Cloudflare-specific runtime and database setup.

| | Server deployment | Edge deployment |
|---|---|---|
| Branch | `main` | `cloudflare` |
| Runtime | Node.js | Cloudflare Workers |
| Typical deploy targets | Docker, Dokploy, VPS, direct Node hosting | Cloudflare Workers |
| Database | Any PostgreSQL (self-hosted or cloud) | Neon PostgreSQL (required) |
| Database driver | `pg` (node-postgres) | `@neondatabase/serverless` (HTTP) |
| Build tool | `next build` | `opennextjs-cloudflare build` |
| Local dev database | Docker Compose by default | Neon cloud instance (always remote) |

### Choose Your Path

**Choose server deployment if you want:**
- Standard Next.js hosting
- Docker or Dokploy-based operations
- Direct Node.js deployment on a VPS or server
- Any PostgreSQL provider

**Choose edge deployment if you want:**
- Cloudflare Workers runtime
- Global edge execution
- A Cloudflare-specific deployment stack

**To use edge deployment:**
1. Read the [Cloudflare deployment overview](../../integrations/cloudflare-deployment) for prerequisites and trade-offs
2. Switch to the `cloudflare` branch: `git checkout cloudflare`
3. Follow the deployment guide at `docs/integrations/cloudflare.md` on that branch

The rest of this page covers the `main` branch server deployment path.

---

This guide focuses on Dokploy because it is the default self-hosted example, but the same `main` branch can also be deployed directly as a Node.js app without containers.

## How CI/CD Works

Every push to `main` triggers the GitHub Actions workflow at `.github/workflows/docker-image.yml`:

1. GitHub Actions builds a Docker image from your `Dockerfile`
2. The image is pushed to your Docker registry (tagged with the commit SHA)
3. After a successful push, GitHub Actions calls the Dokploy deploy webhook to trigger a redeploy

**Required GitHub secrets** (set in your repo under Settings → Secrets and variables → Actions):

| Secret | Description |
|--------|-------------|
| `DOCKER_USERNAME` | Docker registry username |
| `DOCKER_PASSWORD` | Docker registry password or access token |
| `DOKPLOY_DEPLOY_URL` | Dokploy webhook URL for your application |

Once configured, every merge to `main` deploys automatically — no manual steps needed for subsequent deploys.

## Server Deployment Modes

### Option A: Container-Based Deployment

Use this if you want a packaged runtime and a simple path to Dokploy, Coolify, Railway-style container hosting, or your own Docker infrastructure.

This is the workflow documented in the rest of this page.

### Option B: Direct Node.js Deployment

Use this if you want to deploy the `main` branch without containers, for example on a VPS with `systemd`, `pm2`, or a managed Node.js host.

Typical production commands:

```bash
pnpm install --frozen-lockfile
pnpm build
pnpm start
```

You still need the same production environment variables, database, migrations, Stripe webhook setup, and domain configuration described below. The main difference is operational: you manage the Node.js process directly instead of shipping a container image.

## First-Time Dokploy Setup

### 1. Install Dokploy

Follow the [Dokploy installation guide](https://docs.dokploy.com) to set up Dokploy on your server. A VPS with at least 2 GB RAM is recommended.

### 2. Create a Project and Application

1. Log in to your Dokploy dashboard
2. Create a new **Project**
3. Inside the project, create a new **Application**
4. Set the source to your Docker image (the registry image your GitHub Actions workflow pushes to)

### 3. Configure Environment Variables

Set runtime environment variables in the Dokploy UI under your application's **Environment** tab. See the [Environment Variables](#environment-variables) section below for the full list.

### 4. Configure the Deploy Webhook

1. In Dokploy, go to your application → **Deployments** → copy the webhook URL
2. Add it as the `DOKPLOY_DEPLOY_URL` secret in your GitHub repository

### 5. Deploy

Trigger the first deploy by pushing to `main` or clicking **Deploy** in Dokploy. Once the container starts, complete the [First-Deploy Checklist](#first-deploy-checklist).

## Environment Variables

### Build-Time vs Runtime — Critical Distinction

JumpSaaS uses two categories of environment variables:

**Build-time variables (`NEXT_PUBLIC_*`)**
- Baked into the JavaScript bundle when the Docker image is built
- **Cannot be changed after the image is built** — changing them in Dokploy has no effect
- Must be correct in `.env.production` before you push to `main`
- After updating `.env.production`, commit, push, and wait for a full rebuild

**Runtime variables**
- Read by the server at startup, not baked into the bundle
- Safe to update in the Dokploy UI — takes effect on next redeploy

### Build-Time Variables (`.env.production`)

Set these in your `.env.production` file before building:

```bash
NEXT_PUBLIC_APP_URL=https://www.yourdomain.com
NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY=pk_live_51abc...
```

> **Common mistake:** Setting `NEXT_PUBLIC_APP_URL` only in Dokploy causes CORS errors and OAuth failures because the value was not present when the image was built.

### Runtime Variables (Dokploy UI or your server process manager)

Set these in the Dokploy environment tab, or provide them to your direct Node.js process through your host's environment management:

```bash
# Database
DATABASE_URL=postgresql://user:password@host:5432/dbname

# Auth
BETTER_AUTH_SECRET=your-secret-here
BETTER_AUTH_URL=https://www.yourdomain.com

# Stripe
STRIPE_SECRET_KEY=sk_live_...
STRIPE_WEBHOOK_SECRET=whsec_...

# Email
RESEND_API_KEY=re_...
EMAIL_FROM=hello@yourdomain.com

# App
NODE_ENV=production
```

Add any other secrets your application requires (OAuth providers, S3 credentials, etc.).

## First-Deploy Checklist

After the container starts for the first time, complete these steps before sending traffic.

### 1. Run Database Migrations

Open a terminal into the running container via Dokploy, or SSH into your server if you are running Node.js directly, and run:

```bash
pnpm db:push
```

This creates all database tables from your Drizzle schema.

### 2. Seed Billing Plans

```bash
pnpm db:seed:plans
```

This creates the subscription plans in your database. Stripe products and prices must already exist — see [Stripe Setup](../../integrations/stripe) for how to create them.

### 3. Verify Stripe Webhooks

Stripe must be able to deliver events to your production URL. See [Stripe Setup → Webhooks](../../integrations/stripe#webhooks) for the full setup. The webhook endpoint is:

```
https://www.yourdomain.com/api/webhooks/stripe
```

### 4. Send a Test Email

Trigger a sign-up to confirm your email provider is configured and delivering messages.

## Custom Domain and SSL

Dokploy handles domain configuration and SSL termination via Let's Encrypt.

1. Go to your application in Dokploy → **Domains**
2. Add your domain (e.g. `www.yourdomain.com`)
3. Point your DNS A record to your server's IP address
4. Dokploy provisions the SSL certificate automatically

Allow a few minutes for DNS propagation and certificate issuance before testing.

If you are deploying Node.js directly, configure your reverse proxy and TLS the way you normally would for a Next.js app. The application requirements stay the same:

- Route traffic to the running Node.js server
- Terminate TLS at your proxy or platform
- Expose the public URL used by `NEXT_PUBLIC_APP_URL` and `BETTER_AUTH_URL`

## Verifying the Deploy

Once the container is running and the checklist is complete, verify the following:

- **Sign-up works** — create a new account end-to-end
- **Billing works** — complete a checkout flow with a Stripe test card (if still testing) or a real card
- **Emails send** — check that the welcome email arrives
- **Webhooks deliver** — check the Stripe dashboard under Developers → Webhooks for successful event deliveries
- **Auth redirects correctly** — log out and log back in; confirm no double-locale URLs in the address bar

## Rollback

If a deploy causes issues, roll back to a previous image via Dokploy:

1. Go to your application → **Deployments**
2. Find a previous successful deployment
3. Click **Redeploy** on that entry

Dokploy pulls the exact image tag from that deployment and restarts the container. No code changes required.

Alternatively, revert your commit on `main` and push — GitHub Actions will build and deploy the reverted code automatically.

---

When your deploy is stable and verified, work through the [Pre-Launch Checklist](../../PRE_LAUNCH_CHECKLIST) before going live.
