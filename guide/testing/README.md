---
title: "Testing Documentation"
---

# Testing Documentation

This directory contains manual testing guides and test cases for features in JumpSaaS.

## Available Test Guides

### [Billing Email Notifications](./billing-email-notifications)
Comprehensive manual testing cases for the billing email notification system including:
- Email template rendering (5 templates)
- Notification preference handling
- Webhook integration
- Internationalization (EN/DE)
- Edge cases and error handling

## Directory Structure

```
docs/testing/
├── README.md                           # This file
└── billing-email-notifications.md      # Billing email tests
```

## Testing Philosophy

- **Manual tests** documented here provide step-by-step verification
- **Automated tests** live in the project's test directories
- Each feature should have its own testing guide
- Focus on user-facing functionality and critical paths

## Adding New Test Documentation

When adding a new manual testing guide:

1. Create a file: `[feature-name].md`
2. Include test suites organized by concern
3. Provide clear expected results
4. Add troubleshooting section
5. Update this README with a link

## Related Documentation

- [Architecture — Next.js](../reference/architecture/nextjs) · [TanStack](../reference/architecture/tanstack)
- [Billing System](../reference/billing)
- [Email System](../integrations/email)
