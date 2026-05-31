---
title: "Next.js"
---

# Architecture

For a quick overview of the tech stack and directory layout, see [Project Structure](../../getting-started/nextjs/project-structure).

This document covers the deep technical details: data flow patterns, route organization, key architecture decisions, environment variables, and deployment.

---

## Route Organization

### Next.js Directory Structure

```
src/app/
├── [locale]/              # Internationalized routes
│   ├── (marketing)/       # [Template] Public pages (home, pricing)
│   ├── (saas)/            # [Template] SaaS infrastructure
│   │   ├── (auth)/        #    Authentication pages
│   │   ├── (settings)/    #    User settings pages
│   │   └── (admin)/       #    Admin panel (optional, can delete)
│   ├── (legal)/           # [Template] Legal pages
│   ├── (examples)/        # [Optional] Example features (can delete)
│   └── (app)/             # [Custom] Your application routes
└── api/
    ├── (saas)/            # [Template] SaaS API routes
    │   ├── auth/          #    Better Auth endpoints
    │   ├── webhooks/      #    Stripe webhooks
    │   └── upload/        #    File upload endpoints
    └── app/               # [Custom] Your API endpoints
```

**Legend:** `Template` = never modify directly | `Optional` = safe to remove | `Custom` = add your code here

### Route Groups (Next.js)

Next.js route groups (using `()` parentheses) organize routes without affecting URLs.

#### Template Route Groups (Do Not Modify)

**`(marketing)/` - Public Pages**
- `/` - Homepage
- `/pricing` - Pricing page

**`(saas)/(auth)/` - Authentication Pages**
- `/sign-in` - User sign-in
- `/sign-up` - User registration
- `/forgot-password` - Password recovery
- `/reset-password` - Password reset
- `/verify-email` - Email verification
- `/verify-2fa` - Two-factor authentication

**`(saas)/(settings)/` - User Settings**
- `/settings/profile` - Profile management
- `/settings/security` - Security settings
- `/settings/billing` - Billing and subscriptions
- `/settings/notification` - Notification preferences

**`(saas)/(admin)/` - Optional Admin Panel**
- Can be deleted if not needed
- Separated from main application routes

**`(legal)/` - Legal Pages**
- `/privacy`, `/terms`, `/cookies`

Each route group carries its own layout — auth pages use a centered form layout with no navigation header, settings pages use a sidebar layout, and marketing pages use the full navigation header.

#### Custom Route Group (Your Code)

**`(app)/` - Your Application Routes**

All custom routes belong here:

```
src/app/[locale]/(app)/
├── dashboard/
│   └── page.tsx           # /dashboard
├── projects/
│   ├── page.tsx           # /projects
│   ├── [id]/
│   │   └── page.tsx       # /projects/[id]
│   └── new/
│       └── page.tsx       # /projects/new
└── settings/
    ├── page.tsx           # /settings (app-specific settings)
    └── team/
        └── page.tsx       # /settings/team
```

**`api/app/` - Your API Endpoints**

```
src/app/api/app/
├── projects/
│   └── route.ts           # POST /api/app/projects
├── teams/
│   └── route.ts           # GET/POST /api/app/teams
└── analytics/
    └── route.ts           # GET /api/app/analytics
```

### URL Mapping

Route groups don't appear in URLs:

| File Path | URL |
|---|---|
| `(saas)/(auth)/sign-in/page.tsx` | `/sign-in` |
| `(marketing)/pricing/page.tsx` | `/pricing` |
| `(app)/dashboard/page.tsx` | `/dashboard` |
| `(app)/projects/[id]/page.tsx` | `/projects/123` |
| `api/app/teams/route.ts` | `/api/app/teams` |

### Internationalization of Routes

All page routes are wrapped in `[locale]/` for internationalization:

- English: `/en/dashboard`
- German: `/de/dashboard`

API routes are not localized — `/api/app/teams` is the same for all locales.

Middleware handles locale detection and redirection automatically. Locale-prefixed paths are managed via `@/i18n/navigation` — never `next/navigation` directly. See [conventions.md](../../customizing/conventions#navigation-links) for why this matters.

### Route Protection

Template settings routes are automatically protected by the settings layout. For your custom routes, add protection at the layout level.

**Next.js** (`src/app/[locale]/(app)/dashboard/layout.tsx`):
```typescript
import { auth } from "@/lib/saas/auth/server";
import { headers } from "next/headers";
import { redirect } from "@/i18n/navigation";

export default async function DashboardLayout({ children }: { children: React.ReactNode }) {
  const session = await auth.api.getSession({ headers: await headers() });
  if (!session?.user) redirect("/sign-in");
  return <>{children}</>;
}
```

### Rules for Routes

**Never modify template route groups.** Template routes receive updates; modifying them creates merge conflicts.

```typescript
// Avoid: editing a template route
// src/app/[locale]/(saas)/(settings)/settings/billing/page.tsx

// Preferred: create a custom route in (app)
// src/app/[locale]/(app)/custom-billing/page.tsx
```

**Keep all custom routes in `(app)/`.** Never create new route groups at the root level of `[locale]/`.

### Template API Routes

**`api/(saas)/auth/[...all]/` - Better Auth**
- Handles all authentication endpoints
- Configured via Better Auth

**`api/(saas)/webhooks/` - Stripe Integration**
- `stripe/route.ts` - Stripe webhook handler

**`api/(saas)/upload/` - File Uploads**
- `avatar/route.ts` - Avatar upload endpoint

Custom API endpoints follow RESTful conventions.

For read queries and mutations, prefer `createServerFn` over API routes in both frameworks — see [services.md](../services#3-server-functions). Use API routes when you need a stable HTTP endpoint (webhooks, third-party integrations).

**Next.js** (`src/app/api/app/projects/route.ts`):
```typescript
import { NextResponse } from "next/server";
import { auth } from "@/lib/saas/auth/server";
import { headers } from "next/headers";

export async function GET() {
  const session = await auth.api.getSession({ headers: await headers() });
  if (!session?.user) return NextResponse.json({ error: "UNAUTHORIZED" }, { status: 401 });
  return NextResponse.json({ projects: [] });
}
```

---

## Data Flow Patterns

### Authentication Flow

```
Client Component
  ↓ (useSession hook)
authClient.useSession()
  ↓ (cached)
Better Auth Cookie
  ↓ (if expired)
Database Query

Server Component
  ↓ (getSession)
auth.api.getSession({ headers })
  ↓ (cached)
Better Auth Cookie
  ↓ (if expired)
Database Query
```

### File Upload Flow

```
User selects file
  ↓
useUploadFile hook (better-upload)
  ↓
POST /api/upload/avatar
  ↓ (onBeforeUpload)
Authentication check
Custom file naming
  ↓
Pre-signed URL generation
  ↓ (direct upload)
Browser → Cloudflare R2
  ↓ (onAfterSignedUrl)
Save URL to database
Update user.image field
  ↓
Refresh session cache
  ↓
UI updates automatically
```

### i18n Translation Flow

**Next.js (next-intl) — runtime:**
```
Component needs text
  ↓
useTranslations("Namespace")        ← hook call at render time
  ↓
Load messages/{locale}/*.json       ← loaded per request
  ↓
Return translated string
```

next-intl loads translations at request time per locale.

---

## Key Architecture Decisions

### Why Better Auth?

- Modern, type-safe authentication
- Built-in email verification and password reset
- Easy OAuth integration
- Automatic session management and caching
- No complex setup required

### Why Drizzle ORM?

- Type-safe SQL queries
- Lightweight and performant
- Great TypeScript integration
- Easy migrations
- Direct SQL when needed

### Why next-intl?

- First-class Next.js App Router support
- Type-safe translations
- Locale routing built-in
- Server and client component support

### Why Cloudflare R2?

- S3-compatible API (easy migration path)
- Zero egress fees compared to S3
- Fast global CDN
- Competitive pricing
- Works with better-upload helpers

### Why Radix UI?

- Headless (full styling control)
- Accessibility built-in (WCAG)
- Composable primitives
- Works perfectly with Tailwind
- Battle-tested components

---

## Environment Variables

### Required

```bash
DATABASE_URL=postgresql://user:password@localhost:5432/dbname
BETTER_AUTH_SECRET=your-secret-key-here
BETTER_AUTH_URL=http://localhost:3000
```

### Email (Optional)

```bash
RESEND_API_KEY=your_resend_api_key
FROM_EMAIL=hello@yourdomain.com
NEXT_PUBLIC_APP_NAME=YourApp
```

### Storage (Optional)

```bash
# Cloudflare R2
CLOUDFLARE_ACCOUNT_ID=your-account-id
STORAGE_BUCKET=your-bucket-name
STORAGE_ACCESS_KEY_ID=your-access-key
STORAGE_SECRET_ACCESS_KEY=your-secret-key
R2_PUBLIC_URL_DOMAIN=https://pub-xxx.r2.dev

# Or custom domain
STORAGE_PUBLIC_URL=https://cdn.yourdomain.com
```

### OAuth (Optional)

```bash
GOOGLE_CLIENT_ID=your-google-client-id
GOOGLE_CLIENT_SECRET=your-google-client-secret
GITHUB_CLIENT_ID=your-github-client-id
GITHUB_CLIENT_SECRET=your-github-client-secret
```

> **Build-time vs. runtime**: Variables prefixed with `NEXT_PUBLIC_` are baked into the client bundle at build time and cannot be overridden at runtime. Set them correctly in `.env.production` before building. See the [Billing](../billing) and [Stripe Setup](../../integrations/stripe) docs for deployment-specific guidance.

---

## Performance Considerations

### Server Components by Default

- Use server components when possible
- Client components only when needed (interactivity, hooks)
- Reduces JavaScript bundle size
- Better SEO and initial page load

### Better Auth Caching

- Session automatically cached
- Multiple `getSession()` calls don't hit database
- Cache invalidation handled automatically
- No manual cache management needed

### Direct-to-Storage Uploads

- Files upload directly to R2/S3
- No server bandwidth usage
- Faster uploads for users
- Lower server costs

### Translation Loading

- Translations loaded per-route
- No unnecessary bundle bloat
- Split by namespace
- Cached by Next.js

---

## Development Workflow

### Local Development

```bash
# 1. Start database
docker compose up -d

# 2. Install dependencies
pnpm install

# 3. Run migrations
pnpm db:push

# 4. Start dev server
pnpm dev
```

### Database Changes

```bash
# Update schema in src/db/schema/app/
# Then push changes
pnpm db:push

# Or generate migration SQL
pnpm db:generate
```

### Adding Features

1. Update database schema if needed (`src/db/schema/app/`)
2. Add translation keys to `messages/en.json` and `messages/de.json`
3. Create components in `src/components/app/`
4. Add routes in `src/app/[locale]/(app)/`
5. TypeScript will surface any type errors

See [Adding Features](../../customizing/adding-features/nextjs) for a step-by-step guide.

---

## Deployment

### Application Hosting

- **Vercel** (recommended) - Native Next.js support, automatic deployments, edge middleware, built-in analytics
- **Dokploy / Docker** - Self-hosted option; note that `NEXT_PUBLIC_*` vars must be set before the build, not at runtime

### Database

- Vercel Postgres
- Supabase
- Railway
- Neon

### Storage

- Cloudflare R2 (recommended — zero egress fees)
- AWS S3 + CloudFront
- Vercel Blob

### Email

- Resend (recommended)
- SendGrid
- AWS SES

---

## See Also

- [Project Structure](../../getting-started/nextjs/project-structure)
- [Template Boundaries](../../getting-started/nextjs/template-boundaries)
- [Adding Features](../../customizing/adding-features/nextjs) - Step-by-step guide for adding custom features
- [Billing](../billing) - Billing architecture and Stripe integration
- [Database](../database) - Schema conventions and query patterns
- [Upload System](../../integrations/storage) - File upload architecture
- [Email System](../../integrations/email) - Email templates and delivery
- [TanStack Architecture](./tanstack) - TanStack Start version of this document
