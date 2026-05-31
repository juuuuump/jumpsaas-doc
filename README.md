# JumpSaaS Documentation

User-facing guide for [JumpSaaS](https://jumpsaas.com) — production-ready SaaS starter templates for Next.js and TanStack Start.

The rendered docs are available at [jumpsaas.com/docs](https://www.jumpsaas.com/docs).

## What's covered

| Section | What you'll find |
|---------|-----------------|
| **Getting Started** | Install the template, run it locally, understand the project structure, and choose your deployment path — with separate guides for Next.js and TanStack |
| **Customizing** | Code conventions, how to add your own features without breaking template updates, and how to receive future updates |
| **Integrations** | Step-by-step setup for Stripe, Resend email, Cloudflare R2 storage, and Cloudflare deployment |
| **Reference** | Architecture deep-dives, billing model, database schema, security, SEO, legal pages, and troubleshooting |
| **Testing** | Manual testing guides for billing flows and email notifications |

## Two frameworks, one feature set

JumpSaaS ships as two templates — Next.js and TanStack Start — with identical features but different implementation patterns. Where the two frameworks diverge (routing, server functions, i18n), the docs provide parallel guides. Where they're identical (Stripe billing, database, email), one shared doc covers both.

## Editing these docs

This repo is the source. `jumpsaas-web` consumes it to render the docs site — do not edit docs there directly. See [CLAUDE.md](CLAUDE.md) for editing conventions.
