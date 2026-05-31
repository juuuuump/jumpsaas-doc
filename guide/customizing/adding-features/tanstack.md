---
title: "TanStack"
---

# Adding Custom Features

This guide explains how to add your own features to JumpSaaS while keeping them separate from template code. This makes template updates easier and your codebase cleaner.

---

## Quick Start

**Golden Rule:** Keep your custom code in designated areas, separate from template code.

**Where to put your code:**

- Routes: `src/routes/{-$locale}/_app/`
- Components: `src/components/app/`
- Services: `src/services/app/`
- Server functions: `src/server/app/`
- Database: `src/db/schema/app.ts`

---

## Project Structure

### Template vs Custom Code

```
src/
├── routes/{-$locale}/
│   ├── _auth/           🔒 Template - Don't modify
│   ├── _settings/       🔒 Template - Don't modify
│   └── _app/            ✅ YOUR ROUTES HERE
│       └── app/
│           ├── dashboard.tsx
│           └── projects.tsx
├── server/
│   ├── saas/            🔒 Template - Don't modify
│   └── app/             ✅ YOUR SERVER FUNCTIONS HERE
├── components/
│   ├── ui/              🟡 Template - Can extend
│   ├── saas/            🔒 Template - Don't modify
│   └── app/             ✅ YOUR COMPONENTS HERE
├── services/
│   ├── saas/            🔒 Template - Don't modify
│   └── app/             ✅ YOUR SERVICES HERE
└── db/schema/
    ├── auth.ts          🔒 Don't modify
    ├── billing.ts       🔒 Don't modify
    └── app.ts           ✅ YOUR SCHEMAS HERE
```

**Legend:** 🔒 Never modify | 🟡 Can modify with care | ✅ Your code goes here

---

## Step-by-Step: Adding a New Feature

### Example: Adding a "Projects" Feature

#### 1. Create Database Schema

**File:** `src/db/schema/app.ts`

```typescript
import { pgTable, text, timestamp, uuid, boolean } from "drizzle-orm/pg-core";
import { user } from "./auth"; // Import template schema

export const project = pgTable("project", {
  id: uuid("id").defaultRandom().primaryKey(),
  userId: uuid("user_id")
    .notNull()
    .references(() => user.id, { onDelete: "cascade" }),
  name: text("name").notNull(),
  description: text("description"),
  isActive: boolean("is_active").default(true).notNull(),
  createdAt: timestamp("created_at").defaultNow().notNull(),
  updatedAt: timestamp("updated_at").defaultNow().notNull(),
});

export const task = pgTable("task", {
  id: uuid("id").defaultRandom().primaryKey(),
  projectId: uuid("project_id")
    .notNull()
    .references(() => project.id, { onDelete: "cascade" }),
  title: text("title").notNull(),
  completed: boolean("completed").default(false).notNull(),
  createdAt: timestamp("created_at").defaultNow().notNull(),
});
```

**File:** `src/db/index.ts` (add your exports)

```typescript
// Template schemas (already there)
export * from "./schema/auth";
export * from "./schema/billing";

// [YOURS] Add your custom schemas
export * from "./schema/app";
```

#### 2. Push Schema to Database

```bash
npm run db:push
```

#### 3. Create Service Layer

**File:** `src/services/app/project/project.service.ts`

```typescript
import { db } from "@/db";
import { project, task } from "@/db/schema/app";
import { eq, and, desc } from "drizzle-orm";

export type Project = typeof project.$inferSelect;
export type Task = typeof task.$inferSelect;

export async function getProjects(userId: string) {
  return await db.query.project.findMany({
    where: eq(project.userId, userId),
    with: {
      tasks: true,
    },
    orderBy: [desc(project.createdAt)],
  });
}

export async function createProject(userId: string, data: {
  name: string;
  description?: string;
}) {
  const [newProject] = await db
    .insert(project)
    .values({
      userId,
      name: data.name,
      description: data.description,
    })
    .returning();

  return newProject;
}

export async function deleteProject(projectId: string, userId: string) {
  await db
    .delete(project)
    .where(and(eq(project.id, projectId), eq(project.userId, userId)));
}
```

#### 4. Create Server Functions

Server functions are the layer between your UI and service layer.

**Server Functions** (`src/server/app/projects.ts`):

```typescript
import { createServerFn } from "@tanstack/react-start";
import { getWebRequest } from "@tanstack/react-start/server";
import { auth } from "@/lib/saas/auth/server";
import {
  getProjects,
  createProject as createProjectService,
} from "@/services/app/project/project.service";

export const getMyProjects = createServerFn().handler(async () => {
  const request = getWebRequest();
  const session = await auth.api.getSession({ headers: request.headers });
  if (!session?.user) return { error: "UNAUTHORIZED" as const };

  try {
    return { data: await getProjects(session.user.id) };
  } catch (error) {
    console.error("Failed to fetch projects:", error);
    return { error: "FETCH_FAILED" as const };
  }
});

export const createProject = createServerFn()
  .validator((data: { name: string; description?: string }) => data)
  .handler(async ({ data }) => {
    const request = getWebRequest();
    const session = await auth.api.getSession({ headers: request.headers });
    if (!session?.user) return { error: "UNAUTHORIZED" as const };
    if (!data.name?.trim()) return { error: "NAME_REQUIRED" as const };

    try {
      return { data: await createProjectService(session.user.id, data) };
    } catch (error) {
      console.error("Failed to create project:", error);
      return { error: "CREATE_FAILED" as const };
    }
  });
```

#### 5. Create Page Components

**File:** `src/routes/{-$locale}/_app/app/projects.tsx`

```typescript
import { createFileRoute } from "@tanstack/react-router";
import { getMyProjects } from "@/server/app/projects";
import { ProjectList } from "@/components/app/projects/project-list";

export const Route = createFileRoute("/{-$locale}/_app/app/projects")({
  loader: () => getMyProjects(),
  component: ProjectsPage,
});

function ProjectsPage() {
  const result = Route.useLoaderData();
  if (result.error) return <div>Error loading projects</div>;
  return (
    <div className="container py-8">
      <h1 className="text-3xl font-bold">My Projects</h1>
      <ProjectList projects={result.data || []} />
    </div>
  );
}
```

**File:** `src/components/app/projects/project-list.tsx`

```typescript
"use client";

import { Card } from "@/components/ui/card";
import { Button } from "@/components/ui/button";
import type { Project } from "@/services/app/project/project.service";

interface ProjectListProps {
  projects: Project[];
}

export function ProjectList({ projects }: ProjectListProps) {
  return (
    <div className="grid gap-4 md:grid-cols-2 lg:grid-cols-3">
      {projects.map((project) => (
        <Card key={project.id} className="p-4">
          <h3 className="font-semibold">{project.name}</h3>
          <p className="text-sm text-muted-foreground">
            {project.description}
          </p>
          <Button variant="outline" size="sm" className="mt-4">
            View Project
          </Button>
        </Card>
      ))}
    </div>
  );
}
```

#### 6. Add Navigation

**File:** `src/components/app/app-nav.tsx` (create this)

```typescript
import { Link } from "@tanstack/react-router"; // locale param handled by router

export function AppNav() {
  return (
    <nav className="flex gap-4">
      <Link to="/$locale/app/dashboard" params={{ locale: "en" }}>Dashboard</Link>
      <Link to="/$locale/app/projects" params={{ locale: "en" }}>Projects</Link>
    </nav>
  );
}
```

#### 7. Add Translations

**File:** `messages/en.json`

```json
{
  "app": {
    "projects": {
      "title": "My Projects",
      "createNew": "Create Project",
      "noProjects": "No projects yet",
      "errors": {
        "UNAUTHORIZED": "You must be logged in",
        "NAME_REQUIRED": "Project name is required",
        "CREATE_FAILED": "Failed to create project",
        "FETCH_FAILED": "Failed to load projects"
      }
    }
  }
}
```

---

## The Three-Layer Pattern: Repository → Service → Action

Every feature should follow the three-layer pattern to keep database access, business logic, and request handling cleanly separated:

| Layer | File | Responsibility |
|-------|------|----------------|
| Repository | `{domain}.repository.ts` | All raw `db.*` calls — no exceptions |
| Service | `{feature}.service.ts` | Business logic, computed fields, upsert patterns |
| Action | `actions.ts` | Auth check, input validation, error handling |

### Worked Example: Notification Preferences

The notification preferences feature demonstrates the full three-layer pattern end-to-end. It lives in `src/services/saas/notifications/` and manages per-user email notification opt-ins across four categories (billing, security, product updates, marketing).

**Repository** (`preferences.repository.ts`) — all raw DB access:

```typescript
import "server-only";
import { db } from "@/db";
import { notificationPreferences } from "@/db/schema";
import { eq } from "drizzle-orm";

export async function getPreferences(userId: string) {
  const result = await db.query.notificationPreferences.findFirst({
    where: eq(notificationPreferences.userId, userId),
  });
  return result;
}

export async function createPreferences(
  userId: string,
  preferences: {
    billing: boolean;
    security: boolean;
    productUpdates: boolean;
    marketing: boolean;
  }
) {
  const [newPrefs] = await db
    .insert(notificationPreferences)
    .values({ userId, ...preferences })
    .returning();
  return newPrefs;
}

export async function updatePreferences(
  userId: string,
  preferences: {
    billing?: boolean;
    security?: boolean;
    productUpdates?: boolean;
    marketing?: boolean;
  }
) {
  const [updated] = await db
    .update(notificationPreferences)
    .set({ ...preferences, updatedAt: new Date() })
    .where(eq(notificationPreferences.userId, userId))
    .returning();
  return updated;
}
```

**Service** (`preferences.service.ts`) — business logic, no raw DB calls:

```typescript
import "server-only";
import {
  getPreferences,
  createPreferences,
  updatePreferences,
} from "./preferences.repository";
import type { PreferencesData, UpdatePreferencesInput } from "./types";

const DEFAULT_PREFERENCES: PreferencesData = {
  billing: true,
  security: true,
  productUpdates: true,
  marketing: true,
};

// Auto-creates preferences if missing (upsert pattern)
export async function getUserPreferences(userId: string): Promise<PreferencesData> {
  let prefs = await getPreferences(userId);

  if (!prefs) {
    prefs = await createPreferences(userId, DEFAULT_PREFERENCES);
  }

  return {
    billing: prefs.billing,
    security: prefs.security,
    productUpdates: prefs.productUpdates,
    marketing: prefs.marketing,
  };
}

export async function updateUserPreferences(
  userId: string,
  updates: UpdatePreferencesInput
): Promise<PreferencesData> {
  let prefs = await getPreferences(userId);

  if (!prefs) {
    prefs = await createPreferences(userId, DEFAULT_PREFERENCES);
  }

  const updated = await updatePreferences(userId, updates);

  return {
    billing: updated.billing,
    security: updated.security,
    productUpdates: updated.productUpdates,
    marketing: updated.marketing,
  };
}

// Convenience helper used before sending emails
export async function shouldSendEmail(
  userId: string,
  category: "billing" | "security" | "productUpdates" | "marketing"
): Promise<boolean> {
  const prefs = await getUserPreferences(userId);
  return prefs[category];
}
```

**Server Functions** (`preferences.actions.ts`) — auth + error boundary:

```typescript
import { createServerFn } from "@tanstack/react-start";
import { getWebRequest } from "@tanstack/react-start/server";
import { auth } from "@/lib/saas/auth/server";
import { getUserPreferences, updateUserPreferences } from "./preferences.service";
import type { PreferencesData, UpdatePreferencesInput } from "./types";

export const getNotificationPreferencesAction = createServerFn().handler(async () => {
  const request = getWebRequest();
  const session = await auth.api.getSession({ headers: request.headers });

  if (!session?.user) {
    return { error: "UNAUTHORIZED" as const };
  }

  try {
    const preferences = await getUserPreferences(session.user.id);
    return { data: preferences };
  } catch (error) {
    console.error("Failed to get notification preferences:", error);
    return { error: "FETCH_FAILED" as const };
  }
});

export const updateNotificationPreferencesAction = createServerFn()
  .validator((updates: UpdatePreferencesInput) => updates)
  .handler(async ({ data: updates }) => {
    const request = getWebRequest();
    const session = await auth.api.getSession({ headers: request.headers });

    if (!session?.user) {
      return { error: "UNAUTHORIZED" as const };
    }

    try {
      const preferences = await updateUserPreferences(session.user.id, updates);
      return { data: preferences };
    } catch (error) {
      console.error("Failed to update notification preferences:", error);
      return { error: "UPDATE_FAILED" as const };
    }
  });
```

Key takeaways from this example:
- The repository owns every `db.*` call; the service never touches `db` directly.
- The service encapsulates the upsert logic (create-if-missing) so actions stay thin.
- Actions only handle auth, call one service function, and return a `{ data?, error? }` discriminated union.
- `shouldSendEmail` is a domain helper that composes on top of `getUserPreferences` — it belongs in the service, not the action.

---

## Best Practices

### 1. Use the Correct Route Location

The `{-$locale}` segment provides optional locale prefix:
```
src/routes/{-$locale}/_app/app/projects.tsx    → /en/projects
src/routes/{-$locale}/_app/app/projects/$id.tsx → /en/projects/[id]
```

### 2. Follow Template Patterns

Match the template's architecture:

```
[YOURS] Good: Follow repository → service → server function pattern
[NO] Bad: Direct database access in components

[YOURS] Good: Use createServerFn() for mutations
[NO] Bad: API routes for simple mutations when a server function will do

[YOURS] Good: Internationalize all text
[NO] Bad: Hardcode English strings
```

### 3. Namespace Your Code

Prevent naming conflicts with template updates:

```typescript
// [YOURS] Good: Namespaced
export async function getMyProjects() { ... }
export async function createMyProject() { ... }

// [NO] Bad: Generic names
export async function getProjects() { ... }  // Might conflict
export async function create() { ... }       // Too generic
```

### 4. Document Customizations

Keep a `CUSTOMIZATIONS.md` in your project:

```markdown
# Project Customizations

## Added Features
- Projects management (`src/routes/{-$locale}/_app/app/projects.tsx`)
- Task tracking (`src/routes/{-$locale}/_app/app/tasks.tsx`)

## Modified Template Files
- `src/components/nav/user-menu.tsx` - Added "Projects" link
- `messages/en.json` - Added app.projects namespace

## Custom Database Tables
- `project` - User projects
- `task` - Project tasks

## Environment Variables
- `ENABLE_PROJECTS=true` - Enable projects feature
```

### 5. Extend, Don't Modify

When you need to use template utilities, extend them:

```typescript
// [NO] Bad: Modifying template file
// src/lib/utils.ts
export function cn(...) { ... }  // Template function
export function myCustomHelper() { ... }  // Your addition

// [YOURS] Good: Create wrapper/extension
// src/lib/app-utils.ts
import { cn } from "@/lib/utils";

export function myCustomHelper() {
  // Your custom logic
}

// Re-export template utils if needed
export { cn } from "@/lib/utils";
```

---

## Template Features You Can Use

### Reuse Template Services

```typescript
// [YOURS] Use template services in your code
import { canSendEmail } from "@/services/notifications/notifications.service";
import { deductCredits } from "@/services/billing/credits.service";

export async function processProjectTask(userId: string) {
  // Use template's credit system
  const result = await deductCredits(userId, 10, "Task processing");

  // Use template's notification system
  if (await canSendEmail(userId, "productUpdates")) {
    await sendEmail(...);
  }
}
```

### Reuse UI Components

```typescript
// [YOURS] Use template UI components
import { Card } from "@/components/ui/card";
import { Button } from "@/components/ui/button";
import { Badge } from "@/components/ui/badge";

export function ProjectCard({ project }) {
  return (
    <Card>
      <h3>{project.name}</h3>
      <Badge>{project.status}</Badge>
      <Button>View</Button>
    </Card>
  );
}
```

### Extend Auth Middleware

```typescript
// src/middleware.ts (if you need to customize)
import { betterAuth } from "better-auth/client";

// Template middleware already handles auth
// Add your custom logic here if needed
export default betterAuth({
  // ... existing config
});
```

---

## Updating with Custom Code

When template updates arrive:

### Git Merge Approach

```bash
# Your custom code in (app)/ rarely conflicts
git fetch template
git merge template/master

# Conflicts will likely be in:
# - package.json
# - messages/en.json
# - Files you modified (nav, etc.)
```

### Migration Guide Approach

Most updates won't touch your `(app)/` code, making migration easy:

1. Copy new/updated template files
2. Your custom code remains unchanged
3. Merge only shared files (package.json, translations)

---

## Common Patterns

### Feature Flag

```typescript
// src/config/features.ts
export const FEATURES = {
  projects: process.env.ENABLE_PROJECTS === "true",
  tasks: process.env.ENABLE_TASKS === "true",
} as const;

// Use in navigation
import { FEATURES } from "@/config/features";

{FEATURES.projects && <Link href="/projects">Projects</Link>}
```

### Permissions/RBAC

```typescript
// Extend template's role system
import { auth } from "@/lib/saas/auth/server";
import { getWebRequest } from "@tanstack/react-start/server";

export async function requireProjectAccess(projectId: string) {
  const request = getWebRequest();
  const session = await auth.api.getSession({ headers: request.headers });

  if (!session?.user) {
    throw new Error("Unauthorized");
  }

  const project = await db.query.project.findFirst({
    where: eq(project.id, projectId),
  });

  if (project?.userId !== session.user.id) {
    throw new Error("Forbidden");
  }

  return { session, project };
}
```

### Credits Integration

```typescript
// Use template's credit system
import { deductCredits, getUserCredits } from "@/services/billing/credits.service";

export async function processAITask(userId: string, taskData: unknown) {
  // Check if user has credits
  const credits = await getUserCredits(userId);
  if (credits < 10) {
    return { error: "INSUFFICIENT_CREDITS" };
  }

  // Process task
  const result = await processWithAI(taskData);

  // Deduct credits
  await deductCredits(userId, 10, "AI task processing");

  return { data: result };
}
```

---

## Troubleshooting

### "Cannot find module '@/services/app/...'"

Make sure you created the directory and added `index.ts`:

```bash
mkdir -p src/services/app/project
touch src/services/app/project/index.ts
```

### Database schema not found

Run the migration:

```bash
npm run db:push
npm run db:generate  # If using typed queries
```

### Type errors after update

Regenerate types:

```bash
npm run db:generate
npm run type-check
```

---

## FAQ

**Q: Can I modify template files?**
A: Avoid it when possible. Extending is better. If you must modify, document it in `CUSTOMIZATIONS.md`.

**Q: Where should API routes go?**
A: `src/server/app/` for custom server functions

**Q: Can I delete example features?**
A: Yes, remove any example routes from `src/routes/{-$locale}/_app/` safely.

**Q: How do I add my own email templates?**
A: Create them in `src/components/email-templates/app/` following template patterns.

**Q: What if I need to modify template services?**
A: Create a wrapper service in `src/services/app/` that uses the template service.

---

## Related Documentation

- [Updating Template](../../customizing/updating) - How to receive updates
- [Architecture](../../reference/architecture/tanstack) - Understanding the structure
- [Conventions](../../customizing/conventions) - Code style and patterns
- Template Boundaries: [Next.js](../../getting-started/nextjs/template-boundaries) · [TanStack](../../getting-started/tanstack/template-boundaries)

See also: [Next.js version](./nextjs)

---

**Last Updated:** March 11, 2026
