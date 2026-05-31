---
title: JumpSaaS Documentation
description: Production-ready SaaS starter templates for Next.js and TanStack.
---

# JumpSaaS Documentation

The SaaS starter for shipping real products fast — available in two framework flavours.

- Auth, billing, email, storage, and i18n included
- Ship your SaaS in a weekend
- Customize without fighting the template
- Deploy on your server or at the edge

## Pick Your Framework

| | Next.js | TanStack |
|---|---------|----------|
| **Routing** | App Router (React Server Components) | File-based routes + server functions |
| **Build tool** | Next.js / Webpack / Turbopack | Vite |
| **i18n library** | next-intl | ParaglideJS |
| **Edge deploy** | Cloudflare Workers (`cloudflare` branch) | Not yet |
| **Deploy targets** | Vercel, Node, Docker, Cloudflare | Node, Docker, any VPS |
| **Best if you...** | Want RSC, strong Vercel integration, or Cloudflare edge | Prefer Vite, TanStack Router type-safety, or a leaner runtime |
| **Quick Start** | [Get started](getting-started/nextjs/quick-start) | [Get started](getting-started/tanstack/quick-start) |

Both templates ship identical features (auth, billing, email, storage, i18n). The choice is about runtime model and ecosystem preference, not feature coverage.

## Start Here

New to JumpSaaS? Follow this path:

1. Pick a framework above and follow its Quick Start — running locally in 5 minutes
2. [Template Boundaries](getting-started/nextjs/template-boundaries) — what you can and can't modify
3. [Adding Features — Next.js](customizing/adding-features/nextjs) · [TanStack](customizing/adding-features/tanstack) — build your product on top
4. [Pre-Launch Checklist](PRE_LAUNCH_CHECKLIST) — before you go live

## What's Built In

| Feature | Doc |
|---------|-----|
| Auth (GitHub, Google, email/password) | [Architecture — Next.js](reference/architecture/nextjs) · [TanStack](reference/architecture/tanstack) |
| Stripe billing + subscriptions | [Billing](reference/billing) · [Stripe Setup](integrations/stripe) |
| File uploads (R2 / S3) | [Storage](integrations/storage) |
| Email (Resend) | [Email](integrations/email) |
| i18n (EN + DE) | [Conventions](customizing/conventions) |
| SEO + sitemap | [SEO](reference/seo) |
| Legal pages | [Legal Pages](reference/legal-pages) |
| Deployment paths | [Deployment (Next.js)](getting-started/nextjs/deployment) · [Deployment (TanStack)](getting-started/tanstack/deployment) · [Cloudflare Edge](integrations/cloudflare-deployment) |

## FAQ

**Can I use this without Stripe?**
Yes — billing is optional. Remove the Stripe env vars and the billing UI won't render.

**Do I need Cloudflare?**
No. Both templates support standard server deployment with Docker or Node.js. Cloudflare edge deployment is a Next.js-specific option — see [Cloudflare Deployment](integrations/cloudflare-deployment).

**Can I add more languages?**
Yes — Next.js uses `next-intl` (add a locale to `src/i18n/routing.ts` and create `messages/[locale].json`). TanStack uses ParaglideJS (add a locale to `project.inlang/` and create the corresponding messages file).

**What happens when I update the template?**
See [Updating](customizing/updating) — the `saas/` vs `app/` directory separation means most updates have 2-3 file conflicts instead of 20+.
