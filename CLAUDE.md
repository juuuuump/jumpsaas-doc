# jumpsaas-doc

Markdown-only documentation repo. `guide/` is synced into `jumpsaas-web` via `scripts/sync-docs.mjs` — never edit docs inside the template repos or `jumpsaas-web` directly.

## When to split vs inline

- **Split into `topic/nextjs.md` + `topic/tanstack.md`** — when framework differences dominate (>40% of content, code walkthrough differs step by step). Current examples: `customizing/adding-features/`, `reference/architecture/`.
- **Inline callout** — isolated framework-specific sections in an otherwise shared doc. Use `> **Next.js only.**` or labelled `**Next.js:** / **TanStack:**` code blocks.
- **No labels** — when content is identical for both frameworks.

## Non-obvious gotchas

**Link depth** — files two levels deep need `../../` to reach top-level sections, not `../`:

| Location | To `customizing/` | To `reference/` | To `integrations/` |
|---|---|---|---|
| `guide/customizing/*.md` | `foo` | `../reference/foo` | `../integrations/foo` |
| `guide/customizing/adding-features/*.md` | `../../customizing/foo` | `../../reference/foo` | `../../integrations/foo` |
| `guide/reference/architecture/*.md` | `../../customizing/foo` | `../foo` | `../../integrations/foo` |
| `guide/getting-started/nextjs/*.md` | `../../customizing/foo` | `../../reference/foo` | `../../integrations/foo` |

**Shiki** — use ` ```bash ` for `.env` files. ` ```env ` is not in the bundle and crashes the build.

**meta.json** — adding a file or directory without updating the parent `meta.json` pages array means it won't appear in the sidebar.

**After editing** — run `node scripts/sync-docs.mjs` from `jumpsaas-web` to push changes into the site.
