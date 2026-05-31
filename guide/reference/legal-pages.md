---
title: "Legal Pages Setup Guide"
---

# Legal Pages Setup Guide

> **[CAUTION] IMPORTANT**: These are placeholder templates. You MUST customize them before going live.

## Quick Start Checklist

- [ ] Replace all `[COMPANY_NAME]` placeholders
- [ ] Replace all `[SERVICE_NAME]` and `[SERVICE_DESCRIPTION]` placeholders
- [ ] Set up legal email addresses (`[LEGAL_EMAIL]`, `[PRIVACY_EMAIL]`)
- [ ] Update company address `[COMPANY_ADDRESS]`
- [ ] Review and customize all legal content for your service
- [ ] Have a lawyer review your final legal pages
- [ ] Update the `LAST_UPDATED` date in each page component
- [ ] Remove or customize the orange warning banner

## What's Included

JumpSaaS includes three template legal pages:
- **Terms of Service** (`/legal/terms`) - User agreement and service terms
- **Privacy Policy** (`/legal/privacy`) - GDPR-compliant data protection policy
- **Cookie Policy** (`/legal/cookies`) - Cookie usage disclosure

## File Structure

**Next.js:**
```
src/app/[locale]/(legal)/
├── layout.tsx                              # Shared layout
└── legal/
    ├── _components/
    │   └── template-warning.tsx
    ├── terms/page.tsx
    ├── privacy/page.tsx
    └── cookies/page.tsx

messages/en/legal.json                      # Legal content (EN)
messages/de/legal.json                      # Legal content (DE)
```

**TanStack:**
```
src/routes/{-$locale}/
└── _legal.tsx                              # Shared layout route
    └── legal/
        ├── terms.tsx
        ├── privacy.tsx
        └── cookies.tsx

messages/en.json                            # Legal content under "legal" key
messages/de.json
```

## Customization Steps

### Step 1: Replace Placeholders in Translation Files

Edit `messages/en/legal.json` and `messages/de/legal.json`:

**Company Information:**
```json
"placeholders": {
    "companyName": "Your Company Inc.",           // Was: [COMPANY_NAME]
    "serviceName": "YourApp",                     // Was: [SERVICE_NAME]
    "serviceDescription": "project management",    // Was: [SERVICE_DESCRIPTION]
    "companyEmail": "support@yourapp.com",        // Was: [COMPANY_EMAIL]
    "legalEmail": "legal@yourapp.com",            // Was: [LEGAL_EMAIL]
    "privacyEmail": "privacy@yourapp.com",        // Was: [PRIVACY_EMAIL]
    "companyAddress": "123 Main St, City, Country" // Was: [COMPANY_ADDRESS]
}
```

**Additional Placeholders to Replace:**
- `[YOUR_JURISDICTION]` - e.g., "California, USA" or "Germany"
- `[DATA_PROCESSING_LOCATION]` - e.g., "United States" or "European Union"
- `[IF USING ANALYTICS]` - Remove if not using, or specify your analytics provider
- `[ADD OTHER THIRD PARTIES IF APPLICABLE]` - List additional third parties

### Step 2: Customize Legal Content

**Review each section** in `messages/en/legal.json` and `messages/de/legal.json`:

1. **Terms of Service:**
   - Add/remove sections based on your service
   - Specify refund policy details
   - Update acceptable use restrictions
   - Add industry-specific terms if needed

2. **Privacy Policy:**
   - List all data you actually collect
   - Specify exact third-party services you use
   - Update data retention periods if different from 30 days
   - Add jurisdiction-specific requirements (CCPA, etc.)

3. **Cookie Policy:**
   - List cookies you actually use (check browser dev tools)
   - Update third-party cookie providers
   - Remove analytics section if not using analytics

### Step 3: Update Footer Copyright

Edit `messages/en/footer.json` and `messages/de/footer.json`:

```json
{
    "footer": {
        "copyright": "© {year} Your Company Inc. All rights reserved."
    }
}
```

### Step 4: Update Last Updated Date

In each page component (Next.js: `terms/page.tsx`; TanStack: `terms.tsx`):

```typescript
// Change this date when you finalize your legal content
const LAST_UPDATED = "2026-02-15";  // Use your actual date
```

### Step 5: Remove Template Warning (Optional)

Once content is finalized, you can:

**Option A: Remove the warning banner entirely**
```typescript
// Delete this line from each page:
<TemplateWarning />
```

**Option B: Customize the warning message**

Edit `src/app/[locale]/(legal)/legal/_components/template-warning.tsx` to show a different message.

### Step 6: Set Up Email Addresses

Create and monitor these email addresses:
- Legal inquiries: `legal@yourcompany.com`
- Privacy/GDPR requests: `privacy@yourcompany.com`
- General support: `support@yourcompany.com`

Configure auto-responders or ticketing system to ensure timely responses to GDPR requests (required within 30 days).

## Adding More Languages

To add legal pages in additional languages (e.g., French):

**Next.js:** Create `messages/fr/legal.json` and `messages/fr/footer.json` — next-intl picks up the new locale automatically.

**TanStack:** Add French keys to `messages/fr.json` under the `legal` and `footer` namespaces, then follow the [ParaglideJS language setup](../customizing/conventions#adding-a-new-language).

## Legal Compliance Checklist

### GDPR (European Users)

- [ ] Privacy Policy explains data collection clearly
- [ ] User rights section includes all GDPR rights
- [ ] Data retention policy specified
- [ ] Contact information for privacy inquiries
- [ ] Legal basis for processing specified
- [ ] Information about international data transfers
- [ ] Cookie consent mechanism (if needed)

### CCPA (California Users)

- [ ] "Do Not Sell My Personal Information" link (if applicable)
- [ ] Categories of personal information disclosed
- [ ] Right to deletion mentioned
- [ ] Right to opt-out mentioned

### General Best Practices

- [ ] Terms are written in clear, plain language
- [ ] Dispute resolution process specified
- [ ] Limitation of liability clause
- [ ] Indemnification clause (if needed)
- [ ] Intellectual property rights specified
- [ ] Termination conditions clear

## Getting Legal Review

**[CAUTION] STRONGLY RECOMMENDED**: Have a lawyer review your legal pages before launch.

**Options:**
1. **Hire a lawyer** specializing in tech/SaaS law
2. **Legal template services:**
   - [Termly](https://termly.io/) - Automated policy generator
   - [iubenda](https://www.iubenda.com/) - Privacy policy generator
   - [TermsFeed](https://www.termsfeed.com/) - Legal document generator
3. **Legal marketplaces:**
   - UpCounsel
   - LegalZoom
   - Rocket Lawyer

**Cost:** $500-$2,000 for basic review, $2,000-$10,000 for comprehensive legal package

## Common Customization Scenarios

### Scenario 1: B2B SaaS

Add to Terms of Service:
- Data Processing Agreement (DPA) section
- Service Level Agreement (SLA) commitments
- Enterprise support terms
- Custom contract provisions link

### Scenario 2: Marketplace/Platform

Add to Terms of Service:
- Seller/provider terms
- Buyer/user terms
- Transaction fees and payment terms
- Intellectual property rights for user content
- Dispute resolution between users

### Scenario 3: AI/ML Service

Add to Privacy Policy:
- How AI models use customer data
- Training data retention
- Model output ownership
- Accuracy disclaimers

Add to Terms:
- AI-specific acceptable use restrictions
- Output accuracy disclaimers

### Scenario 4: Healthcare/Finance (Regulated Industries)

- Add HIPAA compliance section (healthcare)
- Add PCI DSS compliance (if handling payments directly)
- Add industry-specific regulatory compliance
- Consider additional legal review requirements

## Testing Your Legal Pages

Before launch:

1. **Read through completely** - Make sure everything makes sense
2. **Check all links** - Ensure email links work
3. **Test all languages** - Verify translations are accurate
4. **Mobile test** - Ensure pages are readable on mobile
5. **Print test** - Some users may print legal pages
6. **Accessibility test** - Run screen reader to check compliance

## Maintenance

**When to update legal pages:**
- Adding new features that collect data
- Changing third-party service providers
- Expanding to new jurisdictions
- Legal regulation changes
- Significant business model changes

**Best practice:** Review legal pages every 6-12 months.

## FAQ

**Q: Can I just use these templates as-is?**
A: No. These are examples only. You must customize them for your specific service and have them reviewed by a lawyer.

**Q: Do I need all three pages?**
A: Yes, for GDPR compliance. Some jurisdictions may require additional pages.

**Q: What if I'm not in the EU? Do I need GDPR compliance?**
A: If you have ANY users in the EU, yes. GDPR applies to data controllers regardless of location.

**Q: Can I copy legal pages from other companies?**
A: No. Each company's legal pages should reflect their specific practices and jurisdiction. Copying may be copyright infringement.

**Q: How do I know if my legal pages are GDPR compliant?**
A: Hire a lawyer familiar with GDPR. Automated tools can help but aren't substitutes for legal advice.

## Resources

- [GDPR Official Text](https://gdpr-info.eu/)
- [CCPA Official Text](https://oag.ca.gov/privacy/ccpa)
- [Stripe Privacy Best Practices](https://stripe.com/docs/security/guide#privacy-and-data-collection)
- [Google GDPR Resources](https://privacy.google.com/businesses/compliance/)

## Support

For questions about customizing these templates, see the main project documentation or open an issue on GitHub.

**Remember:** These templates are not legal advice. Consult with a qualified attorney before using legal pages in production.
