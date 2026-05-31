---
title: "Email System"
---

# Email System

Email utilities using Resend for sending transactional emails with React templates, integrated with Better Auth.

## Setup

Add to `.env.local`:

**Next.js:**
```bash
RESEND_API_KEY=your_resend_api_key
NEXT_PUBLIC_APP_NAME=YourApp           # Optional, defaults to 'JumpSaaS'
FROM_EMAIL=noreply@yourdomain.com      # Required
NEXT_PUBLIC_APP_URL=http://localhost:3000  # Required for Better Auth URLs
```

**TanStack:**
```bash
RESEND_API_KEY=your_resend_api_key
VITE_APP_NAME=YourApp                  # Optional, defaults to 'JumpSaaS'
FROM_EMAIL=noreply@yourdomain.com      # Required
VITE_APP_URL=http://localhost:3000     # Required for Better Auth URLs
```

## Directory Structure

The email utilities live in the same logical locations in both frameworks, but the paths differ slightly:

**Next.js:**
```
src/lib/saas/email/
├── index.ts                  # Resend client + FROM_EMAIL export
└── utils.ts                  # Email utilities (detectLocaleFromUrl)

src/lib/saas/auth/server/
└── email-handlers.ts         # Auth email handlers

src/lib/saas/stripe/
├── email-handlers.ts         # Billing email handlers
└── webhook-handlers.ts       # Webhook event handlers
```

**TanStack:**
```
src/lib/email/
├── index.ts                  # Resend client + FROM_EMAIL export
└── utils.ts                  # Email utilities

src/server/saas/auth/
└── email-handlers.ts         # Auth email handlers

src/server/saas/stripe/
├── email-handlers.ts         # Billing email handlers
└── webhook-handlers.ts       # Webhook event handlers
```

## Core Functions

### Email Client (`@/lib/saas/email`)
- `resend` - Configured Resend client instance
- `FROM_EMAIL` - Default from email address

### Email Handlers (`@/lib/saas/auth/server`)
- `sendPasswordResetEmail(email, resetUrl)` - Password reset with complete URL (1hr expiry)
- `sendVerificationEmail(email, verifyUrl)` - Email verification with complete URL (24hr expiry)

### Billing Email Handlers (`@/lib/saas/stripe`)
- `sendPaymentFailedEmail(userId, invoice)` - Payment failed notification with retry instructions
- `sendSubscriptionRenewedEmail(userId, subscription)` - Subscription renewal confirmation
- `sendSubscriptionCancelledEmail(userId, subscription)` - Subscription cancellation notice
- `sendTrialExpiringEmail(userId, subscription)` - Trial expiring reminder (sent 3 days before)

### Email Utilities (`@/lib/saas/email/utils`)
- `detectLocaleFromUrl(url)` - Extract locale from URL path

## Usage

```typescript
import { resend, FROM_EMAIL } from '@/lib/saas/email';
import { sendPasswordResetEmail, sendVerificationEmail } from '@/lib/saas/auth/server';

// Password reset & email verification are triggered automatically by Better Auth
// when users request password reset or sign up with email verification enabled
```

## Better Auth Configuration

Email functions are automatically called by Better Auth (configured in `src/lib/saas/auth/server/index.ts`):
- **Email verification**: Required before login, 24hr token expiry
- **Password reset**: 1hr token expiry, uses complete URLs
- **Configurable**: App name and from email via environment variables

## Templates

React email templates in `src/components/email-templates/`:

### Authentication Templates
- `password-reset-email.tsx` - Password reset with URL
- `verification-email.tsx` - Email verification with URL

### Billing Templates
- `payment-failed-email.tsx` - Payment failure notification with retry CTA
- `subscription-renewed-email.tsx` - Subscription renewal confirmation
- `subscription-cancelled-email.tsx` - Cancellation confirmation with reactivation option
- `trial-expiring-email.tsx` - Trial expiration reminder sent 3 days before

All templates are internationalized and support multiple locales (English and German).

## Billing Email Notifications

Automated emails triggered by Stripe webhook events to keep users informed about their subscription status.

### Email Triggers

| Email | Trigger | Webhook Event |
|-------|---------|---------------|
| Payment Failed | Invoice payment failed | `invoice.payment_failed` |
| Subscription Renewed | Successful payment on renewal | `invoice.paid` (for renewal invoices) |
| Subscription Cancelled | Subscription cancelled | `customer.subscription.deleted` |
| Trial Expiring | 3 days before trial ends | `customer.subscription.trial_will_end` |

### Implementation Flow

```
Stripe Event → Webhook Route → Event Handler → Email Service → Resend
```

1. **Webhook receives event** (`/api/webhooks/stripe/route.ts`)
2. **Event handler processes** (`@/lib/saas/stripe/webhook-handlers.ts`)
3. **Email service sends** (`@/lib/saas/stripe/email-handlers.ts`)
4. **React template renders** (`@/components/email-templates/billing/`)
5. **Resend delivers email**

### Example Usage

Emails are triggered automatically by webhooks - no manual invocation needed:

```typescript
// In webhook-handlers.ts
export async function handleInvoicePaymentFailed(invoice: Stripe.Invoice) {
  const subscription = await getActiveSubscription(userId);
  await sendPaymentFailedEmail(userId, invoice);
}
```

### Content Localization

All billing emails detect the user's locale and send in their preferred language:
- English (`en`)
- German (`de`)

Locale detection uses the user's database setting or falls back to English.

### Testing Billing Emails

See [BILLING.md - Testing Trial Emails](../reference/billing#testing-trial-emails) for Stripe webhook testing instructions.

## Security

- `@/lib/saas/email` is protected with `import "server-only"`
- Email handlers are in `auth/server/` (server-only)
- Resend API key never exposed to client
- Email utilities are pure functions (can run anywhere)
