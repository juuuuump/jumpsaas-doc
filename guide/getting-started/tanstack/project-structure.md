---
title: "Project Structure"
---

# Project Structure

A map of the codebase — what lives where and why.

## Tech Stack

| Layer | Technology |
|---|---|
| Framework | TanStack Start |
| Router | TanStack Router (file-based, type-safe) |
| UI library | React 19 |
| Language | TypeScript (strict mode) |
| Build tool | Vite |
| Styling | Tailwind CSS v4 |
| UI primitives | Radix UI |
| Icons | Phosphor Icons |
| Auth | Better Auth |
| Database | PostgreSQL + Drizzle ORM |
| Forms | React Hook Form + Zod |
| Email | Resend + React Email |
| File uploads | better-upload + Cloudflare R2 / AWS S3 |
| i18n | Paraglide JS + Inlang |

## Directory Overview

```
src/
├── routes/                    # File-based routing (TanStack Router)
│   ├── __root.tsx             # Root document shell
│   ├── api/
│   │   ├── auth/$.ts          # Better Auth handler → /api/auth/*
│   │   ├── upload.avatar.ts   # File upload → /api/upload/avatar
│   │   └── webhooks.stripe.ts # Stripe webhooks → /api/webhooks/stripe
│   └── {-$locale}/            # Optional locale prefix (en, de, etc.)
│       ├── route.tsx          # Locale param setup + Paraglide binding
│       ├── index.tsx          # / (marketing home)
│       ├── pricing.tsx        # /pricing
│       ├── legal/             # /legal/privacy, /legal/terms, /legal/cookies
│       ├── _auth/             # Auth layout (guest-only — redirects authed users)
│       │   ├── route.tsx      #   Layout + beforeLoad guard
│       │   ├── sign-in.tsx    #   /sign-in
│       │   ├── sign-up.tsx    #   /sign-up
│       │   ├── forgot-password.tsx
│       │   ├── reset-password.tsx
│       │   └── verify-email.tsx
│       ├── _app/              # App layout (authenticated)
│       │   ├── route.tsx      #   Layout + beforeLoad guard
│       │   └── app/index.tsx  #   /app (main dashboard)
│       ├── _protected/        # Protected layout (authenticated)
│       │   ├── route.tsx      #   beforeLoad guard
│       │   ├── settings/      #   /settings/profile, billing, security, etc.
│       │   └── admin/         #   /admin (optional, can delete)
│       └── -components/       # Route-collocated components (not a route)
│           └── not-found.tsx
│
├── server/                    # Server-only code (TanStack Start server fns)
│   └── saas/
│       ├── auth.ts            # getSession, signOut, listSessions, etc.
│       ├── billing.ts         # Server functions for billing
│       ├── plan.ts
│       └── ...
│
├── components/
│   ├── ui/                    # Radix/shadcn UI primitives
│   ├── shared/                # App-wide infrastructure components
│   ├── saas/                  # SaaS feature components
│   ├── marketing/             # Marketing page components
│   └── app/                   # [Custom] Your components
│
├── services/
│   ├── saas/                  # Template services (billing, auth, etc.)
│   └── app/                   # [Custom] Your services
│
├── lib/
│   ├── saas/                  # Template SaaS infrastructure
│   │   ├── auth.ts            #   Better Auth config
│   │   ├── email/             #   Email (Resend + React Email)
│   │   ├── upload/            #   File upload helpers
│   │   └── ...
│   ├── shared/                # Shared utilities
│   └── app/                   # [Custom] Your utilities
│
├── db/
│   └── schema/
│       ├── saas/              # Template schemas (auth, billing)
│       ├── app/               # [Custom] Your database tables
│       └── index.ts
│
├── i18n/                      # Paraglide i18n helpers
│   ├── navigation.tsx         # Locale-aware Link (wraps TanStack Router)
│   └── use-translations.ts    # Client-side translation hook
│
└── paraglide/                 # Generated — compiled Paraglide output
    ├── messages.js
    ├── runtime.js
    └── server.js

messages/                      # Translation source files
├── en.json                    # English
└── de.json                    # German

public/                        # Static assets
docs/                          # Documentation
```

## Routing Conventions

Routes live in `src/routes/` and are picked up automatically by TanStack Router. Each route file exports a route created with `createFileRoute()`.

### Locale prefix: `{-$locale}/`

The outer `{-$locale}/` directory wraps all user-facing pages. The curly-brace syntax denotes a path parameter; the leading `-` makes it optional. This means `/about` and `/de/about` both resolve to the same route — Paraglide JS handles the locale-aware URL rewriting transparently through the router config, so you don't need to think about it when building pages.

### Layout routes: `_auth/`, `_protected/`, `_app/`

Directories prefixed with `_` are **layout routes** — they provide a shared layout and/or a `beforeLoad` guard but do not add a segment to the URL. For example, `_protected/settings/profile.tsx` maps to `/settings/profile`, not `/protected/settings/profile`.

This is how route-level auth guards work in TanStack Start. The `route.tsx` inside each layout directory runs `beforeLoad` to check the session and redirect unauthenticated users — there is no global `middleware.ts`.

### Collocated components: `-components/`

Directories prefixed with `-` are ignored by the router. Use them for components that belong logically next to their routes but shouldn't become routes themselves.

### Splat routes: `$.ts`

`$` in a filename is a catch-all (splat) segment. `routes/api/auth/$.ts` forwards all requests under `/api/auth/*` to the Better Auth handler.

### API routes: `routes/api/`

Server-side API handlers live in `routes/api/`. Dots in the filename map to slashes in the URL: `upload.avatar.ts` → `/api/upload/avatar`.

## Where Does My Code Go?

| What you're building | Where to put it |
|---|---|
| New pages / routes | `src/routes/{-$locale}/_app/` or `_protected/` |
| New API endpoints | `src/routes/api/` |
| UI components | `src/components/app/` |
| Business logic / services | `src/services/app/` |
| Server functions | `src/server/app/` |
| Database tables | `src/db/schema/app/` |
| Utility functions | `src/lib/app/` |
| Translations | `messages/en.json` and `messages/de.json` |

The `app/` subdirectory inside each top-level directory is the designated space for your own code. Template code lives in `saas/`, `shared/`, `marketing/`, and `ui/` — these you generally leave alone. See [Template Boundaries](template-boundaries) for the full breakdown of what's safe to modify.

## Key Files

| File | Purpose |
|---|---|
| `src/routes/__root.tsx` | HTML document shell, global providers, router devtools |
| `src/routes/{-$locale}/route.tsx` | Sets the active Paraglide locale for the current request |
| `src/lib/saas/auth.ts` | Better Auth instance — plugins, providers, session config |
| `src/db/schema/index.ts` | Barrel export of all Drizzle schema tables |
| `vite.config.ts` | Vite + TanStack Start plugin config |
| `app.config.ts` | TanStack Start app config (server entry, SSR settings) |
| `.env.example` | Canonical list of all supported environment variables |

## Import Aliases

```ts
import something from "#/components/ui/button"   // src/components/ui/button
import something from "@/lib/saas/auth"           // src/lib/saas/auth (also works)
```

Both `#/` and `@/` resolve to `src/`. Prefer `#/` for new code — it's the primary alias used throughout the template.
