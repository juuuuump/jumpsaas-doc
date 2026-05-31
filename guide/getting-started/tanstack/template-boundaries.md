---
title: "Template Boundaries"
---

# Template Boundaries

Understanding what's safe to modify versus what belongs to the template.

JumpSaaS is designed to be a living foundation — not a library you install, but a codebase you own. That means everything is editable. The distinction between "Template" and "Custom" isn't about what you're *allowed* to touch; it's about what will create **merge conflicts** when you pull future template updates.

The golden rule: **modify `saas/` directories and you'll have conflicts. Modify `app/` directories and you won't.**

---

## The Map

### Your Code (safe to modify freely)

These directories are reserved for your product. The template never writes here:

| Directory | Purpose |
|---|---|
| `src/routes/{-$locale}/_app/app/` | Your main app pages |
| `src/routes/{-$locale}/_protected/` (new subdirs) | Your protected pages (settings/admin are template) |
| `src/routes/api/` (new files) | Your API endpoints |
| `src/components/app/` | Your UI components |
| `src/services/app/` | Your business logic |
| `src/server/app/` | Your server functions |
| `src/db/schema/app/` | Your database tables |
| `src/lib/app/` | Your utility functions |
| `messages/en.json`, `messages/de.json` | Your translations (add keys freely) |

When adding new pages, put them inside `_app/` or create new subdirectories inside `_protected/`. When adding new API routes, create new files in `routes/api/` — the existing `auth/`, `upload.avatar.ts`, and `webhooks.stripe.ts` files are template-owned.

### Template Code (avoid modifying directly)

Modifying these files will cause conflicts on template updates. If you need different behaviour, see [Extending Template Code](#extending-template-code) below.

**Routes:**
- `src/routes/{-$locale}/__root.tsx`
- `src/routes/{-$locale}/route.tsx`
- `src/routes/{-$locale}/index.tsx`
- `src/routes/{-$locale}/pricing.tsx`
- `src/routes/{-$locale}/legal/`
- `src/routes/{-$locale}/_auth/` (all auth pages)
- `src/routes/{-$locale}/_protected/settings/`
- `src/routes/{-$locale}/_app/route.tsx` (layout + guard)
- `src/routes/api/auth/`
- `src/routes/api/upload.avatar.ts`
- `src/routes/api/webhooks.stripe.ts`

**Components, services, lib, server:**
- `src/components/saas/`
- `src/components/marketing/`
- `src/components/shared/`
- `src/components/ui/`
- `src/services/saas/`
- `src/lib/saas/`
- `src/server/saas/`
- `src/db/schema/saas/`
- `src/i18n/`
- `src/paraglide/` (generated — never edit manually)

### Optional (can delete)

- `src/routes/{-$locale}/_protected/admin/` — the admin dashboard. Delete it if you don't want it. Remove the link from your nav and it's gone.

---

## Extending Template Code

Sometimes you need to change something in the template — different redirect behaviour, extra fields on a settings page, custom auth logic. The pattern is always the same: **wrap or extend at the boundary, don't edit the source**.

### Adding fields to a settings page

Create a new route file alongside the existing one (e.g., `_protected/settings/my-settings.tsx`) rather than modifying `_protected/settings/profile.tsx`. Add a link to it from your nav.

If you genuinely need to change the profile page itself, accept that it'll be a conflict surface on updates, fork it into your own copy, and remove the template version.

### Custom route guards

Route guards in TanStack Start run in `beforeLoad`. The template's `_auth/route.tsx` and `_protected/route.tsx` each export a `beforeLoad` that checks the session. If you need additional guard logic — e.g., checking a user role — add a nested layout route inside the existing one rather than modifying the template's `route.tsx`:

```ts
// src/routes/{-$locale}/_protected/admin-only/route.tsx
export const Route = createFileRoute("/{-$locale}/_protected/admin-only")({
  beforeLoad: async () => {
    const session = await getSession();
    if (session?.user.role !== "admin") {
      throw redirect({ href: "/app" });
    }
  },
  component: () => <Outlet />,
});
```

### Custom server functions

Server functions in TanStack Start use `createServerFn()` from `@tanstack/react-start`. The template's server functions live in `src/server/saas/`. Add yours in `src/server/app/`:

```ts
// src/server/app/my-feature.ts
import { createServerFn } from "@tanstack/react-start";
import { getSession } from "#/server/saas/auth";

export const myServerFn = createServerFn().handler(async () => {
  const session = await getSession();
  if (!session) throw new Error("Unauthorized");
  // your logic here
});
```

Import `getSession` from `#/server/saas/auth` — it's the canonical way to access the current session in any server function.

### Overriding a UI component

If you want a different version of a template component, copy it into `src/components/app/` and import from there. Don't modify the original in `components/saas/` or `components/ui/`.

### Adding database tables

Add new Drizzle schema files to `src/db/schema/app/` and re-export them from `src/db/schema/index.ts`. The template schemas in `src/db/schema/saas/` define the auth and billing tables — leave them as-is unless you're doing something very deliberate.

---

## On the Marketing Site

The landing page (`index.tsx`), pricing page (`pricing.tsx`), and legal pages are template-owned because they contain the default copy and structure. In practice, you'll want to customise the landing page heavily.

The recommended approach:

1. Accept the landing page as a conflict surface and edit it directly — the copy is yours to change.
2. Pull template updates carefully, resolving conflicts in those files by hand.

Alternatively, treat the default landing page as a starting point, copy the components you want into `components/app/`, and build your own `index.tsx` on top of it. The template won't know the difference.

---

## Pulling Template Updates

When you run `git fetch template && git merge template/main`, conflicts will be in the template-owned files listed above. The merge is usually straightforward — the template tries to keep its internal API stable and avoids unnecessary churn in these files.

See [Updating](../../customizing/updating) for the full update workflow and conflict resolution strategy.
