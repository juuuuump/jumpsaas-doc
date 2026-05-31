---
title: "TanStack"
---

# Architecture

For a quick overview of the tech stack and directory layout, see [Project Structure](../../getting-started/tanstack/project-structure).

This document covers the deep technical details: data flow patterns, route organization, key architecture decisions, environment variables, and deployment.

---

## Route Organization

### TanStack Directory Structure

```
src/routes/
├── {-$locale}/            # Optional locale prefix (e.g. /en/, /de/)
│   ├── _marketing/        # [Template] Public pages (home, pricing)
│   ├── _saas/             # [Template] SaaS infrastructure
│   │   ├── _auth/         #    Authentication pages
│   │   └── _settings/     #    User settings pages
│   ├── _legal/            # [Template] Legal pages
│   └── _app/              # [Custom] Your application routes
│       └── app/
└── api/
    ├── auth/              # Better Auth endpoints
    ├── webhooks/          # Stripe webhooks
    └── upload/            # File upload endpoints

src/server/                # Server functions (replaces Next.js API routes for mutations)
├── saas/                  # [Template] SaaS server functions
└── app/                   # [Custom] Your server functions
```

**Legend:** `Template` = never modify directly | `Optional` = safe to remove | `Custom` = add your code here

### URL Mapping

| File Path | URL |
|---|---|
| `(saas)/(auth)/sign-in/page.tsx` | `/sign-in` |
| `(marketing)/pricing/page.tsx` | `/pricing` |
| `(app)/dashboard/page.tsx` | `/dashboard` |
| `(app)/projects/[id]/page.tsx` | `/projects/123` |
| `api/app/teams/route.ts` | `/api/app/teams` |

### Internationalization of Routes

All page routes are wrapped in `{-$locale}/` for internationalization:

- English: `/en/dashboard`
- German: `/de/dashboard`

API routes are not localized — `/api/app/teams` is the same for all locales.

Locale is an optional route param (`{-$locale}`), not middleware. TanStack Router handles it natively — use `useParams` to read it and pass it when constructing links.

### Route Protection

Template settings routes are automatically protected by the settings layout. For your custom routes, add protection at the layout level.

**TanStack** (`src/routes/{-$locale}/_app.tsx` — the layout route):
```typescript
import { createFileRoute, redirect } from "@tanstack/react-router";
import { getWebRequest } from "@tanstack/react-start/server";
import { auth } from "@/lib/saas/auth/server";

export const Route = createFileRoute("/{-$locale}/_app")({
  beforeLoad: async () => {
    const request = getWebRequest();
    const session = await auth.api.getSession({ headers: request.headers });
    if (!session?.user) throw redirect({ to: "/$locale/sign-in" });
  },
});
```

### Rules for Routes

**Never modify template route groups.** Template routes receive updates; modifying them creates merge conflicts.

**Keep all custom routes in `_app/`.** Never create new layout routes at the root level of `{-$locale}/`.

### Template API Routes

**`api/auth/` - Better Auth**
- Handles all authentication endpoints
- Configured via Better Auth

**`api/webhooks/` - Stripe Integration**
- Stripe webhook handler

**`api/upload/` - File Uploads**
- Avatar upload endpoint

Custom API endpoints follow RESTful conventions.

For read queries and mutations, prefer `createServerFn` over API routes in both frameworks — see [services.md](../services#3-server-functions). Use API routes when you need a stable HTTP endpoint (webhooks, third-party integrations).

**TanStack** (`src/routes/api/app/projects.ts`):
```typescript
import { createAPIFileRoute } from "@tanstack/react-start/api";
import { getWebRequest } from "@tanstack/react-start/server";
import { json } from "@tanstack/react-start";
import { auth } from "@/lib/saas/auth/server";

export const APIRoute = createAPIFileRoute("/api/app/projects")({
  GET: async () => {
    const request = getWebRequest();
    const session = await auth.api.getSession({ headers: request.headers });
    if (!session?.user) return json({ error: "UNAUTHORIZED" }, { status: 401 });
    return json({ projects: [] });
  },
});
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

**TanStack (ParaglideJS) — compile-time:**
```
messages/{locale}.json edited
  ↓
pnpm build / pnpm dev               ← Paraglide generates typed functions
  ↓
import * as m from "@/paraglide/messages"
  ↓
m.namespace_key()                   ← plain function call, no hook
  ↓
Render in UI (zero runtime overhead)
```

ParaglideJS generates a separate bundle per locale at build time — there is no runtime lookup.

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

### Why ParaglideJS?

- Compile-time message extraction — zero runtime i18n overhead
- Generates typed message functions from source JSON
- Integrates with TanStack Router's param-based locale routing
- Inlang ecosystem tooling (Sherlock IDE extension, lint rules)

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

---

## Performance Considerations

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

- ParaglideJS generates per-locale bundles at build time
- Zero runtime i18n overhead
- No per-request translation loading

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
4. Add routes in `src/routes/{-$locale}/_app/`
5. TypeScript will surface any type errors

See [Adding Features](../../customizing/adding-features/tanstack) for a step-by-step guide.

---

## Deployment

### Application Hosting

- **Vercel** (recommended) - Automatic deployments, edge functions, built-in analytics
- **Dokploy / Docker** - Self-hosted option

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

- [Project Structure](../../getting-started/tanstack/project-structure)
- [Template Boundaries](../../getting-started/tanstack/template-boundaries)
- [Adding Features](../../customizing/adding-features/tanstack) - Step-by-step guide for adding custom features
- [Billing](../billing) - Billing architecture and Stripe integration
- [Database](../database) - Schema conventions and query patterns
- [Upload System](../../integrations/storage) - File upload architecture
- [Email System](../../integrations/email) - Email templates and delivery
- [Next.js Architecture](./nextjs) - Next.js version of this document
