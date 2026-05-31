# jumpsaas-doc

Shared documentation source for [JumpSaaS](https://jumpsaas.com) — production-ready SaaS starter templates for Next.js and TanStack Start.

## What this repo is

This repo is the **single source of truth** for all user-facing JumpSaaS documentation. It is consumed by `jumpsaas-web` to render the docs section at [jumpsaas.com/docs](https://www.jumpsaas.com/docs).

Do not edit documentation inside `jumpsaas-nextjs`, `jumpsaas-tanstack`, or `jumpsaas-web` directly — always edit here.

## Structure

```
guide/
├── getting-started/
│   ├── nextjs/            # Quick start, project structure, template boundaries, deployment (Next.js)
│   └── tanstack/          # Same four pages for TanStack Start
├── customizing/
│   ├── adding-features/
│   │   ├── index.md       # Framework picker
│   │   ├── nextjs.md      # Next.js step-by-step guide
│   │   └── tanstack.md    # TanStack step-by-step guide
│   ├── conventions.md     # Code conventions (shared, with framework callouts)
│   └── updating.md        # How to receive template updates (shared)
├── integrations/          # Stripe, email (Resend), storage (R2/S3), Cloudflare
├── reference/
│   ├── architecture/
│   │   ├── index.md       # Framework picker
│   │   ├── nextjs.md      # Next.js architecture reference
│   │   └── tanstack.md    # TanStack architecture reference
│   └── *.md               # billing, database, security, SEO, services, troubleshooting (shared)
├── architecture/          # Template architecture notes
└── testing/               # Testing guides (shared)
```

Each section has an `index.md` (section landing page) and a `meta.json` (nav order and titles for `jumpsaas-web`).

## How framework coverage works

- **Fully split** — pages where Next.js and TanStack differ substantially (routing, server functions, i18n) live in separate `nextjs.md` / `tanstack.md` files inside a subdirectory with a framework-picker `index.md`.
- **Shared with callouts** — pages that are mostly identical use inline callout blocks (`> **Next.js only.**`) or labelled code blocks for the parts that differ.
- **Fully shared** — pages where both frameworks are identical (billing, database, security, testing) have no framework labels at all.

## Contributing

1. Edit `.md` files in `guide/` directly — no build step required.
2. Every `.md` file needs `---\ntitle: "..."\n---` frontmatter.
3. Nav order and section titles are controlled by `meta.json` files in each directory. Adding a new page or directory requires updating the parent `meta.json`.
4. After editing, sync to `jumpsaas-web` and verify locally:
   ```bash
   # from the jumpsaas-web directory
   node scripts/sync-docs.mjs
   pnpm dev
   ```

See [CLAUDE.md](CLAUDE.md) for the full editing rules, link depth reference, and framework coverage decision tree.
