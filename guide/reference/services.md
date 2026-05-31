---
title: "Service Organization"
---

# Service Organization

This document explains how services and business logic are organized in the JumpSaaS template to separate template code from your custom application code.

## Directory Structure

```
src/services/
├── saas/                  # [Template] Core SaaS services
│   ├── auth/              # Authentication service
│   ├── billing/           # Billing service
│   ├── plan/              # Plan service
│   └── notifications/     # Notification service
└── app/                   # [Custom] Your services
```

**Legend:**
- `Template` - template code that receives updates
- `Custom` - safe customization area

## Service Layer Purpose

Services handle business logic and data transformation:

**Services should:**
- Transform raw database data into domain models
- Convert units (cents → dollars, timestamps → dates)
- Add computed fields (`isPaid`, `isActive`, `daysRemaining`)
- Merge and sort data from multiple sources
- Enforce business rules
- Cache expensive operations

**Services should not:**
- Format for display (currency symbols, date formatting)
- Handle translations (i18n)
- Shape data for specific UI components
- Direct database operations (use repositories instead)

## Naming Conventions

### Files
- **Directories**: `kebab-case` (`payment-method/`)
- **Features**: `{feature}.service.ts` (`query.service.ts`, `create.service.ts`)
- **Types**: `types.ts`
- **Index**: `index.ts`

### Types
- **Domain models**: `{Entity}Info` (`SubscriptionInfo`, `InvoiceInfo`)
- **Inputs**: `{Feature}Input` (`CheckoutInput`)
- **Results**: `{Feature}Result` (`CheckoutResult`)

### Functions
- **Queries**: `get{Entity}` (`getSubscription`, `getInvoices`)
- **Commands**: `{verb}{Entity}` (`cancelSubscription`, `createCustomer`)
- **Checks**: `has{State}` / `is{State}` (`hasActiveSubscription`)

## Template Services (saas/)

### `saas/auth/` - Authentication Service

Handles user authentication and session management:

```
auth/
├── session.service.ts     # Session queries
├── session.actions.ts     # Session mutations
├── deletion.actions.ts    # Account deletion
├── email.service.ts       # Email verification
└── types.ts               # Auth types
```

**Example:**
```typescript
import { getUserSessions } from "@/services/saas/auth/session.service";
import { signOut } from "@/services/saas/auth/session.actions";
```

### `saas/billing/` - Billing Service

Manages subscriptions, payments, and invoices:

```
billing/
├── types.ts                     # CustomerInfo, SubscriptionInfo, InvoiceInfo
├── errors.ts                    # SubscriptionNotFoundError, etc.
├── index.ts
├── customer/
│   ├── query.service.ts         # getCustomerByUserId
│   ├── create.service.ts        # createStripeCustomer
│   ├── sync.service.ts          # syncCustomerFromStripe
│   └── index.ts
├── subscription/
│   ├── types.ts
│   ├── query.service.ts         # getActiveSubscription, getUserPlanSlug
│   ├── cancel.service.ts        # cancelSubscription
│   ├── resume.service.ts        # resumeSubscription
│   ├── update.service.ts        # updateSubscriptionPlan
│   ├── sync.service.ts
│   └── index.ts
├── invoice/
│   ├── types.ts
│   ├── query.service.ts         # getInvoices
│   ├── sync.service.ts
│   └── index.ts
└── payment-method/
    ├── types.ts
    ├── query.service.ts         # getPaymentMethods, getDefaultPaymentMethod
    ├── update.service.ts        # updateDefaultPaymentMethod
    ├── sync.service.ts
    └── index.ts
```

**Example Usage:**
```typescript
import {
  getActiveSubscription,
  getInvoices,
  getDefaultPaymentMethod,
} from "@/services/saas/billing";

export async function loadBillingData(userId: string) {
  const subscription = await getActiveSubscription(userId);
  const invoices = await getInvoices(userId, { limit: 10 });
  const paymentMethod = await getDefaultPaymentMethod(userId);

  return { subscription, invoices, paymentMethod };
}
```

**Data Transformation Example:**
```typescript
// billing/invoice.ts
export async function getInvoices(userId: string): Promise<InvoiceInfo[]> {
  const rawInvoices = await db.query.invoice.findMany(...);

  return rawInvoices.map((invoice) => ({
    id: invoice.id,
    amount: invoice.total / 100,        // Convert cents to dollars
    isPaid: invoice.status === "paid",  // Add a computed field
    date: invoice.createdAt,            // Keep raw dates here
    // currency: "$" + invoice.total    // Avoid UI formatting in services
  }));
}
```

### `saas/plan/` - Plan Service

Manages pricing plans:

```
plan/
├── query.service.ts       # Plan queries
├── plan.repository.ts     # Plan database operations
└── types.ts               # Plan types
```

**Example:**
```typescript
import { getPlans, getPlanBySlug } from "@/services/saas/plan";

const plans = await getPlans("monthly");
const plan = await getPlanBySlug("pro");
```

### `saas/notifications/` - Notification Service

Manages user notification preferences:

```
notifications/
├── query.service.ts       # Preference queries
└── actions.ts             # Preference updates
```

## Custom Services (app/)

All your custom business logic goes here. The service and repository layers are identical between frameworks — only the server function layer differs.

**Next.js:**
```
src/services/app/
├── projects/
│   ├── project.repository.ts    # Database operations
│   ├── project.service.ts       # Business logic
│   ├── project.actions.ts       # Server actions ("use server")
│   └── types.ts
└── teams/
    ├── team.repository.ts
    ├── team.service.ts
    └── types.ts
```

**TanStack:**
```
src/services/app/
├── projects/
│   ├── project.repository.ts    # Database operations (identical)
│   ├── project.service.ts       # Business logic (identical)
│   └── types.ts

src/server/app/
├── projects.ts                  # Server functions (createServerFn)
└── teams.ts
```

## Implementation Rules

### Server-Only (Required)

All service files must start with:
```typescript
import 'server-only';
```

This prevents service code from being accidentally bundled into the client.

## Service Patterns

### 1. Repository Pattern

Repositories handle raw database operations:

```typescript
// app/projects/project.repository.ts
import { db } from "@/db";
import { project } from "@/db/schema";
import { eq } from "drizzle-orm";

export const projectRepository = {
  async findById(id: string) {
    return db.query.project.findFirst({
      where: eq(project.id, id),
    });
  },

  async findByUserId(userId: string) {
    return db.query.project.findMany({
      where: eq(project.userId, userId),
      orderBy: (project, { desc }) => [desc(project.createdAt)],
    });
  },

  async create(data: NewProject) {
    const [created] = await db.insert(project).values(data).returning();
    return created;
  },

  async update(id: string, data: Partial<NewProject>) {
    const [updated] = await db
      .update(project)
      .set(data)
      .where(eq(project.id, id))
      .returning();
    return updated;
  },

  async delete(id: string) {
    await db.delete(project).where(eq(project.id, id));
  },
};
```

### 2. Service Layer

Services add business logic and transformations:

```typescript
// app/projects/project.service.ts
import { projectRepository } from "./project.repository";
import type { ProjectInfo } from "./types";

export async function getUserProjects(userId: string): Promise<ProjectInfo[]> {
  const projects = await projectRepository.findByUserId(userId);

  return projects.map((project) => ({
    id: project.id,
    name: project.name,
    status: project.status,
    createdAt: project.createdAt,
    // Add computed fields
    isActive: project.status === "active",
    daysOld: Math.floor(
      (Date.now() - project.createdAt.getTime()) / (1000 * 60 * 60 * 24)
    ),
  }));
}

export async function getProjectById(
  id: string,
  userId: string
): Promise<ProjectInfo | null> {
  const project = await projectRepository.findById(id);

  // Business logic: verify ownership
  if (!project || project.userId !== userId) {
    return null;
  }

  return {
    id: project.id,
    name: project.name,
    status: project.status,
    createdAt: project.createdAt,
    isActive: project.status === "active",
    daysOld: Math.floor(
      (Date.now() - project.createdAt.getTime()) / (1000 * 60 * 60 * 24)
    ),
  };
}
```

### 3. Server Functions

This layer handles auth, input validation, and calls into the service. The pattern differs by framework.

**Next.js — Server Actions** (`src/services/app/projects/project.actions.ts`):

```typescript
"use server";

import { revalidatePath } from "next/cache";
import { headers } from "next/headers";
import { auth } from "@/lib/saas/auth/server";
import { projectRepository } from "./project.repository";
import type { NewProject } from "./types";

export async function createProject(
  data: NewProject
): Promise<{ data?: { id: string }; error?: string }> {
  const session = await auth.api.getSession({ headers: await headers() });
  if (!session?.user) return { error: "UNAUTHORIZED" };

  try {
    const project = await projectRepository.create({ ...data, userId: session.user.id });
    revalidatePath("/projects");
    return { data: { id: project.id } };
  } catch (error) {
    console.error("Failed to create project:", error);
    return { error: "CREATE_FAILED" };
  }
}

export async function updateProject(
  id: string,
  data: Partial<NewProject>
): Promise<{ data?: { id: string }; error?: string }> {
  const session = await auth.api.getSession({ headers: await headers() });
  if (!session?.user) return { error: "UNAUTHORIZED" };

  try {
    const existing = await projectRepository.findById(id);
    if (!existing || existing.userId !== session.user.id) return { error: "NOT_FOUND" };

    const updated = await projectRepository.update(id, data);
    revalidatePath("/projects");
    revalidatePath(`/projects/${id}`);
    return { data: { id: updated.id } };
  } catch (error) {
    console.error("Failed to update project:", error);
    return { error: "UPDATE_FAILED" };
  }
}

export async function deleteProject(
  id: string
): Promise<{ data?: { success: true }; error?: string }> {
  const session = await auth.api.getSession({ headers: await headers() });
  if (!session?.user) return { error: "UNAUTHORIZED" };

  try {
    const existing = await projectRepository.findById(id);
    if (!existing || existing.userId !== session.user.id) return { error: "NOT_FOUND" };

    await projectRepository.delete(id);
    revalidatePath("/projects");
    return { data: { success: true } };
  } catch (error) {
    console.error("Failed to delete project:", error);
    return { error: "DELETE_FAILED" };
  }
}
```

**TanStack — Server Functions** (`src/server/app/projects.ts`):

```typescript
import { createServerFn } from "@tanstack/react-start";
import { getWebRequest } from "@tanstack/react-start/server";
import { auth } from "@/lib/saas/auth/server";
import { projectRepository } from "@/services/app/projects/project.repository";
import type { NewProject } from "@/services/app/projects/types";

export const createProject = createServerFn()
  .validator((data: NewProject) => data)
  .handler(async ({ data }) => {
    const request = getWebRequest();
    const session = await auth.api.getSession({ headers: request.headers });
    if (!session?.user) return { error: "UNAUTHORIZED" as const };

    try {
      const project = await projectRepository.create({ ...data, userId: session.user.id });
      return { data: { id: project.id } };
    } catch (error) {
      console.error("Failed to create project:", error);
      return { error: "CREATE_FAILED" as const };
    }
  });

export const updateProject = createServerFn()
  .validator((input: { id: string; data: Partial<NewProject> }) => input)
  .handler(async ({ data: { id, data } }) => {
    const request = getWebRequest();
    const session = await auth.api.getSession({ headers: request.headers });
    if (!session?.user) return { error: "UNAUTHORIZED" as const };

    try {
      const existing = await projectRepository.findById(id);
      if (!existing || existing.userId !== session.user.id) return { error: "NOT_FOUND" as const };

      const updated = await projectRepository.update(id, data);
      return { data: { id: updated.id } };
    } catch (error) {
      console.error("Failed to update project:", error);
      return { error: "UPDATE_FAILED" as const };
    }
  });

export const deleteProject = createServerFn()
  .validator((id: string) => id)
  .handler(async ({ data: id }) => {
    const request = getWebRequest();
    const session = await auth.api.getSession({ headers: request.headers });
    if (!session?.user) return { error: "UNAUTHORIZED" as const };

    try {
      const existing = await projectRepository.findById(id);
      if (!existing || existing.userId !== session.user.id) return { error: "NOT_FOUND" as const };

      await projectRepository.delete(id);
      return { data: { success: true as const } };
    } catch (error) {
      console.error("Failed to delete project:", error);
      return { error: "DELETE_FAILED" as const };
    }
  });
```

> **Key differences:** TanStack uses `createServerFn()` with `.validator()` for input validation and `getWebRequest()` to access headers. There is no `revalidatePath` — TanStack Router's loader invalidation handles cache refresh via the router's `invalidate()` API at the call site.

### 4. Type Definitions

Keep types in separate files:

```typescript
// app/projects/types.ts

// Database schema type (from Drizzle)
export type Project = typeof project.$inferSelect;
export type NewProject = typeof project.$inferInsert;

// Service layer type (with computed fields)
export interface ProjectInfo {
  id: string;
  name: string;
  status: "active" | "inactive";
  createdAt: Date;
  // Computed fields
  isActive: boolean;
  daysOld: number;
}

// Form types
export interface ProjectFormData {
  name: string;
  description?: string;
}
```

## Service Organization Patterns

### Simple Feature (Single File)

For simple features, use a single service file:

```typescript
// app/analytics/analytics.service.ts
export async function getAnalytics(userId: string) {
  const visits = await analyticsRepository.getVisits(userId);
  const conversions = await analyticsRepository.getConversions(userId);

  return {
    visits: visits.length,
    conversions: conversions.length,
    conversionRate: conversions.length / visits.length,
  };
}
```

### Complex Feature (Directory Structure)

For complex features, use subdirectories:

```
teams/
├── types.ts               # Type definitions
├── team.repository.ts     # Database operations
├── team.service.ts        # Core business logic
├── team.actions.ts        # Server actions
├── member/                # Subdomain
│   ├── member.repository.ts
│   ├── member.service.ts
│   └── member.actions.ts
└── invite/                # Subdomain
    ├── invite.repository.ts
    ├── invite.service.ts
    └── invite.actions.ts
```

For features with significant internal complexity, further separate database operations and pure helpers:

```
src/services/plan/sync/
├── types.ts        # Feature-specific types
├── db.ts           # Database operations
├── helpers.ts      # Pure helper functions
├── sync.service.ts # Business logic
└── index.ts        # Public exports
```

```typescript
// db.ts - database operations
export async function upsertPlan(data: PlanInsert): Promise<string> { ... }
export async function deactivatePlanById(id: string): Promise<void> { ... }

// helpers.ts - pure functions
export function getStripeProductId(data: unknown): string | null { ... }

// sync.service.ts - business logic only
import { upsertPlan, deactivatePlanById } from "./db";
import { getStripeProductId } from "./helpers";

export async function syncAllPlansFromStripe(): Promise<SyncResult> { ... }
```

### Query vs Command Separation

Separate reads from writes:

```
projects/
├── project.repository.ts  # Database operations
├── query.service.ts       # Read operations
├── command.service.ts     # Write operations
└── types.ts               # Types
```

## Database Operations Pattern

- **Repository layer** (`repositories/`) - Generic database operations (CRUD, queries, transactions)
- **`query.service.ts`** - Domain-specific read operations built on repositories
- **Simple features** - Single file using repository methods
- **Complex features** - Directory with separated concerns

**Rule:** Use repositories for generic DB operations (CRUD, basic queries). If a query involves domain logic or is used across services, put it in `query.service.ts`. Feature-specific DB operations go in `db.ts`.

## Error Handling

### Service Layer Errors

Return discriminated unions:

```typescript
export async function getProjectById(
  id: string
): Promise<{ data?: ProjectInfo; error?: string }> {
  try {
    const project = await projectRepository.findById(id);

    if (!project) {
      return { error: "NOT_FOUND" };
    }

    return { data: transformProject(project) };
  } catch (error) {
    console.error("Failed to fetch project:", error);
    return { error: "FETCH_FAILED" };
  }
}
```

### Custom Error Classes

For complex error handling:

```typescript
// app/projects/errors.ts
export class ProjectError extends Error {
  constructor(
    message: string,
    public code: string
  ) {
    super(message);
    this.name = "ProjectError";
  }
}

export class ProjectNotFoundError extends ProjectError {
  constructor(id: string) {
    super(`Project ${id} not found`, "NOT_FOUND");
  }
}

// project.service.ts
export async function getProjectById(id: string): Promise<ProjectInfo> {
  const project = await projectRepository.findById(id);

  if (!project) {
    throw new ProjectNotFoundError(id);
  }

  return transformProject(project);
}
```

## Caching

Use React's cache for expensive operations:

```typescript
import { cache } from "react";

export const getProjects = cache(async (userId: string) => {
  console.log("Fetching projects for:", userId); // Only logs once per request
  return projectRepository.findByUserId(userId);
});
```

## Testing Services

### Unit Testing Repositories

```typescript
// project.repository.test.ts
import { projectRepository } from "./project.repository";

describe("projectRepository", () => {
  it("creates a project", async () => {
    const project = await projectRepository.create({
      name: "Test Project",
      userId: "user-123",
    });

    expect(project).toHaveProperty("id");
    expect(project.name).toBe("Test Project");
  });
});
```

### Integration Testing Services

```typescript
// project.service.test.ts
import { getUserProjects } from "./project.service";

describe("getUserProjects", () => {
  it("returns projects with computed fields", async () => {
    const projects = await getUserProjects("user-123");

    expect(projects[0]).toHaveProperty("isActive");
    expect(projects[0]).toHaveProperty("daysOld");
  });
});
```

## Best Practices

### 1. Keep Services Pure

Services should be pure functions where possible:

```typescript
// Preferred - pure function
export function calculateDiscount(amount: number, percent: number): number {
  return amount * (percent / 100);
}

// Avoid - side effects
export function calculateDiscount(amount: number): number {
  console.log("Calculating discount"); // Side effect
  return amount * 0.1;
}
```

### 2. Single Responsibility

Each service file should have one clear purpose:

```typescript
// Preferred
// user-profile.service.ts - handles user profiles
// user-preferences.service.ts - handles preferences

// Avoid
// user.service.ts - handles everything
```

### 3. Explicit Dependencies

Make dependencies explicit in function signatures:

```typescript
// Preferred - dependencies are clear
export async function createProject(
  userId: string,
  data: NewProject,
  db: Database
) {
  // ...
}

// Avoid - hidden dependencies
export async function createProject(data: NewProject) {
  // Imports db from global scope
}
```

### 4. Type Safety

Always define return types:

```typescript
// Preferred
export async function getProject(id: string): Promise<ProjectInfo | null> {
  // ...
}

// Avoid
export async function getProject(id: string) {
  // Return type inferred
}
```

### 5. Error Codes

Use error codes for internationalization:

```typescript
// Service
export async function createProject(data: NewProject) {
  if (!data.name) {
    return { error: "NAME_REQUIRED" };
  }
  // ...
}

// Translation file (messages/en.json)
{
  "projects": {
    "errors": {
      "NAME_REQUIRED": "Project name is required",
      "CREATE_FAILED": "Failed to create project"
    }
  }
}

// Next.js component
import { useTranslations } from "next-intl";
const t = useTranslations("projects.errors");
const result = await createProject(data);
if (result.error) toast.error(t(result.error));

// TanStack component
import * as m from "@/paraglide/messages";
const result = await createProject({ data });
if (result.error) toast.error(m[`projects_errors_${result.error}`]?.() ?? result.error);
```

## See Also

- [Database](database) - Database schema and Drizzle ORM
- [Architecture](architecture) - Overall system architecture
- [Adding Features — Next.js](../customizing/adding-features/nextjs) · [TanStack](../customizing/adding-features/tanstack) - Complete guide for adding features
- [Template Boundaries](../getting-started/nextjs/template-boundaries) - What you can/cannot modify
