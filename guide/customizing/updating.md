---
title: "Updating JumpSaaS Template"
---

# Updating JumpSaaS Template

This guide explains how to receive and apply updates to your JumpSaaS template after you've started customizing it.

---

## Quick Start

Choose your update method based on your situation:

| Situation | Method | Risk | Time |
|-----------|--------|------|------|
| Early project, minor update | Git merge | Low | 5-15 min |
| Production app, minor update | Migration guide | Very low | 30-60 min |
| Production app, major update | Migration guide | Medium | 1-3 hours |
| Framework upgrade (Next.js 15→16) | Migration guide | High | 2-4 hours |

---

## Method 1: Git Merge (Automatic)

### When to use:
- [YOURS] Small updates (patches, bug fixes)
- [YOURS] Early-stage projects (not yet in production)
- [YOURS] You're comfortable with git and merge conflicts
- [YOURS] You want the fastest update method

### Initial Setup (one-time):

After cloning the template, set up the upstream remote:

```bash
# Navigate to your project
cd my-saas-app

# Rename current remote to 'template'
git remote rename origin template

# Add your own repo as 'origin'
git remote add origin git@github.com:yourusername/my-saas-app.git

# Configure branch tracking
git branch --set-upstream-to=origin/master master

# Verify remotes
git remote -v
# Should show:
# origin    git@github.com:yourusername/my-saas-app.git (fetch)
# origin    git@github.com:yourusername/my-saas-app.git (push)
# template  git@github.com:juuuuump/jumpersaas.git (fetch)
# template  git@github.com:juuuuump/jumpersaas.git (push)
```

### Applying Updates:

```bash
# 1. Commit any uncommitted changes
git status
git add .
git commit -m "Save work before template update"

# 2. Create a backup branch (recommended)
git branch backup-before-update

# 3. Fetch latest template changes
git fetch template

# 4. Review what's changed (optional but recommended)
git log HEAD..template/master --oneline

# 5. Merge template updates
git merge template/master

# 6. If there are NO conflicts:
#    You're done! Skip to step 9.

# 7. If there ARE conflicts:
#    Git will list conflicting files
#    Open each file and resolve conflicts:
#    - Look for <<<<<<< HEAD markers
#    - Keep your changes, template changes, or both
#    - Remove conflict markers

# 8. After resolving conflicts:
git add .
git commit -m "Merge template v1.1 updates"

# 9. Test thoroughly before deploying
npm install           # Update dependencies
npm run build         # Check for build errors
npm run type-check    # Check TypeScript errors
npm run dev           # Test locally
```

### Handling Conflicts:

```bash
# If conflicts are too complex or numerous:
git merge --abort

# Then use Method 2 (Migration Guide) instead
```

### Rolling Back:

```bash
# If update breaks something:
git reset --hard backup-before-update

# Or if you didn't create a backup:
git log  # Find commit hash before merge
git reset --hard <commit-hash>
```

---

## Method 2: Migration Guide (Manual)

### When to use:
- [YOURS] Production applications
- [YOURS] Major updates (v1.0 → v2.0)
- [YOURS] You want full control over changes
- [YOURS] Git merge produced too many conflicts
- [YOURS] You prefer understanding every change

### Process:

1. **Read the migration guide thoroughly**
   - Available in update announcement email
   - Or at: https://docs.jumpsaas.com/migrations/
   - Example: `docs/migrations/v1.0-to-v1.1.md`

2. **Review code diffs**
   - Compare template changes on GitHub
   - Understand what changed and why
   - Identify which changes you need

3. **Apply changes file by file**
   - Copy new files from template
   - Update existing files (use diffs as reference)
   - Modify if needed to fit your customizations

4. **Test each change**
   - Run `npm run build` after each major change
   - Test affected functionality
   - Commit incrementally

5. **Run full test suite**
   ```bash
   npm install
   npm run build
   npm run type-check
   npm run db:push  # If schema changed
   npm run dev
   ```

### Example Migration:

See migration guide template in your welcome email or at:
- `docs/migrations/v1.0-to-v1.1.md` (example)

---

## Method 3: Fresh Start (Clean Slate)

### When to use:
- [YOURS] Major version jump (v1 → v2)
- [YOURS] Template architecture significantly changed
- [YOURS] You have minimal custom code
- [YOURS] Starting fresh is faster than merging

### Process:

1. **Clone the new template version**
   ```bash
   git clone git@github.com:juuuuump/jumpersaas.git my-saas-v2
   cd my-saas-v2
   git checkout v2.0  # Or latest version tag
   ```

2. **Manually port your custom features**
   - Review your old project's custom code
   - Copy business logic to new structure
   - Adapt to new architecture if needed

3. **Compare old vs new**
   - Use a diff tool to compare directories
   - Ensure no custom code is lost
   - Test all custom features

4. **Test everything from scratch**
   - Full QA pass
   - Verify all integrations work
   - Deploy to staging first

---

## Best Practices

### Before Updating:

- [YOURS] **Read changelog completely** - Know what's changing
- [YOURS] **Back up your code** - Create git branch or push to remote
- [YOURS] **Test on staging first** - Never update production directly
- [YOURS] **Schedule during low-traffic** - Minimize user impact
- [YOURS] **Review migration guide** - Understand required steps
- [YOURS] **Check dependencies** - Ensure compatible versions

### After Updating:

- [YOURS] **Run build checks**
  ```bash
  npm run build
  npm run type-check
  npm run lint
  ```

- [YOURS] **Test critical paths**
  - Authentication (sign up, sign in, sign out)
  - Billing (checkout, subscription, invoices)
  - Core features (your custom functionality)
  - Email sending
  - File uploads

- [YOURS] **Check console for warnings**
  - Browser console (F12)
  - Server logs
  - Build warnings

- [YOURS] **Monitor errors**
  - Check Sentry or error tracking
  - Watch for new errors after deployment
  - Be ready to rollback if needed

- [YOURS] **Update documentation**
  - Update your project's README if needed
  - Document any customizations you made during update

### Staying Updated:

- ⭐ **Watch template repo** - Get notified of updates
- 📧 **Read update emails** - Don't skip announcements
- 💬 **Join Discord** - #announcements and #updates channels
- 📰 **Check changelog** - Review monthly: `CHANGELOG.md`
- 🔔 **Enable notifications** - GitHub watch for releases only

---

## Troubleshooting

### Build Errors After Update

```bash
# Clear caches and reinstall
rm -rf .next node_modules package-lock.json
npm install
npm run build
```

### TypeScript Errors

```bash
# Regenerate types
npm run db:generate
npm run type-check

# Check for type conflicts
# Update your custom code to match new types
```

### Database Issues

```bash
# Apply migrations
npm run db:push

# If that fails, check schema changes:
git diff template/master -- src/db/schema/
```

### Environment Variable Changes

```bash
# Compare .env.example files
git diff template/master -- .env.example

# Add any new required variables to your .env
```

### Dependency Conflicts

```bash
# Check what changed
git diff template/master -- package.json

# Update your dependencies
npm update

# Or reinstall everything
rm -rf node_modules package-lock.json
npm install
```

### Merge Conflicts

```bash
# To see conflicting files:
git status

# To abort merge and try manual method:
git merge --abort

# To get help with conflicts:
# 1. Open file in editor
# 2. Look for <<<<<<< markers
# 3. Keep your code, template code, or both
# 4. Remove all conflict markers
# 5. Test the file
# 6. git add <file>
# 7. git commit
```

### Still Stuck?

**Get Help:**
- 📖 Check migration guide: Detailed steps in update email
- 💬 Ask in Discord: #support channel
- 📧 Email support: support@jumpsaas.com (24-48h response)
- 🎥 Watch video: Update walkthrough in members area

**Provide When Asking:**
- What you're trying to update from/to (versions)
- What method you tried (git merge, migration guide)
- Full error message or issue description
- What you've already tried

---

## Update Schedule

### What to Expect:

**Patch Updates (v1.0.x)**
- **Frequency:** Weekly or as-needed
- **Size:** Small (1-5 files changed)
- **Risk:** Very low
- **Example:** Bug fixes, security patches

**Minor Updates (v1.x.0)**
- **Frequency:** Monthly
- **Size:** Medium (10-30 files changed)
- **Risk:** Low (backward compatible)
- **Example:** New features, new examples

**Major Updates (vX.0.0)**
- **Frequency:** Quarterly or for framework updates
- **Size:** Large (50+ files changed)
- **Risk:** Medium-High (may require code changes)
- **Example:** Next.js upgrade, architecture changes

### Notification Methods:

You'll be notified of updates via:
1. **Email** - Detailed announcement with changelog
2. **Discord** - Immediate notification in #announcements
3. **GitHub** - Release notes and version tags
4. **In-app** - (Future) Update notification in admin dashboard

---

## Version Branches

For production apps requiring stability, you can track stable branches:

```bash
# Template maintains version branches:
master               # Latest development
v1.0-stable         # Frozen v1.0 (security patches only)
v1.1-stable         # Frozen v1.1 (security patches only)

# Track stable version instead of main:
git remote add template git@github.com:juuuuump/jumpersaas.git
git fetch template
git merge template/v1.0-stable  # Only get security fixes
```

**Benefits:**
- More predictable updates
- Only security patches, no new features
- Less risk of breaking changes

**Trade-off:**
- Miss out on new features
- Need to manually upgrade to newer stable versions

---

## Tips for Easier Updates

### 1. Keep Customizations Organized

```
src/
├── app/
│   └── (custom)/          # Your custom routes here
├── components/
│   └── custom/            # Your custom components here
├── services/
│   └── custom/            # Your business logic here
```

Benefits:
- Fewer merge conflicts in core files
- Easier to see what's yours vs template
- Faster to port to new versions

### 2. Document Your Changes

Keep a `CUSTOMIZATIONS.md` file:

```markdown
# Customizations to JumpSaaS Template

## Modified Files
- src/components/ui/button.tsx - Added custom variant "brand"
- src/lib/auth/server.ts - Added custom auth callback

## Added Files
- src/app/(custom)/ - All custom routes
- src/services/custom/ - Custom business logic

## Deleted Files
- src/app/(examples)/ - Removed example features
```

### 3. Use Feature Flags

```bash
# .env - Enable/disable template features
ENABLE_ADMIN_DASHBOARD=true
ENABLE_EXAMPLE_FEATURES=false
ENABLE_CUSTOM_FEATURE=true
```

Benefits:
- Can disable template features you don't use
- Easier to update if features are modular
- Less code to maintain

### 4. Test in Staging

Always test updates in a staging environment:

```bash
# Deploy to staging
vercel --prod --env=staging

# Test thoroughly
# Then deploy to production
vercel --prod
```

---

## FAQ

**Q: Do I have to update?**
A: No, updates are optional. Update when you're ready. However, security patches are strongly recommended.

**Q: What if I skip several updates?**
A: You can jump versions (e.g., v1.0 → v1.3), but review all migration guides between versions.

**Q: Will updates break my app?**
A: Minor updates (v1.x) are backward compatible. Major updates (vX.0) may require code changes but include migration guides.

**Q: Can I cherry-pick specific updates?**
A: Yes! With git merge, you can cherry-pick individual commits:
```bash
git cherry-pick <commit-hash>
```

**Q: How long is each version supported?**
A: Major versions receive security patches for 1 year after next major version releases.

**Q: What if I've heavily customized the template?**
A: Use migration guides instead of git merge. Port changes manually at your own pace.

**Q: Can I get help with updates?**
A: Yes! Priority support includes update assistance. Email support@jumpsaas.com or ask in Discord.

---

## Related Documentation

- [Architecture — Next.js](../reference/architecture/nextjs) · [TanStack](../reference/architecture/tanstack) - Understanding template structure
- [Adding Features — Next.js](adding-features/nextjs) · [TanStack](adding-features/tanstack) - Guide for adding custom features

---

**Last Updated:** February 2, 2026
**Template Version:** v1.0.0
