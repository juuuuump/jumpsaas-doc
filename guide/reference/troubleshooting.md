---
title: "Troubleshooting"
---

# Troubleshooting

Common problems, their causes, and how to fix them.

## Quick Reference

| Problem | Cause | Fix |
|---------|-------|-----|
| OAuth callback error | Wrong `BETTER_AUTH_URL` or redirect URI | Check env + OAuth app settings |
| Stripe webhooks not firing | Wrong webhook secret or endpoint | Re-run Stripe CLI, verify secret |
| "FROM_EMAIL required" on startup | Missing env var | Add to `.env` or `docker run -e` |
| CORS errors on file upload | R2 bucket CORS not configured | Run the CORS setup in [storage.md](../integrations/storage) |
| `/en/en/` double-locale in URLs *(Next.js)* | Using `next/navigation` instead of `@/i18n/navigation` | See [conventions.md](../customizing/conventions) |
| Route param undefined *(TanStack)* | Accessing locale outside a route context | Use `useParams` inside a route component |
| Emails sending but going to spam | FROM_EMAIL domain not verified in Resend | Verify domain in Resend dashboard |
| `pnpm db:push` fails | `DATABASE_URL` wrong or Postgres not running | Check docker compose status |
| Admin page 403 | User not marked as admin in DB | Set `role = 'admin'` in `auth_users` |
| Build fails — wrong URLs in bundle *(Next.js)* | `NEXT_PUBLIC_*` vars missing at build time | Set in `.env.production` before build |
| Build fails — env vars not exposed *(TanStack)* | `VITE_*` vars missing or wrong prefix | Set in `.env.production` before build |
| Billing plans not showing | Stripe price metadata not set | Follow [stripe.md](../integrations/stripe) metadata format |

---

## OAuth callback error

**Symptoms**

After signing in with Google, GitHub, or another OAuth provider, you are redirected to an error page. The URL may contain `error=redirect_uri_mismatch` or the provider shows "The redirect URI does not match".

**Root cause**

Better Auth constructs callback URLs using `BETTER_AUTH_URL`. If this variable is missing, wrong, or does not match the redirect URI registered in your OAuth app, the provider rejects the callback.

**Fix**

1. Confirm `BETTER_AUTH_URL` in your `.env` (local) or in Dokploy (production) is set to the exact base URL of your app — no trailing slash:
   ```bash
   BETTER_AUTH_URL=https://www.yourdomain.com
   ```
2. Open your OAuth provider's developer console (e.g. Google Cloud Console, GitHub OAuth Apps) and add the callback URL to the allowed redirect URIs list:
   ```
   https://www.yourdomain.com/api/auth/callback/google
   https://www.yourdomain.com/api/auth/callback/github
   ```
3. For local development, register `http://localhost:3000` as well and set `BETTER_AUTH_URL=http://localhost:3000` in `.env.local`.
4. After changing `BETTER_AUTH_URL` in production, trigger a full rebuild — it is a runtime variable and does not require a rebuild itself, but ensure the value is correct before the app starts.

See Deployment: [Next.js](../getting-started/nextjs/deployment) · [TanStack](../getting-started/tanstack/deployment) for full environment variable guidance.

---

## Stripe webhooks not firing

**Symptoms**

Subscriptions appear in Stripe but are not reflected in the database. Webhook events show as undelivered in the Stripe dashboard. The app logs show no incoming webhook requests.

**Root cause**

The most common causes are a mismatched `STRIPE_WEBHOOK_SECRET`, the Stripe CLI not forwarding to the correct local port, or the production webhook endpoint URL not matching the deployed app.

**Fix**

**Local development:**

1. Start the Stripe CLI listener and point it at your local server:
   ```bash
   stripe listen --forward-to localhost:3000/api/webhooks/stripe
   ```
2. Copy the webhook signing secret printed by the CLI (starts with `whsec_`):
   ```
   Ready! Your webhook signing secret is whsec_abc123...
   ```
3. Set it in `.env.local`:
   ```bash
   STRIPE_WEBHOOK_SECRET=whsec_abc123...
   ```
4. Restart the dev server after updating the variable.

**Production:**

1. In the [Stripe dashboard](https://dashboard.stripe.com/webhooks), open your webhook endpoint.
2. Confirm the endpoint URL is `https://www.yourdomain.com/api/webhooks/stripe`.
3. Click "Reveal" under Signing secret and compare it to `STRIPE_WEBHOOK_SECRET` in Dokploy. Update if they differ.
4. Check that the webhook is listening for all required events (at minimum: `checkout.session.completed`, `customer.subscription.updated`, `customer.subscription.deleted`, `invoice.payment_succeeded`, `invoice.payment_failed`).

See [stripe.md](../integrations/stripe) for the full list of required events.

---

## "FROM_EMAIL required" on startup

**Symptoms**

The app crashes on startup with an error message such as `FROM_EMAIL is required` or `Missing required environment variable: FROM_EMAIL`.

**Root cause**

The email configuration validates required environment variables at startup. `FROM_EMAIL` (and typically `RESEND_API_KEY`) must be present before the server initializes.

**Fix**

Add the missing variable to the appropriate env file:

```bash
FROM_EMAIL=no-reply@yourdomain.com
RESEND_API_KEY=re_abc123...
```

- **Local development**: add to `.env.local`
- **Docker Compose**: add to `.env` or pass with `-e`:
  ```bash
  docker run -e FROM_EMAIL=no-reply@yourdomain.com ...
  ```
- **Dokploy**: add via the Environment Variables panel, then redeploy

`FROM_EMAIL` must use a domain you have verified in Resend (see below). Using an unverified domain will cause delivery failures even if startup succeeds.

---

## CORS errors on file upload

**Symptoms**

File uploads fail in the browser with a CORS error in the console:

```
Access to fetch at 'https://your-bucket.r2.cloudflarestorage.com/...' from origin
'https://www.yourdomain.com' has been blocked by CORS policy
```

**Root cause**

Cloudflare R2 buckets block cross-origin requests by default. The bucket must have a CORS policy that allows your app's origin to perform `PUT` requests.

**Fix**

Follow the CORS configuration steps in [storage.md](../integrations/storage). The required policy allows `PUT` and `GET` from your app's origin:

```json
[
  {
    "AllowedOrigins": ["https://www.yourdomain.com"],
    "AllowedMethods": ["GET", "PUT"],
    "AllowedHeaders": ["Content-Type"],
    "MaxAgeSeconds": 3600
  }
]
```

For local development, add `http://localhost:3000` to `AllowedOrigins` as well, or use a separate development bucket.

Apply the policy using the Wrangler CLI:

```bash
npx wrangler r2 bucket cors put YOUR_BUCKET_NAME --rules cors-rules.json
```

---

## `/en/en/` double-locale in URLs

> **Next.js only.** This bug is specific to next-intl's middleware-based locale routing. TanStack uses locale as a route param, so double-prefixing cannot occur.

**Symptoms**

After a redirect or navigation, the URL contains a duplicated locale segment such as `/en/en/dashboard` or `/de/de/settings`. Authentication callbacks may redirect to broken URLs.

**Root cause**

Code that uses `next/navigation`'s `usePathname` or `useRouter` receives the full locale-prefixed path (e.g. `/en/dashboard`). When that path is passed to a router or used as a `callbackUrl`, the i18n middleware prepends the locale again, producing `/en/en/dashboard`.

**Fix**

Replace all imports of navigation hooks and components with the locale-aware equivalents from `@/i18n/navigation`:

```tsx
// Wrong
import { useRouter, usePathname } from "next/navigation";
import Link from "next/link";

// Correct
import { useRouter, usePathname, Link, redirect } from "@/i18n/navigation";
```

The only exceptions are `useSearchParams` and `useParams` — no i18n equivalent exists, still use `next/navigation`.

Search your codebase for incorrect imports:

```bash
grep -r "from \"next/navigation\"" src/app src/components src/lib
```

See [conventions.md](../customizing/conventions) for the full navigation rules.

---

## Route param undefined (TanStack)

> **TanStack only.**

**Symptoms**

`useParams()` returns `undefined` for the locale param, or a server function receives no locale context even though the URL includes a locale prefix.

**Root cause**

TanStack Router's `{-$locale}` segment makes the locale optional. Code that reads `params.locale` without a fallback will get `undefined` on the default locale when no prefix is in the URL.

**Fix**

Always provide a fallback when reading the locale param:

```tsx
import { useParams } from "@tanstack/react-router";

const { locale = "en" } = useParams({ strict: false });
```

In server functions, read it from the request URL rather than route params:

```typescript
import { getWebRequest } from "@tanstack/react-start/server";

const request = getWebRequest();
const locale = request.url.split("/")[3] ?? "en"; // or use your i18n helper
```

---

## Emails sending but going to spam

**Symptoms**

Emails are sent without errors and appear in Resend logs as delivered, but recipients find them in their spam or junk folder. Gmail may show a warning banner about the sender.

**Root cause**

Resend requires domain verification to send on behalf of a custom domain. Without verified SPF, DKIM, and DMARC records, receiving mail servers treat the messages as suspicious.

**Fix**

1. Log in to the [Resend dashboard](https://resend.com/domains) and open the Domains section.
2. Add your sending domain (the domain used in `FROM_EMAIL`).
3. Resend will display DNS records (SPF, DKIM, DMARC) to add to your DNS provider.
4. Add each record to your DNS configuration and wait for propagation (up to 48 hours).
5. Once Resend shows the domain as verified, send a test email and check deliverability with a tool such as [mail-tester.com](https://www.mail-tester.com).

Using a subdomain such as `mail.yourdomain.com` for `FROM_EMAIL` is a common practice that isolates sending reputation from your main domain.

---

## `pnpm db:push` fails

**Symptoms**

Running `pnpm db:push` exits with an error such as `ECONNREFUSED`, `could not connect to server`, `password authentication failed`, or `database does not exist`.

**Root cause**

Either `DATABASE_URL` is incorrect, the PostgreSQL container is not running, or the database has not been created.

**Fix**

1. Check that the Docker Compose stack is running:
   ```bash
   docker compose ps
   ```
   If the `db` service is not listed or shows a non-running status, start it:
   ```bash
   docker compose up -d
   ```

2. Confirm `DATABASE_URL` in `.env` (or `.env.local`) is correct. For the default Docker Compose setup:
   ```bash
   DATABASE_URL=postgresql://postgres:postgres@localhost:5432/jumpsaas
   ```

3. Verify the database is reachable:
   ```bash
   psql $DATABASE_URL -c "SELECT 1"
   ```

4. If the database does not exist, create it:
   ```bash
   psql postgresql://postgres:postgres@localhost:5432 -c "CREATE DATABASE jumpsaas"
   ```

5. Re-run the push:
   ```bash
   pnpm db:push
   ```

---

## Admin page 403

**Symptoms**

Navigating to `/admin` (or `/en/admin`) returns a 403 Forbidden response or redirects away, even when signed in.

**Root cause**

The admin route checks that the authenticated user has `role = 'admin'` in the `auth_users` table. New accounts are created with `role = 'user'` by default.

**Fix**

Promote your account to admin directly in the database. Connect with your preferred Postgres client or use `psql`:

```sql
UPDATE auth_users
SET role = 'admin'
WHERE email = 'your@email.com';
```

Using `psql` with Docker Compose:

```bash
docker compose exec db psql -U postgres -d jumpsaas \
  -c "UPDATE auth_users SET role = 'admin' WHERE email = 'your@email.com';"
```

Refresh the page after updating. Only grant admin access to accounts you control — admin routes expose sensitive user and billing data.

---

## Build fails — wrong URLs or missing env vars in bundle

**Symptoms**

The deployed app has broken OAuth, CORS, or Stripe errors that point to wrong URLs. Setting variables in the Dokploy environment panel did not fix it.

**Root cause**

Public client-side variables are baked into the JavaScript bundle at build time. Setting them at runtime has no effect — they must be present when the build runs. The prefix differs by framework:

| Framework | Public variable prefix |
|-----------|----------------------|
| Next.js | `NEXT_PUBLIC_` |
| TanStack (Vite) | `VITE_` |

**Fix — Next.js**

1. Open `.env.production` and set all `NEXT_PUBLIC_*` variables:
   ```bash
   NEXT_PUBLIC_APP_URL=https://www.yourdomain.com
   NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY=pk_live_51abc...
   ```

2. Commit, push, and trigger a **full rebuild** in Dokploy (not just a restart).

**Fix — TanStack**

1. Open `.env.production` and set all `VITE_*` variables:
   ```bash
   VITE_APP_URL=https://www.yourdomain.com
   VITE_STRIPE_PUBLISHABLE_KEY=pk_live_51abc...
   ```

2. Commit, push, and trigger a **full rebuild** in Dokploy.

Variables without the public prefix (`DATABASE_URL`, `STRIPE_SECRET_KEY`, etc.) are read at runtime and can be set in Dokploy's environment panel without a rebuild.

See Deployment: [Next.js](../getting-started/nextjs/deployment) · [TanStack](../getting-started/tanstack/deployment) for the full variable list.

---

## Billing plans not showing

**Symptoms**

The pricing page renders no plans, or plans appear with missing names and prices. The admin billing section may show an empty plan list.

**Root cause**

JumpSaaS reads plan metadata from Stripe price objects. If the required metadata keys are not set on your Stripe prices, the billing service cannot map them to display plans.

**Fix**

Each Stripe price used as a billing plan must have the correct metadata. Follow the format documented in [stripe.md](../integrations/stripe). At minimum, set:

| Key | Example value | Description |
|-----|---------------|-------------|
| `plan` | `pro` | Internal plan identifier |
| `interval` | `month` or `year` | Billing interval |

Set metadata in the [Stripe dashboard](https://dashboard.stripe.com/prices) by opening the price, clicking Edit, and adding key-value pairs under Metadata. Alternatively use the Stripe CLI:

```bash
stripe prices update price_abc123 \
  --metadata[plan]=pro \
  --metadata[interval]=month
```

After updating metadata, restart the app (or wait for the cache to expire) and reload the pricing page.

---

## Still stuck?

If none of the above resolves your issue, open a GitHub issue on the JumpSaaS repository:

1. Include the exact error message or screenshot.
2. Describe what you have already tried.
3. Share relevant environment details (Node version, deployment platform, which third-party services are configured).

Issues with a clear description and reproduction steps are resolved significantly faster.
