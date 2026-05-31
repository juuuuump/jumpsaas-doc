---
title: "Manual Testing Cases: Billing Email Notifications"
---

# Manual Testing Cases: Billing Email Notifications

## Test Environment Setup

**Prerequisites:**
- Dev server running (`pnpm dev`)
- Stripe CLI installed and authenticated
- Resend account configured (or email logs enabled)
- Test user account created
- PostgreSQL database running

**Setup Commands:**
```bash
# Terminal 1: Start dev server
pnpm dev

# Terminal 2: Start Stripe webhook listener
stripe listen --forward-to localhost:3000/api/webhooks/stripe
```

**Implementation Notes:**
- All emails are sent in English (`locale = "en"`) regardless of user's UI language
- Email subjects use `getTranslations()` from translation files (no hardcoded strings)
- Payment succeeded email includes plan name, billing period, and payment type detection
- `PaymentType` is defined in `src/services/billing/types.ts`
- `determinePaymentType()` maps Stripe `billing_reason` to payment type

---

## Test Suite 1: Email Template Rendering

### Test 1.1: Payment Succeeded Email - Initial Payment

**Objective:** Verify payment succeeded email renders correctly for a new subscription

**Steps:**
1. Navigate to billing settings: `http://localhost:3000/en/settings/billing`
2. Ensure billing notifications are ENABLED
3. Subscribe to a plan (triggers `invoice.paid` with `billing_reason: subscription_create`)
4. Check Resend dashboard or email logs

**Expected Results:**
- [YOURS] Email sent successfully
- [YOURS] Subject: "Payment Received - JumpSaaS" (from translation file)
- [YOURS] Heading: "Payment Received"
- [YOURS] Body: "Welcome! Your first payment for {planName} has been processed."
- [YOURS] Plan name displayed in detail box
- [YOURS] Amount displayed with correct currency format
- [YOURS] Payment date formatted correctly
- [YOURS] Billing period shown (e.g., "Jan 31, 2026 - Feb 28, 2026")
- [YOURS] Next billing date shown
- [YOURS] "View Invoice" button present (if invoice URL available)
- [YOURS] No TypeScript/rendering errors in logs

---

### Test 1.2: Payment Succeeded Email - Renewal

**Objective:** Verify renewal-specific messaging

**Steps:**
1. Trigger `invoice.paid` webhook with `billing_reason: subscription_cycle`
2. Check email content

**Expected Results:**
- [YOURS] Body: "Your {planName} subscription has been renewed."
- [YOURS] Plan name, amount, billing period all displayed
- [YOURS] Payment type correctly detected as "renewal"


DONE

---

### Test 1.3: Payment Succeeded Email - Upgrade

**Objective:** Verify upgrade-specific messaging

**Steps:**
1. Trigger `invoice.paid` webhook with `billing_reason: subscription_update`
2. Check email content

**Expected Results:**
- [YOURS] Body: "Your upgrade to {planName} has been processed."
- [YOURS] Payment type correctly detected as "upgrade"

DONE

---

### Test 1.4: Payment Succeeded Email - No Plan Name

**Objective:** Verify graceful fallback when plan name is unavailable

**Steps:**
1. Trigger `invoice.paid` with an invoice that has no line item description
2. Check email content

**Expected Results:**
- [YOURS] Body: "We've successfully processed your payment." (generic fallback)
- [YOURS] Plan label row NOT shown in detail box
- [YOURS] Amount and date still displayed correctly
- [YOURS] No errors or "undefined" text

---

### Test 1.5: Payment Failed Email

**Objective:** Verify payment failed email has critical styling and correct content

**Steps:**
1. Trigger: `stripe trigger invoice.payment_failed`
2. Check email

**Expected Results:**
- [YOURS] Email sent successfully (bypasses notification preferences)
- [YOURS] Subject: "Payment Failed - JumpSaaS" (from translation file)
- [YOURS] Heading: "Payment Failed"
- [YOURS] Red alert styling (background: #fef2f2, border: #fecaca)
- [YOURS] Amount due displayed
- [YOURS] Payment attempt timestamp shown with time
- [YOURS] "Update Payment Method" button links to `/en/settings/billing`
- [YOURS] Critical notification disclaimer at bottom
- [YOURS] Console log shows email was sent (NOT skipped)

---

### Test 1.6: Subscription Renewed Email

**Objective:** Verify subscription renewal confirmation

**Steps:**
1. Trigger: `stripe trigger customer.subscription.updated`
2. Check webhook logs to see if renewal was detected
3. Check email if sent

**Expected Results:**
- [YOURS] Email sent only if subscription period changed (renewal detected)
- [YOURS] Subject: "Subscription Renewed - JumpSaaS" (from translation file)
- [YOURS] Green success styling (background: #f0fdf4, border: #bbf7d0)
- [YOURS] Plan name displayed
- [YOURS] Amount charged shown
- [YOURS] Next billing date shown
- [YOURS] "Manage Subscription" button links to `/en/settings/billing`

**Note:** May not always send if subscription.updated event doesn't indicate renewal

---

### Test 1.7: Subscription Cancelled Email

**Objective:** Verify cancellation confirmation

**Steps:**
1. Trigger: `stripe trigger customer.subscription.deleted`
2. Check email

**Expected Results:**
- [YOURS] Email sent successfully
- [YOURS] Subject: "Subscription Cancelled - JumpSaaS" (from translation file)
- [YOURS] Yellow warning styling (background: #fef3c7, border: #fde68a)
- [YOURS] Plan name displayed
- [YOURS] Access end date shown
- [YOURS] Friendly message: "We'll miss you! You'll continue to have access until [date]."
- [YOURS] "Resubscribe" button links to `/en/pricing`

---

### Test 1.8: Trial Expiring Email

**Objective:** Verify trial warning email

**Steps:**
1. Trigger: `stripe trigger customer.subscription.trial_will_end`
2. Check email

**Expected Results:**
- [YOURS] Email sent successfully
- [YOURS] Subject: "Trial Ending Soon - JumpSaaS" (from translation file)
- [YOURS] Blue info styling (background: #dbeafe, border: #93c5fd)
- [YOURS] Days remaining calculated correctly
- [YOURS] Trial end date shown
- [YOURS] Next billing date displayed
- [YOURS] Auto-charge disclaimer present
- [YOURS] "No action needed" message shown
- [YOURS] "Continue Subscription" button links to `/en/settings/billing`

---

## Test Suite 2: Payment Type Detection

### Test 2.1: determinePaymentType Mapping

**Objective:** Verify Stripe `billing_reason` maps to correct `PaymentType`

**Mapping Table:**

| Stripe `billing_reason` | `PaymentType` | Email Body |
|--------------------------|---------------|------------|
| `subscription_create` | `initial` | "Welcome! Your first payment for {planName}..." |
| `subscription_cycle` | `renewal` | "Your {planName} subscription has been renewed." |
| `subscription_update` | `upgrade` | "Your upgrade to {planName} has been processed." |
| `manual` | `prorated` | "We've successfully processed your payment for {planName}." |
| `upcoming` | `prorated` | "We've successfully processed your payment for {planName}." |
| (any other) | `prorated` | "We've successfully processed your payment for {planName}." |

**Steps:**
1. For each billing reason, trigger an `invoice.paid` webhook
2. Verify the email body matches the expected message

---

## Test Suite 3: Notification Preferences

### Test 3.1: Billing Notifications Enabled

**Objective:** Verify emails send when billing notifications enabled

**Steps:**
1. Navigate to: `http://localhost:3000/en/settings/notification`
2. Ensure "Billing notifications" toggle is ON
3. Click "Save preferences"
4. Trigger: `stripe trigger invoice.paid`
5. Check logs

**Expected Results:**
- [YOURS] Console shows: Email sent successfully
- [YOURS] Email received in Resend dashboard
- [YOURS] No "skipped" message in logs

---

### Test 3.2: Billing Notifications Disabled (Non-Critical)

**Objective:** Verify non-critical emails are skipped when preferences disabled

**Steps:**
1. Navigate to notification settings
2. Turn OFF "Billing notifications" toggle
3. Click "Save preferences"
4. Trigger each non-critical email:
   - `stripe trigger invoice.paid`
   - `stripe trigger customer.subscription.updated`
   - `stripe trigger customer.subscription.deleted`
   - `stripe trigger customer.subscription.trial_will_end`
5. Check logs for each

**Expected Results for Each:**
- [YOURS] Console shows: "User has disabled billing notifications, skipping [email type]"
- [YOURS] No email sent to Resend
- [YOURS] Webhook still processes successfully (200 OK)
- [YOURS] No errors in application logs

---

### Test 3.3: Critical Email Bypasses Preferences

**Objective:** Verify payment failed email ALWAYS sends

**Steps:**
1. Ensure "Billing notifications" is DISABLED
2. Verify setting is saved
3. Trigger: `stripe trigger invoice.payment_failed`
4. Check logs and email

**Expected Results:**
- [YOURS] Email SENT despite disabled preference
- [YOURS] Console shows: Payment failed email sent successfully
- [YOURS] Email received in Resend
- [YOURS] No "skipped" message
- [YOURS] Critical email bypasses user preferences

**This is the most important test - critical emails must NEVER be blocked**

---

### Test 3.4: Re-enabling Notifications

**Objective:** Verify emails resume after re-enabling

**Steps:**
1. With notifications disabled, trigger: `stripe trigger invoice.paid`
2. Verify email was skipped
3. Navigate to notification settings
4. Turn ON "Billing notifications"
5. Save preferences
6. Trigger: `stripe trigger invoice.paid`
7. Check email

**Expected Results:**
- [YOURS] First trigger: Email skipped
- [YOURS] After re-enabling: Email sent successfully
- [YOURS] Preference changes take effect immediately

---

## Test Suite 4: Webhook Integration

### Test 4.1: Webhook Endpoint Availability

**Objective:** Verify webhook endpoint is accessible

**Steps:**
1. Ensure dev server is running
2. Check Stripe webhook listener output
3. Send test webhook: `stripe trigger invoice.paid`

**Expected Results:**
- [YOURS] Webhook listener shows: "Webhook received"
- [YOURS] Status code: 200
- [YOURS] No connection errors
- [YOURS] Application logs show webhook processing

---

### Test 4.2: Multiple Webhooks in Sequence

**Objective:** Test webhook processing doesn't block

**Steps:**
1. Trigger webhooks rapidly:
```bash
stripe trigger invoice.paid
stripe trigger invoice.payment_failed
stripe trigger customer.subscription.updated
```
2. Monitor logs

**Expected Results:**
- [YOURS] All webhooks processed
- [YOURS] All emails sent (or skipped per preferences)
- [YOURS] No race conditions or errors
- [YOURS] Each webhook returns 200 OK

---

### Test 4.3: Email Failure Doesn't Break Webhook

**Objective:** Verify webhook succeeds even if email fails

**Steps:**
1. Temporarily break email configuration (invalid Resend API key)
2. Trigger: `stripe trigger invoice.paid`
3. Check logs

**Expected Results:**
- [YOURS] Webhook still returns 200 OK
- [YOURS] Error logged: "Failed to send payment succeeded email"
- [YOURS] Webhook processing completes
- [YOURS] No application crash

**Restore email configuration after test**

---

## Test Suite 5: i18n Subject Lines

### Test 5.1: Subjects Use Translation Files

**Objective:** Verify all email subjects come from `messages/en/email.json`

**Steps:**
1. Trigger each webhook type and check email subject
2. Cross-reference with translation file keys

**Expected Subjects (from translation files):**

| Email Type | Subject |
|------------|---------|
| Payment Succeeded | "Payment Received - JumpSaaS" |
| Payment Failed | "Payment Failed - JumpSaaS" |
| Subscription Renewed | "Subscription Renewed - JumpSaaS" |
| Subscription Cancelled | "Subscription Cancelled - JumpSaaS" |
| Trial Expiring | "Trial Ending Soon - JumpSaaS" |

**Expected Results:**
- [YOURS] All subjects match translation file values
- [YOURS] `{appName}` placeholder resolved to "JumpSaaS"
- [YOURS] No hardcoded subject strings in email service

---

### Test 5.2: Translation Key Completeness

**Objective:** Verify all translation keys present in both locales

**Steps:**
1. Check `messages/en/email.json` billing section
2. Check `messages/de/email.json` billing section
3. Compare keys

**Expected Results:**
- [YOURS] All English keys present (including new: `bodyWithPlan`, `bodyInitial`, `bodyRenewal`, `bodyUpgrade`, `planLabel`, `billingPeriodLabel`)
- [YOURS] All German keys present and match English key structure
- [YOURS] No missing translations

---

## Test Suite 6: Edge Cases

### Test 6.1: Missing User Email

**Objective:** Handle missing email gracefully

**Steps:**
1. Create test scenario with user without email (if possible)
2. Trigger webhook
3. Check logs

**Expected Results:**
- [YOURS] No email sent
- [YOURS] No error thrown (early return in webhook handler)
- [YOURS] Webhook still succeeds
- [YOURS] No application crash

---

### Test 6.2: Missing Customer Data

**Objective:** Handle missing Stripe customer

**Steps:**
1. Trigger webhook with customer ID that doesn't exist in DB
2. Check logs

**Expected Results:**
- [YOURS] Error caught by try/catch
- [YOURS] Error logged: "Failed to send payment succeeded email"
- [YOURS] Webhook processing continues
- [YOURS] No application crash

---

### Test 6.3: Invoice With No Line Items

**Objective:** Handle invoice with empty lines array

**Steps:**
1. Trigger `invoice.paid` with an invoice that has no line items
2. Check email content

**Expected Results:**
- [YOURS] `planName` is undefined → generic body message used
- [YOURS] Billing period not shown (start/end are undefined)
- [YOURS] Email still sends with amount and date
- [YOURS] No errors or "undefined" text in email

---

### Test 6.4: Subscription Without Price Data

**Objective:** Handle subscriptions missing price info

**Steps:**
1. Trigger: `stripe trigger customer.subscription.deleted`
2. Check if plan name defaults correctly

**Expected Results:**
- [YOURS] Plan name defaults to "Plan" if product name missing
- [YOURS] Email still sends
- [YOURS] No errors in email rendering

---

## Test Suite 7: Data Accuracy

### Test 7.1: Amount Conversion

**Objective:** Verify cents to dollars conversion

**Steps:**
1. Trigger `invoice.paid` webhook
2. Note amount in Stripe (cents)
3. Check email amount (dollars)

**Expected Results:**
- [YOURS] Stripe: 2900 cents → Email: $29.00
- [YOURS] Conversion accurate (divide by 100)
- [YOURS] Two decimal places maintained

---

### Test 7.2: Timestamp Conversion

**Objective:** Verify Unix timestamp to Date conversion

**Steps:**
1. Trigger webhook with timestamp
2. Check email date
3. Verify timezone handling

**Expected Results:**
- [YOURS] Unix timestamp (seconds) converted correctly
- [YOURS] Date matches webhook timestamp
- [YOURS] Timezone handled appropriately
- [YOURS] No off-by-one date errors

---

### Test 7.3: Trial Days Remaining Calculation

**Objective:** Verify days remaining math

**Steps:**
1. Trigger: `stripe trigger customer.subscription.trial_will_end`
2. Check email content
3. Verify calculation

**Expected Results:**
- [YOURS] Days remaining calculated correctly
- [YOURS] Ceil function used (always rounds up)
- [YOURS] Shows 3 days for Stripe's 3-day notice
- [YOURS] No negative days

---

### Test 7.4: Billing Period Extraction

**Objective:** Verify billing period dates from invoice line items

**Steps:**
1. Trigger `invoice.paid` with a subscription invoice
2. Check billing period in email

**Expected Results:**
- [YOURS] Billing period start/end extracted from `invoice.lines.data[0].period`
- [YOURS] Dates formatted with "short" format (e.g., "Jan 31, 2026")
- [YOURS] Period only shown when both start and end are available
- [YOURS] Not shown for invoices without period data

---

## Test Suite 8: URL Generation

### Test 8.1: All URLs Use English Locale

**Objective:** Verify all email URLs use hardcoded "en" locale

**Expected URLs:**

| Email Type | Button | URL |
|------------|--------|-----|
| Payment Failed | Update Payment Method | `http://localhost:3000/en/settings/billing` |
| Subscription Renewed | Manage Subscription | `http://localhost:3000/en/settings/billing` |
| Subscription Cancelled | Resubscribe | `http://localhost:3000/en/pricing` |
| Trial Expiring | Continue Subscription | `http://localhost:3000/en/settings/billing` |
| Payment Succeeded | View Invoice | Stripe hosted invoice URL |

**Steps:**
1. Trigger each email type
2. Inspect button URLs in email source

**Expected Results:**
- [YOURS] All URLs contain `/en/` locale prefix
- [YOURS] URLs are clickable and point to correct pages
- [YOURS] Invoice URL comes from Stripe (not constructed)

---

## Test Suite 9: TypeScript & Code Quality

### Test 9.1: Type Checking

**Objective:** Verify no type errors

**Steps:**
```bash
npx tsc --noEmit
```

**Expected Results:**
- [YOURS] No TypeScript errors
- [YOURS] `PaymentType` imported from `@/services/billing` (not locally defined)
- [YOURS] No `as` type assertions in modified files
- [YOURS] No `any` types used
- [YOURS] Proper `import type` for type-only imports

---

## Final Checklist

Before marking feature complete, verify:

**Email Templates:**
- [ ] All 5 templates render without errors
- [ ] All templates have correct styling (grey/red/green/yellow/blue)
- [ ] Payment succeeded shows plan name, billing period, payment type message
- [ ] All subjects from translation files (no hardcoded strings)

**Payment Type Detection:**
- [ ] `subscription_create` → "initial" → welcome message
- [ ] `subscription_cycle` → "renewal" → renewal message
- [ ] `subscription_update` → "upgrade" → upgrade message
- [ ] Other billing reasons → "prorated" → generic plan message
- [ ] No plan name → generic body without plan reference

**Notification Preferences:**
- [ ] Enabled preferences allow all emails
- [ ] Disabled preferences skip non-critical emails
- [ ] Payment failed ALWAYS sends (critical)

**Webhook Integration:**
- [ ] All 5 webhook events handled
- [ ] Webhooks return 200 OK
- [ ] Email failures don't break webhooks

**Data Accuracy:**
- [ ] Currency conversion correct (cents → dollars)
- [ ] Timestamp conversion correct (Unix → Date)
- [ ] Billing period extracted from invoice line items
- [ ] All amounts/dates formatted correctly

**Translation Files:**
- [ ] All new keys in `messages/en/email.json`
- [ ] All new keys in `messages/de/email.json`
- [ ] Keys match between locales

**URLs:**
- [ ] All button links use `/en/` locale
- [ ] URLs point to correct pages

**Code Quality:**
- [ ] TypeScript compilation passes
- [ ] No `as` assertions, no `any` types
- [ ] `PaymentType` in shared billing types file

---

## Troubleshooting Guide

### Issue: Email not sending

**Check:**
1. Resend API key configured?
2. FROM_EMAIL set correctly?
3. User email exists in database?
4. Check console logs for errors

### Issue: Email skipped when should send

**Check:**
1. Notification preferences in database
2. Is it a critical email? (payment_failed should never skip)
3. `canSendEmail` function working?

### Issue: Webhook returns error

**Check:**
1. Stripe signature verification
2. Customer exists in database?
3. Database connection working?
4. Check webhook handler logs

### Issue: Wrong body message for payment type

**Check:**
1. Stripe invoice `billing_reason` value
2. `determinePaymentType()` switch cases in `webhook-handlers.ts`
3. Translation keys: `bodyInitial`, `bodyRenewal`, `bodyUpgrade`, `bodyWithPlan`
4. Is `planName` undefined? (falls back to generic `body`)

---

## Notes for Production Testing

When testing in production:

1. **Use Stripe test mode first**
2. **Test with real user account**
3. **Verify production webhook endpoint configured**
4. **Check production Resend dashboard**
5. **Monitor production logs**
6. **Test all email types before going live**
7. **Have rollback plan ready**
8. **Verify all 4 payment types produce correct messages**

Remember: Payment failed emails are CRITICAL - test thoroughly in production before launch!
