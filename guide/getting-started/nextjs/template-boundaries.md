---
title: "Template Boundaries - What Can You Modify?"
---

# Template Boundaries - What Can You Modify?

## Quick Reference

| Label | Meaning | Rule |
|--------|---------|------|
| `Template` | Never modify directly | Template core that receives updates |
| `Optional` | Safe to remove | Example or non-essential features |
| `Extend` | Wrap or extend | Avoid editing the original file directly |
| `Custom` | Your code | Full control - add freely |

---

## The Simple Rule

**Template code (`Template`):** Never modify directly - template updates will improve it

**Optional features (`Optional`):** Delete with `rm -rf` if you don't need them

**Your code (`Custom`):** Goes in `app/` directories - never touched by updates

---

## Directory Guide

```
src/
├── app/
│   ├── [locale]/
│   │   ├── (marketing)/    [Template] Public pages
│   │   │   ├── page.tsx    # Home
│   │   │   └── pricing/    # Pricing
│   │   ├── (saas)/         [Template] SaaS infrastructure
│   │   │   ├── (auth)/     # Authentication
│   │   │   ├── (settings)/ # User settings
│   │   │   └── (admin)/    # Admin dashboard
│   │   ├── (legal)/        [Template] Legal pages
│   │   ├── (examples)/     [Optional] Delete after learning
│   │   └── (app)/          [Custom] Your routes
│   └── api/
│       ├── (saas)/         [Template] SaaS APIs
│       │   ├── auth/       # → /api/auth
│       │   ├── webhooks/   # → /api/webhooks
│       │   └── upload/     # → /api/upload
│       └── app/            [Custom] Your APIs
│
├── components/
│   ├── ui/                 [Template] Primitives
│   ├── shared/             [Template] Shared infrastructure
│   │   ├── icons/
│   │   ├── brand/
│   │   ├── theme/
│   │   ├── locale/
│   │   └── providers.tsx
│   ├── marketing/          [Template] Marketing components
│   │   ├── navigation/
│   │   └── footer/
│   ├── saas/               [Template] SaaS components
│   │   ├── auth/
│   │   ├── billing/
│   │   ├── settings/
│   │   ├── notifications/
│   │   └── email-templates/
│   ├── legal/              [Template] Legal components
│   └── app/                [Custom] Your components
│
├── services/
│   ├── saas/               [Template] Core services
│   │   ├── auth/           # Authentication services
│   │   ├── billing/        # Billing services
│   │   ├── plan/           # Plan services
│   │   └── notifications/  # Notification services
│   ├── marketing/          [Template] Marketing services (if needed)
│   └── app/                [Custom] Your services
│
├── db/schema/
│   ├── saas/               [Template] Core schemas
│   │   ├── auth.ts         # Auth tables
│   │   ├── billing.ts      # Billing tables
│   │   └── index.ts        # Export all saas schemas
│   ├── marketing/          [Template] Marketing schemas (if needed)
│   │   └── index.ts
│   ├── app/                [Custom] Your schemas
│   │   └── index.ts
│   └── index.ts            Modified - Exports all
│
└── lib/
    ├── saas/               [Template] SaaS infrastructure
    │   ├── auth/           # Authentication (server + client)
    │   ├── billing/        # Billing providers
    │   ├── stripe/         # Stripe integration & webhook sync
    │   ├── email/          # Email sending & translations
    │   ├── notification/   # Notification utilities
    │   └── upload/         # File upload utilities
    ├── shared/             [Template] Shared utilities
    │   ├── utils.ts        # General utilities (cn, etc.)
    │   ├── format.ts       # Currency/number formatting
    │   ├── id.ts           # ID generation
    │   ├── ui/             # Shadcn UI utilities
    │   ├── date/           # Date formatting & utilities
    │   ├── seo/            # SEO metadata generation
    │   └── locale/         # i18n constants
    └── app/                [Custom] Your utilities
```

---

## Deleting Optional Features

### Don't need admin dashboard?

```bash
rm -rf src/app/[locale]/\(saas\)/\(admin\)
```

### Don't need examples?

```bash
rm -rf src/app/[locale]/\(examples\)
```

That's it. No config, no flags. Just delete.

---

## Extending Template Code

### Pattern: Wrapping Components

```typescript
// Avoid: modifying a template file
// src/components/ui/button.tsx

// Preferred: create a wrapper in your app code
// src/components/app/brand-button.tsx
import { Button } from "@/components/ui/button";

export function BrandButton(props) {
  return <Button className="brand-styles" {...props} />;
}
```

### Pattern: Using Template Services

```typescript
// Recommended: use template services from your app code
// src/services/app/project/project.service.ts
import { canSendEmail } from "@/services/notifications/notifications.service";

export async function createProject(userId, data) {
  const project = await save(data);

  if (await canSendEmail(userId, "productUpdates")) {
    await notify(project);
  }

  return project;
}
```

## Using Template Utilities

```typescript
// Recommended: import template utilities instead of duplicating them
// src/services/app/project/project.service.ts
import { auth } from "@/lib/saas/auth/server";
import { cn } from "@/lib/shared/utils";
import { formatCurrency } from "@/lib/shared/format";

export async function getProjectStats(userId: string) {
  const session = await auth.api.getSession({ headers: await headers() });

  const stats = await calculateStats(userId);

  return {
    total: formatCurrency(stats.revenue, "USD"),
    // ... your custom logic
  };
}
```

## Creating Custom Utilities

```typescript
// Recommended: create your own utilities in lib/app
// src/lib/app/analytics.ts
import { formatDate } from "@/lib/shared/date";

export function trackEvent(name: string, data: Record<string, unknown>) {
  // Your custom analytics logic
}
```

---

## Template Updates

### With This Structure

```bash
git fetch template
git merge template/master

# Conflicts only in:
# - package.json (merge deps)
# - messages/*.json (merge translations)
# Total: 2-3 files

# Your app/ code: untouched
```

### Why It Works

Template updates modify:
- `[locale]/(marketing)/`, `[locale]/(saas)/`, `[locale]/(legal)/` (`Template`)
- `api/(saas)/` (`Template`)
- `components/marketing/`, `components/saas/`, `components/legal/` (`Template`)
- `services/saas/`, `services/marketing/` (`Template`)

Your code lives in:
- `[locale]/(app)/` (`Custom`)
- `api/app/` (`Custom`)
- `components/app/` (`Custom`)
- `services/app/` (`Custom`)

Different directories = no conflicts.

**Bonus:** The semantic grouping (`marketing`, `saas`, `legal`) mirrors the route structure, making it intuitive to find related code. Works consistently across routes, APIs, and components!

---

## Why This Structure Exists

The `saas/` vs `app/` split was designed specifically to make template updates conflict-free.

### The Problem It Solves

Before this structure, buyers who modified template files would encounter merge conflicts every time a template update arrived. A typical update would conflict in 20+ files:

```
git merge template/master
# Conflicts in 20+ files:
# - src/app/dashboard/page.tsx (user modified)
# - src/components/project-card.tsx (user added)
# - src/services/project.service.ts (user added)
```

This made it impractical to stay current with template improvements — users had to choose between staying up-to-date and keeping their customizations.

### How the Separation Works

The directory structure creates clear ownership boundaries:

- **`saas/` directories** — template code that receives updates. The template author controls these files and pushes improvements over time.
- **`app/` directories** — your code that never gets touched by template updates. Everything you build lives here and is completely isolated from template changes.

The key insight: Next.js route groups like `(app)` don't affect URLs (`/app/projects` renders at `/projects`), so you get clean separation in the file tree without any URL impact.

### The Merge Safety Guarantee

When template updates arrive, only 2-3 files conflict:

```
git merge template/master
# Conflicts only in:
# - package.json (dependencies added)
# - messages/*.json (new translation keys)
# Your app/ code: untouched
```

This is an 85-90% reduction in merge conflicts compared to a mixed-code structure. Your `(app)/` routes, `components/app/` components, and `services/app/` services are in completely separate directories from anything the template touches — so git never sees a conflict there.

---

## Emergency: I Already Modified Template

If you modified template files:

1. **Extract your changes:**
   ```bash
   # Copy your custom code
   # Create new file in app/
   # Paste your code there
   ```

2. **Revert template file:**
   ```bash
   git checkout template/main -- path/to/template/file.tsx
   ```

3. **Update imports:**
   ```bash
   # Change imports to point to your new app/ file
   ```

---

## Related Docs

- [Adding Features](../../customizing/adding-features/nextjs)
- [Updating](../../customizing/updating)
- [Architecture](../../reference/architecture/nextjs)

---

**Last Updated:** February 3, 2026
