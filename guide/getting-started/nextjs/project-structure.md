---
title: "Project Structure"
---

# Project Structure

An overview of what's in this template and where things live.

## Tech Stack

### Core Framework
- **Next.js** - React framework with App Router
- **React 19** - UI library
- **TypeScript** (strict mode) - Type safety throughout
- **Turbopack** - Fast build tool (Next.js built-in)

### Styling & UI
- **Tailwind CSS v4** - Utility-first CSS framework
- **Radix UI** - Headless, accessible UI component primitives
- **Phosphor Icons** - Icon library
- **next-themes** - Dark/light mode theming
- **Geist Sans & Mono** - Modern font family

### Authentication & Security
- **Better Auth** - Modern authentication library
- Email/password, Google OAuth, GitHub OAuth
- Email verification and secure password reset built in

### Database & ORM
- **PostgreSQL** - Production-grade relational database
- **Drizzle ORM** - Type-safe SQL ORM
- **Docker Compose** - Local database setup

### Forms & Validation
- **React Hook Form** - Performant form library
- **Zod** - TypeScript-first schema validation

### Email System
- **Resend** - Email delivery service
- **React Email** - Type-safe email templates

### File Storage
- **better-upload** - Upload library with React hooks
- **Cloudflare R2** (recommended) or **AWS S3** - Object storage
- Pre-signed URLs for direct browser-to-storage uploads

### Internationalization
- **next-intl** - i18n for Next.js App Router
- URL-based locale routing (`/en/...`, `/de/...`)
- JSON translation files in `messages/`

---

## Directory Overview

```
src/
├── app/
│   ├── [locale]/          # All internationalized routes
│   │   ├── (marketing)/   # Public pages (home, pricing)
│   │   ├── (saas)/        # SaaS infrastructure
│   │   │   ├── (auth)/    #   Authentication pages (sign-in, sign-up, etc.)
│   │   │   ├── (settings)/ #  User settings (profile, billing, security)
│   │   │   └── (admin)/   #   Admin dashboard
│   │   ├── (legal)/       # Legal pages (privacy, terms)
│   │   ├── (examples)/    # Optional example features (can delete)
│   │   └── (app)/         # [Custom] Your routes go here
│   └── api/
│       ├── (saas)/        # Template API routes
│       │   ├── auth/      #   Better Auth handler
│       │   ├── webhooks/  #   Stripe webhooks
│       │   └── upload/    #   File upload handler
│       └── app/           # [Custom] Your API routes go here
│
├── components/
│   ├── ui/                # Radix/shadcn UI primitives (Button, Card, etc.)
│   ├── shared/            # Shared infrastructure components
│   ├── marketing/         # Marketing page components
│   ├── saas/              # SaaS feature components (auth forms, billing UI, etc.)
│   ├── legal/             # Legal page components
│   └── app/               # [Custom] Your components go here
│
├── services/
│   ├── saas/              # Template services (billing, auth, etc.)
│   ├── marketing/         # Marketing services
│   └── app/               # [Custom] Your services go here
│
├── repositories/          # Generic DB operations (CRUD, queries, transactions)
│
├── lib/
│   ├── saas/              # Template SaaS infrastructure
│   │   ├── auth/          #   Authentication (server + client config)
│   │   ├── billing/       #   Billing providers
│   │   ├── stripe/        #   Stripe integration & sync
│   │   ├── email/         #   Email sending & templates
│   │   ├── notification/  #   Notification utilities
│   │   └── upload/        #   File upload utilities
│   ├── shared/            # Shared utilities
│   │   ├── utils.ts       #   General utilities (cn, etc.)
│   │   ├── format.ts      #   Formatting helpers (currency, dates)
│   │   └── ...
│   └── app/               # [Custom] Your utilities go here
│
├── db/
│   └── schema/
│       ├── saas/          # Template schemas (auth, billing tables)
│       ├── app/           # [Custom] Your database tables go here
│       └── index.ts       # Exports all schemas
│
└── i18n/                  # next-intl config (routing, request, navigation)

messages/                  # Translation files
├── en.json                # English
└── de.json                # German

public/                    # Static assets (logo, images, etc.)
docs/                      # Documentation
```

---

## Route Groups

Next.js route groups (folders in parentheses) organize routes without affecting URLs. Here's what each group is for:

| Group | URL pattern | Purpose |
|---|---|---|
| `(marketing)` | `/`, `/pricing` | Public-facing pages for visitors |
| `(saas)/(auth)` | `/sign-in`, `/sign-up`, etc. | Authentication pages |
| `(saas)/(settings)` | `/settings/profile`, etc. | Authenticated user settings |
| `(saas)/(admin)` | `/admin` | Admin-only dashboard |
| `(legal)` | `/privacy`, `/terms` | Legal pages |
| `(examples)` | `/examples/*` | Optional demo features |
| `(app)` | anything you define | **Your custom application routes** |

Each route group can have its own layout — for example, auth pages use a centered layout with no navigation, while settings pages use a sidebar layout.

---

## Where Does My Code Go?

Everything in `(app)`, `app/`, and `lib/app/` is yours. Template code lives in separate directories, which means template updates won't touch your files.

| What you're building | Where to put it |
|---|---|
| New pages / routes | `src/app/[locale]/(app)/` |
| New API endpoints | `src/app/api/app/` |
| UI components | `src/components/app/` |
| Business logic / services | `src/services/app/` |
| Database tables | `src/db/schema/app/` |
| Utility functions | `src/lib/app/` |
| Translations | `messages/en.json` and `messages/de.json` |

The `(examples)/` route group is optional — delete it once you're familiar with the patterns.

---

## Next

- [Template Boundaries](template-boundaries) — the rules for what to modify
- [Architecture](../../reference/architecture/nextjs) — deep technical details
