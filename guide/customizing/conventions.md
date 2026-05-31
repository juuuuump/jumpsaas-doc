---
title: "Code Conventions"
---

# Code Conventions

## File Naming

### Components
- Use **kebab-case** for all component files
- Examples: `auth-form.tsx`, `user-profile.tsx`, `settings-sidebar.tsx`

### Pages / Routes

**Next.js** — route groups use parentheses and don't affect URLs:
- Examples: `page.tsx`, `layout.tsx`, `loading.tsx`, `error.tsx`
- Route groups: `(auth)/`, `(settings)/`, `(app)/`

**TanStack** — route files use flat file names with underscore-prefixed layout routes:
- Examples: `index.tsx`, `route.tsx`
- Layout routes: `_auth.tsx`, `_settings.tsx`, `_app.tsx`

### Directories
- Use lowercase with hyphens
- Feature directories: `components/auth/`, `components/upload/`

### Hooks and Actions
- **Hooks**: `use-kebab-case.ts`
  ```typescript
  // File: use-project-data.ts
  export function useProjectData() { ... }
  ```

- **Actions**: `actions.ts` or `[feature].actions.ts`
  ```typescript
  // File: project.actions.ts
  export async function createProject() { ... }
  ```

## Component Structure

### File Organization
```tsx
// components/user-profile.tsx
import { useState } from "react";
import { useTranslations } from "next-intl";
import { Button } from "@/components/ui/button";

interface UserProfileProps {
  userId: string;
}

export default function UserProfile({ userId }: UserProfileProps) {
  const t = useTranslations("Profile");
  const [isEditing, setIsEditing] = useState(false);

  return (
    <div>
      <h1>{t("title")}</h1>
      {/* Component content */}
    </div>
  );
}
```

### Export Pattern
- **Default export** for component: `export default function ComponentName()`
- **Named exports** for types/utilities if needed
- Use **PascalCase** for component function names
- **Import** using kebab-case filename: `import UserProfile from "@/components/user-profile"`

## Component Organization

### Directory Structure

Components are separated into template directories (never modify) and your custom directory:

```
src/components/
├── ui/                    # [Template] Shared UI primitives (Radix, shadcn/ui)
├── saas/                  # [Template] SaaS feature components
│   ├── auth/              # Authentication components
│   ├── billing/           # Billing and subscription components
│   ├── plan/              # Pricing plan components
│   ├── notification/      # Notification preference components
│   ├── user/              # User profile components
│   └── email-templates/   # Email templates
├── marketing/             # [Template] Marketing/branding components
│   ├── brand/             # Brand identity
│   ├── footer/            # Site footer
│   ├── logo/              # Logo component
│   ├── navigation/        # Site navigation
│   ├── icons/             # Icon components
│   ├── locale/            # Language selector
│   └── theme/             # Theme switcher
├── app/                   # [Custom] Your components
└── providers.tsx          # [Template] Global providers
```

**Legend:**
- `Template` - template code that receives updates
- `Custom` - safe customization area

### Domain-Based Organization

Organize components by domain following this pattern:

```
domain/
├── ui/                    # UI components
│   └── component.tsx
├── actions.ts             # Server actions (if needed)
├── provider.tsx           # Context provider (if needed)
├── types.ts               # TypeScript types
├── constants.ts           # Constants
└── use-*.ts              # Custom hooks
```

### Custom Components (`app/`)

All your custom components go in `components/app/`, organized by feature:

```
app/
├── dashboard/
│   ├── dashboard-stats.tsx
│   └── activity-feed.tsx
├── projects/
│   ├── project-card.tsx
│   ├── project-list.tsx
│   └── project-form.tsx
└── shared/
    ├── page-header.tsx
    └── empty-state.tsx
```

Co-locate related code within a feature directory:

```
app/projects/
├── project-card.tsx
├── project-list.tsx
├── project-form.tsx
├── project.actions.ts
├── project.types.ts
└── use-projects.ts
```

### UI Primitives (`ui/`)

Do not modify these. Import and use them in your components:

```typescript
import { Button } from "@/components/ui/button";
import { Card, CardHeader, CardTitle } from "@/components/ui/card";

export default function MyComponent() {
  return (
    <Card>
      <CardHeader>
        <CardTitle>My Custom Feature</CardTitle>
      </CardHeader>
      <Button>Click Me</Button>
    </Card>
  );
}
```

### Composition Over Modification

Never edit template components. Compose with them instead:

**Avoid: modifying a template component**
```typescript
// Editing components/saas/billing/ui/billing-subscription-card.tsx
export function BillingSubscriptionCard() {
  // Adding custom fields here
}
```

**Preferred: compose with a template component**
```typescript
// components/app/custom-billing.tsx
import { BillingSubscriptionCard } from "@/components/saas/billing/ui";

export function CustomBilling() {
  return (
    <div>
      <BillingSubscriptionCard />
      {/* Your additional content */}
      <CustomBillingStats />
    </div>
  );
}
```

Use the `className` prop with `cn()` to customize styles on template components:

```typescript
import { Button } from "@/components/ui/button";
import { Card } from "@/components/ui/card";
import { cn } from "@/lib/shared/utils";

export function CustomCard() {
  return (
    <Card className={cn("border-blue-500 shadow-lg")}>
      <Button className="bg-gradient-to-r from-blue-500 to-purple-500">
        Custom Styled Button
      </Button>
    </Card>
  );
}
```

### Always Use Shared UI Components

Before creating custom markup, check if a shared component exists in `components/ui/`:

**Avoid:**
```typescript
<div className="rounded-lg border bg-card p-4">
  <h2>Title</h2>
</div>
```

**Preferred:**
```typescript
import { Card, CardHeader, CardTitle } from "@/components/ui/card";

<Card>
  <CardHeader>
    <CardTitle>Title</CardTitle>
  </CardHeader>
</Card>
```

Always use `Table`, `TableHeader`, `TableBody`, `TableRow`, `TableHead`, `TableCell` from `components/ui/table` instead of raw `<table>` markup.

### Keep Components Focused

Each component should have a single responsibility:

**Avoid: too many responsibilities**
```typescript
export function ProjectDashboard() {
  // Handles auth, data fetching, UI, and actions all in one
}
```

**Preferred: focused components**
```typescript
export function ProjectDashboard() {
  return (
    <>
      <ProjectHeader />
      <ProjectStats />
      <ProjectList />
      <ProjectActions />
    </>
  );
}
```

### Creating Custom Components

#### Simple component
```typescript
// components/app/project-card.tsx
import { Card, CardHeader, CardTitle } from "@/components/ui/card";
import { Badge } from "@/components/ui/badge";

interface ProjectCardProps {
  name: string;
  status: "active" | "inactive";
}

export default function ProjectCard({ name, status }: ProjectCardProps) {
  return (
    <Card>
      <CardHeader>
        <CardTitle>{name}</CardTitle>
        <Badge>{status}</Badge>
      </CardHeader>
    </Card>
  );
}
```

#### Component with translations
```typescript
// components/app/dashboard-header.tsx
import { useTranslations } from "next-intl";

export default function DashboardHeader() {
  const t = useTranslations("dashboard");

  return (
    <header>
      <h1>{t("title")}</h1>
      <p>{t("description")}</p>
    </header>
  );
}

// messages/en.json
{
  "dashboard": {
    "title": "Dashboard",
    "description": "Welcome to your dashboard"
  }
}
```

#### Component with server actions
```typescript
// components/app/project-form.tsx
"use client";

import { useTransition } from "react";
import { Button } from "@/components/ui/button";
import { createProject } from "@/components/app/project.actions";

export default function ProjectForm() {
  const [isPending, startTransition] = useTransition();

  const handleSubmit = () => {
    startTransition(async () => {
      const result = await createProject({ name: "New Project" });
      if (result.error) {
        // Handle error
      }
    });
  };

  return (
    <Button onClick={handleSubmit} disabled={isPending}>
      Create Project
    </Button>
  );
}

// components/app/project.actions.ts
"use server";

import { headers } from "next/headers";
import { auth } from "@/lib/saas/auth/server";
import { db } from "@/db";

export async function createProject(data: { name: string }) {
  const session = await auth.api.getSession({ headers: await headers() });

  if (!session?.user) {
    return { error: "UNAUTHORIZED" };
  }

  try {
    const project = await db.insert(...);
    return { data: project };
  } catch (error) {
    return { error: "CREATE_FAILED" };
  }
}
```

#### Component with context
```typescript
// components/app/project-provider.tsx
"use client";

import { createContext, useContext, ReactNode } from "react";

interface ProjectContextType {
  projectId: string;
  projectName: string;
}

const ProjectContext = createContext<ProjectContextType | undefined>(undefined);

export function ProjectProvider({
  projectId,
  projectName,
  children,
}: {
  projectId: string;
  projectName: string;
  children: ReactNode;
}) {
  return (
    <ProjectContext.Provider value={{ projectId, projectName }}>
      {children}
    </ProjectContext.Provider>
  );
}

export function useProject() {
  const context = useContext(ProjectContext);
  if (!context) {
    throw new Error("useProject must be used within ProjectProvider");
  }
  return context;
}
```

### Import Aliases

Use path aliases for clean imports:

```typescript
// Preferred
import { Button } from "@/components/ui/button";
import { ProjectCard } from "@/components/app/project-card";
import { db } from "@/db";

// Avoid
import { Button } from "../../../components/ui/button";
import { ProjectCard } from "../../app/project-card";
import { db } from "../../../db";
```

### Testing Components

Place tests next to components:

```
app/projects/
├── project-card.tsx
├── project-card.test.tsx
├── project-list.tsx
└── project-list.test.tsx
```

## TypeScript Best Practices

### No Type Assertions
**NEVER** use `as` type assertions. They bypass TypeScript's type safety.

**Avoid:**
```typescript
const locale = params.locale as string;
const user = data as User;
const value = input as number;
```

**Preferred:**
```typescript
// Type guards with fallbacks
const locale = typeof params.locale === "string" ? params.locale : "en";

// Proper type checking
if (isUser(data)) {
  const user = data;
}

// Runtime validation
const value = typeof input === "number" ? input : 0;
```

### Type Guards
Create proper type guards instead of assertions:

```typescript
function isUser(data: unknown): data is User {
  return (
    typeof data === "object" &&
    data !== null &&
    "id" in data &&
    "email" in data
  );
}

// Use the type guard
if (isUser(data)) {
  console.log(data.email); // TypeScript knows data is User
}
```

### Define Clear Interfaces

```typescript
// types.ts
export interface Project {
  id: string;
  name: string;
  createdAt: Date;
  userId: string;
}

// component.tsx
interface ProjectCardProps {
  project: Project;
  onDelete?: (id: string) => void;
}

export function ProjectCard({ project, onDelete }: ProjectCardProps) {
  // ...
}
```

### Optional Chaining
Always use optional chaining for potentially undefined values:

```typescript
// Session access
const userName = session?.user.name;
const userEmail = session?.user?.email ?? "unknown@example.com";

// Nested objects
const city = user?.address?.city ?? "Unknown";
```

### Fallback Values
Provide sensible fallback values:

```typescript
// String with fallback
const displayName = user?.name ?? "Anonymous";

// Number with fallback
const credits = user?.creditsRemaining ?? 0;

// Array with fallback
const items = response?.data ?? [];
```

## Internationalization (i18n)

### Translation Keys
All user-facing text must use translations. The i18n library differs by framework:

**Next.js (next-intl):**
```tsx
import { useTranslations } from "next-intl";

export default function MyComponent() {
  const t = useTranslations("ComponentName");

  return (
    <div>
      <h1>{t("title")}</h1>
      <p>{t("description")}</p>
      <button>{t("submitButton")}</button>
    </div>
  );
}
```

**TanStack (ParaglideJS):**
```tsx
import * as m from "@/paraglide/messages";

export default function MyComponent() {
  return (
    <div>
      <h1>{m.componentName_title()}</h1>
      <p>{m.componentName_description()}</p>
      <button>{m.componentName_submitButton()}</button>
    </div>
  );
}
```

> ParaglideJS generates typed message functions from your `messages/` files — message keys map to function names using underscores. Run `pnpm dev` or `pnpm build` to regenerate after adding new keys.

### Translation File Structure
Organize by component/feature namespace:

```json
{
  "ComponentName": {
    "title": "Page Title",
    "description": "Page description",
    "submitButton": "Submit"
  },
  "Auth": {
    "signIn": "Sign In",
    "signOut": "Sign Out",
    "email": "Email",
    "password": "Password"
  },
  "Settings": {
    "Profile": {
      "title": "Profile Settings",
      "name": "Name",
      "nameDescription": "Your display name"
    }
  }
}
```

### Navigation Links

Navigation imports differ between frameworks — both handle locale routing, but through different mechanisms.

**Next.js** — always import from `@/i18n/navigation`, never from `next/link` or `next/navigation`:

```tsx
// Correct — locale-aware
import { Link, useRouter, usePathname, redirect } from "@/i18n/navigation";

<Link href="/settings">Settings</Link>
// Automatically becomes /en/settings or /de/settings
```

```tsx
// Wrong — breaks locale routing
import Link from "next/link";
import { useRouter, usePathname } from "next/navigation"; // returns /en/pricing, causing double-prefix bugs
```

**The only Next.js exception:** `useSearchParams`, `useParams` — no i18n equivalent, still use `next/navigation`.

**TanStack** — use TanStack Router's `Link` directly; locale is a route param, not a wrapper. Get the current locale from `useParams`:

```tsx
import { Link, useNavigate, useParams } from "@tanstack/react-router";

// Read locale from the current route
const { locale = "en" } = useParams({ strict: false });

// Use it when navigating
<Link to="/$locale/settings" params={{ locale }}>Settings</Link>
```

For programmatic navigation:
```tsx
const navigate = useNavigate();
navigate({ to: "/$locale/sign-in", params: { locale } });
```

The `{ strict: false }` is important because the `{-$locale}` segment is optional — it can be `undefined` on the default locale.

### Adding a New Language

The steps differ by framework.

**Next.js (next-intl):**

1. **Routing config** — `src/i18n/routing.ts`:
   ```typescript
   export const routing = defineRouting({
     locales: ['en', 'de', 'fr'], // add 'fr'
     defaultLocale: 'en'
   });
   ```

2. **Locale constants** — `src/lib/shared/locale/constants.ts`:
   ```typescript
   export const locales = [
     { code: "en", label: "English", flag: "🇺🇸" },
     { code: "de", label: "Deutsch", flag: "🇩🇪" },
     { code: "fr", label: "Français", flag: "🇫🇷" },
   ] as const;
   ```

3. **Message files** — copy and translate:
   ```bash
   cp -r messages/en messages/fr
   ```
   Files to translate: `auth.json`, `billing.json`, `email.json`, `errors.json`, `footer.json`, `legal.json`, `marketing.json`, `navigation.json`, `plan.json`, `settings.json`, `theme.json`.

4. No further code changes — `src/i18n/request.ts` picks up the new locale automatically.

**TanStack (ParaglideJS):**

1. **Inlang project config** — add the locale to `project.inlang/settings.json`:
   ```json
   {
     "sourceLanguageTag": "en",
     "languageTags": ["en", "de", "fr"]
   }
   ```

2. **Message files** — create `messages/fr.json` (TanStack uses a single file per locale, not a directory):
   ```bash
   cp messages/en.json messages/fr.json
   ```
   Translate the values in `messages/fr.json`.

3. **Regenerate Paraglide output** — run `pnpm dev` or `pnpm build`; the `src/paraglide/` directory is regenerated automatically. Do not edit it by hand.

---

### What to Translate
**Translate:**
- All UI text (buttons, labels, headings)
- Form placeholders and validation messages
- Error and success messages
- Page titles and meta descriptions
- Email subject lines and content

**Don't translate:**
- Code comments
- Environment variable names
- Log messages
- API endpoints
- Configuration keys

## Styling

### CSS Variables Over Hardcoded Colors
Use CSS variable-based Tailwind classes for automatic theme support:

**Preferred:**
```tsx
<div className="text-foreground bg-background border-border">
  <h1 className="text-primary">Title</h1>
  <p className="text-muted-foreground">Description</p>
</div>
```

**Avoid:**
```tsx
<div className="text-slate-900 bg-white border-gray-200">
  <h1 className="text-green-600">Title</h1>
  <p className="text-gray-600">Description</p>
</div>
```

### Available CSS Variables
```css
/* Text colors */
text-foreground          /* Primary text */
text-muted-foreground    /* Secondary text */
text-primary             /* Primary brand color text */
text-destructive         /* Error/danger text */

/* Backgrounds */
bg-background            /* Main background */
bg-card                  /* Card backgrounds */
bg-primary               /* Primary brand color */
bg-primary-light         /* Subtle accent */
bg-muted                 /* Muted backgrounds */

/* Borders */
border-border            /* Standard borders */
border-input             /* Input borders */
border-ring              /* Focus rings */

/* Interactive states */
hover:bg-accent          /* Hover backgrounds */
hover:text-accent-foreground
focus:ring-ring          /* Focus rings */
```

### Icon Usage
Use **Phosphor Icons** with the "Icon" suffix.

**TanStack** — all components run on the client, so always use the standard import:
```tsx
import { SignOutIcon, CreditCardIcon, GearIcon } from "@phosphor-icons/react";
```

**Next.js** — the correct import path depends on whether the component is a Server or Client Component:

#### Client Components (with "use client" directive)
```tsx
"use client";
import { SignOutIcon, CreditCardIcon, GearIcon } from "@phosphor-icons/react";
```

#### Server Components (async functions, no "use client")
```tsx
import { CheckIcon, DownloadIcon } from "@phosphor-icons/react/dist/ssr";

export default async function ServerComponent() {
  return <CheckIcon className="h-4 w-4" />;
}
```

**Common Next.js error:** using `@phosphor-icons/react` in a server component causes:
```
TypeError: (0 , d.createContext) is not a function
```
This happens because the client bundle uses React's `createContext` which doesn't exist in the server render environment. TanStack users never hit this error.

**Avoid (Next.js only):**
```tsx
// Server component using client import — will crash at build
import { CheckIcon } from "@phosphor-icons/react";
export default async function PlanCard() { ... }
```

```tsx
// Outdated naming (no "Icon" suffix)
import { SignOut, CreditCard } from "@phosphor-icons/react"; // Avoid this naming
```

### Interactive Element States
Consistent hover and focus patterns across all interactive elements:

```tsx
// Buttons
<button
  className="
    border border-input
    hover:border-ring hover:ring-ring/50 hover:ring-[3px]
    focus-visible:border-ring focus-visible:ring-ring/50 focus-visible:ring-[3px]
    transition-all
  "
>
  Click me
</button>

// Links
<Link
  className="
    text-muted-foreground
    hover:text-foreground
    transition-colors
  "
  href="/settings"
>
  Settings
</Link>

// Dropdown triggers (no persistent ring)
<DropdownMenuTrigger
  className="
    hover:ring-0
    focus-visible:ring-0
  "
>
  Open
</DropdownMenuTrigger>
```

## Code Style

### Minimal Comments
Code should be self-documenting through clear naming and structure.

**Preferred (no comment needed):**
```tsx
function calculateDiscountedPrice(price: number, discountPercent: number) {
  const discountAmount = price * (discountPercent / 100);
  return price - discountAmount;
}
```

**Avoid (redundant comments):**
```tsx
// Calculate the discounted price
function calculatePrice(p: number, d: number) {
  // Get discount amount
  const a = p * (d / 100);
  // Return price minus discount
  return p - a;
}
```

**When to add comments:**
- Complex business logic
- Non-obvious algorithms
- Workarounds for external library issues
- Security-related code

### Avoid Over-Engineering
Only implement what's requested. Don't add unnecessary features.

**Avoid over-engineering:**
```tsx
// Added unnecessary abstraction for 3 similar operations
function createUserOperation(type: "create" | "update" | "delete") {
  return async (data: UserData) => {
    const operation = OPERATIONS[type];
    return await operation(data);
  };
}

const createUser = createUserOperation("create");
const updateUser = createUserOperation("update");
const deleteUser = createUserOperation("delete");
```

**Preferred: simple and direct**
```tsx
async function createUser(data: UserData) {
  return await db.insert(user).values(data);
}

async function updateUser(id: string, data: UserData) {
  return await db.update(user).set(data).where(eq(user.id, id));
}

async function deleteUser(id: string) {
  return await db.delete(user).where(eq(user.id, id));
}
```

**Principle:** Three similar lines of code is better than a premature abstraction.

### No Unnecessary Abstractions
Don't create helpers for one-time operations:

**Avoid unnecessary helpers:**
```tsx
function formatUserDisplayName(user: User | null | undefined) {
  return user?.name ?? "Anonymous";
}

// Used only once
const displayName = formatUserDisplayName(session?.user);
```

**Preferred: direct usage**
```tsx
const displayName = session?.user?.name ?? "Anonymous";
```

### Extract Reusable Logic Into Hooks

Use custom hooks for logic shared across multiple components:

```typescript
// components/app/use-projects.ts
export function useProjects() {
  const [projects, setProjects] = useState([]);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    // Fetch logic
  }, []);

  return { projects, loading };
}

// Multiple components can use this
import { useProjects } from "./use-projects";
```

### No Backwards-Compatibility Hacks
When removing code, delete it completely. No unused variable renames or tombstone comments.

**Avoid:**
```tsx
// Removed - old implementation
// function oldFunction() { ... }

function newFunction(
  _unusedParam: string,  // Keep for backwards compatibility
  activeParam: string
) {
  return activeParam;
}
```

**Preferred:**
```tsx
function newFunction(activeParam: string) {
  return activeParam;
}
```

### No Emojis in Logs
Use plain text for professional console output:

```typescript
// Avoid
console.log("[success] User created successfully");
console.error("[error] Failed to create user");

// Preferred
console.log("User created successfully");
console.error("Failed to create user");
```

## Session Management

### No Global State
Better Auth handles caching automatically. Fetch session where you need it.

**Preferred pattern:**
```tsx
// Each component fetches independently
export default function HeaderMenu() {
  const { data: session } = authClient.useSession();
  return <div>{session?.user.name}</div>;
}

export default function SidebarUser() {
  const { data: session } = authClient.useSession();
  return <Avatar src={session?.user.image} />;
}
```

**Avoid unnecessary prop drilling:**
```tsx
export default function Layout({ session }: { session: Session }) {
  return (
    <>
      <Header session={session} />
      <Sidebar session={session} />
    </>
  );
}
```

### No Prop Drilling
Components that need session should fetch it directly:

```tsx
// Server Component
export default async function ServerPage() {
  const session = await auth.api.getSession({ headers: await headers() });
  // Use session here
}

// Client Component
export default function ClientComponent() {
  const { data: session } = authClient.useSession();
  // Use session here
}
```

### Always Use Optional Chaining
```tsx
// Safe
const userName = session?.user.name;
const userEmail = session?.user?.email ?? "unknown@example.com";

// Unsafe
const userName = session.user.name; // Can crash if session is null
```

## Form Patterns

### React Hook Form + Zod
Standard pattern for all forms:

```tsx
"use client";
import { useForm } from "react-hook-form";
import { zodResolver } from "@hookform/resolvers/zod";
import { z } from "zod";

const formSchema = z.object({
  name: z.string().min(3).max(30),
  email: z.string().email(),
});

type FormValues = z.infer<typeof formSchema>;

export default function MyForm() {
  const form = useForm<FormValues>({
    resolver: zodResolver(formSchema),
    defaultValues: {
      name: "",
      email: "",
    },
  });

  async function onSubmit(values: FormValues) {
    // Handle form submission
  }

  return (
    <form onSubmit={form.handleSubmit(onSubmit)}>
      {/* Form fields */}
    </form>
  );
}
```

### Validation Messages
Add validation messages to translation files:

```json
{
  "Form": {
    "Validation": {
      "required": "This field is required",
      "email": "Please enter a valid email",
      "minLength": "Must be at least {min} characters",
      "maxLength": "Must be at most {max} characters"
    }
  }
}
```

## Error Handling

### User-Friendly Messages
Always show user-friendly error messages:

```tsx
import { toast } from "sonner";

try {
  await updateProfile(data);
  toast.success(t("updateSuccess"));
} catch (error) {
  // Don't expose raw errors to users
  toast.error(t("updateError"));

  // Log for debugging
  console.error("Profile update failed:", error);
}
```

### Defensive Coding
Handle edge cases gracefully:

```tsx
// Handle missing data
const items = response?.data ?? [];
if (items.length === 0) {
  return <EmptyState />;
}

// Handle loading states
if (isPending) {
  return <Skeleton />;
}

// Handle errors
if (error) {
  return <ErrorMessage error={error} />;
}
```

## Performance Best Practices

### Server vs Client Components

**Next.js (App Router)** — server components are the default; opt into client with `"use client"`:

```tsx
// Default: server component — runs only on the server
export default async function ProductList() {
  const products = await db.query.products.findMany();
  return <div>{products.map(...)}</div>;
}

// Client component — add directive when you need interactivity
"use client";
export default function InteractiveComponent() {
  const [state, setState] = useState();
}
```

**TanStack** — all components are client components by default (Vite/React). Use `createServerFn` for server-side data fetching; there is no `"use client"` directive:

```tsx
// Data fetching — use a server function called from the route loader
import { createServerFn } from "@tanstack/react-start";
export const getProducts = createServerFn().handler(async () => {
  return db.query.products.findMany();
});

// Route component — always runs on the client
export function ProductList({ products }: { products: Product[] }) {
  return <div>{products.map(...)}</div>;
}
```

### When to Use Client-Side State
Both frameworks use the same criteria:
- React hooks (`useState`, `useEffect`, etc.)
- Browser APIs (`localStorage`, `window`, etc.)
- Event handlers (`onClick`, `onChange`, etc.)
- Real-time updates or third-party client libraries

### Parallel Data Fetching
```tsx
// Preferred: fetch in parallel
export default async function Page() {
  const [user, posts] = await Promise.all([
    fetchUser(),
    fetchPosts(),
  ]);

  return <div>{/* Use data */}</div>;
}

// Avoid: sequential fetching when requests are independent
export default async function Page() {
  const user = await fetchUser();
  const posts = await fetchPosts(); // Waits for user
  return <div>{/* Use data */}</div>;
}
```

## Security Best Practices

### Input Validation
Always validate user input:

```tsx
// Form validation
const schema = z.object({
  email: z.string().email(),
  name: z.string().min(3).max(30),
});

// API validation
export async function POST(request: Request) {
  const body = await request.json();
  const validated = schema.safeParse(body);

  if (!validated.success) {
    return NextResponse.json(
      { error: "Invalid input" },
      { status: 400 }
    );
  }

  // Use validated.data
}
```

### Authentication Checks
Always check authentication in protected routes:

```tsx
// Server Component
export default async function ProtectedPage() {
  const session = await auth.api.getSession({ headers: await headers() });

  if (!session) {
    redirect("/sign-in");
  }

  // Protected content
}

// API Route
export async function GET(request: Request) {
  const session = await auth.api.getSession({ headers: request.headers });

  if (!session) {
    return NextResponse.json(
      { error: "Unauthorized" },
      { status: 401 }
    );
  }

  // Protected logic
}
```

### SQL Injection Prevention
Drizzle ORM prevents SQL injection, but never use raw string concatenation:

```tsx
// Safe (parameterized)
await db.select().from(users).where(eq(users.id, userId));

// Unsafe (if using raw SQL)
await db.execute(sql`SELECT * FROM users WHERE id = ${userId}`); // Actually safe with Drizzle's sql template
await db.execute(`SELECT * FROM users WHERE id = '${userId}'`); // NEVER do this!
```

## Git Commit Conventions

### Commit Message Format
```
type: brief description

Longer explanation if needed
```

### Types
- `feat` - New feature
- `fix` - Bug fix
- `refactor` - Code refactoring
- `docs` - Documentation changes
- `style` - Code style changes (formatting)
- `test` - Adding or updating tests
- `chore` - Maintenance tasks

### Examples
```bash
feat: add user profile avatar upload

fix: resolve session refresh issue after avatar change

refactor: simplify authentication flow

docs: update i18n setup instructions

style: format code with prettier

chore: update dependencies
```

## See Also

- [Adding Features — Next.js](adding-features/nextjs) · [TanStack](adding-features/tanstack) - Step-by-step guide for adding custom features
- [Template Boundaries](../getting-started/nextjs/template-boundaries) - What you can/cannot modify
- [Architecture — Next.js](../reference/architecture/nextjs) · [TanStack](../reference/architecture/tanstack) - Overall system architecture
- [Route Organization — Next.js](../reference/architecture/nextjs) · [TanStack](../reference/architecture/tanstack) - Route structure
