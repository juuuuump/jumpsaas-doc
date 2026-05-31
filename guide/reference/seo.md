---
title: "SEO Implementation Guide"
---

# SEO Implementation Guide

## Overview

JumpSaaS includes comprehensive SEO support with:
- Internationalized metadata for all pages
- Dynamic sitemap with locale support
- Robots.txt configuration
- OpenGraph images for social sharing

## Metadata System

> **Framework note:** Page metadata APIs differ significantly between frameworks. See your framework's section below.

### Adding Metadata to Pages

**Next.js** uses the `generateMetadata` export — a special async function recognized by the App Router. Use the `generatePageMetadata` utility for consistent, localized metadata:

```typescript
import { generatePageMetadata } from "@/lib/seo";

interface PageProps {
  params: Promise<{ locale: string }>;
}

// Next.js App Router — this export is called at request/build time
export async function generateMetadata({ params }: PageProps) {
  const { locale } = await params;
  return generatePageMetadata({
    locale,
    namespace: "yourNamespace.meta",
    path: "/your-path",
  });
}
```

**TanStack** uses the `head` option in `createFileRoute` to set page metadata:

```typescript
import { createFileRoute } from "@tanstack/react-router";
import * as m from "@/paraglide/messages";

export const Route = createFileRoute("/{-$locale}/_app/app/your-page")({
  head: () => ({
    meta: [
      { title: m.yourNamespace_meta_title() },
      { name: "description", content: m.yourNamespace_meta_description() },
    ],
  }),
  component: YourPage,
});
```

### Translation Structure

Add metadata translations to your namespace JSON file:

```json
{
  "yourNamespace": {
    "meta": {
      "title": "Page Title - JumpSaaS",
      "description": "Page description for SEO"
    }
  }
}
```

## Sitemap

### Next.js

The sitemap is automatically generated at `/sitemap.xml` via `src/app/sitemap.ts` (a Next.js App Router convention). To add your routes:

```typescript
// src/app/sitemap.ts
const routes: Route[] = [
  ...existing routes...,
  { path: "/new-page", changeFrequency: "weekly", priority: 0.7 },
];
```

### TanStack

TanStack Start doesn't have a built-in sitemap convention. Serve a static `public/sitemap.xml` or create an API route:

```typescript
// src/routes/api/sitemap.xml.ts
import { createAPIFileRoute } from "@tanstack/react-start/api";

export const APIRoute = createAPIFileRoute("/api/sitemap.xml")({
  GET: () => {
    const xml = `<?xml version="1.0" encoding="UTF-8"?>
<urlset xmlns="http://www.sitemaps.org/schemas/sitemap/0.9">
  <url><loc>https://yourdomain.com/en</loc></url>
  <!-- add your routes -->
</urlset>`;
    return new Response(xml, { headers: { "Content-Type": "application/xml" } });
  },
});
```

## Robots.txt

### Next.js

Edit `src/app/robots.ts` (a Next.js App Router file convention):

```typescript
export default function robots(): MetadataRoute.Robots {
  return {
    rules: [
      {
        userAgent: "*",
        allow: ["/", "/your-public-path/"],
        disallow: ["/settings/", "/your-private-path/"],
      },
    ],
    sitemap: `${APP_URL}/sitemap.xml`,
  };
}
```

### TanStack

Place a static `public/robots.txt` file, or serve it from an API route:

```typescript
// src/routes/api/robots.txt.ts
import { createAPIFileRoute } from "@tanstack/react-start/api";

export const APIRoute = createAPIFileRoute("/api/robots.txt")({
  GET: () =>
    new Response("User-agent: *\nAllow: /\nDisallow: /settings/\n", {
      headers: { "Content-Type": "text/plain" },
    }),
});
```

## OpenGraph Images

OpenGraph images are generated dynamically using Next.js Image Response API.

### Creating New OG Images

> **Next.js only.** OG image generation uses `next/og` (`ImageResponse`), which is a Next.js primitive. TanStack users should generate OG images via a standard API route returning a PNG — a common approach is [`@vercel/og`](https://vercel.com/docs/functions/og-image-generation) as a standalone package, or serve static OG images from `public/`.

Create `opengraph-image.tsx` in your route directory:

```tsx
import { ImageResponse } from "next/og";

export const runtime = "edge";
export const alt = "Your page description";
export const size = { width: 1200, height: 630 };
export const contentType = "image/png";

export default async function Image() {
  return new ImageResponse(
    (
      <div style={{ /* your design */ }}>
        Your OG Image Content
      </div>
    ),
    { ...size }
  );
}
```

## Environment Variables

The `NEXT_PUBLIC_APP_URL` variable is critical for SEO:

```bash
# Development
NEXT_PUBLIC_APP_URL=http://localhost:3000

# Production
NEXT_PUBLIC_APP_URL=https://www.jumpsaas.com
```

**Important:** This must be set in `.env.production` BEFORE building, as it's baked into the bundle at build time.

## Testing SEO

### Local Testing

```bash
# View sitemap
curl http://localhost:3000/sitemap.xml

# View robots.txt
curl http://localhost:3000/robots.txt

# View OG image
open http://localhost:3000/en/opengraph-image
```

### Production Validation

1. Google Search Console: Submit sitemap
2. Social media debuggers:
   - Twitter: https://cards-dev.twitter.com/validator
   - Facebook: https://developers.facebook.com/tools/debug/
   - LinkedIn: https://www.linkedin.com/post-inspector/

## Best Practices

1. **Keep titles under 60 characters** for optimal display in search results
2. **Keep descriptions between 150-160 characters**
3. **Use unique metadata for each page** - avoid duplicates
4. **Update sitemap when adding public routes**
5. **Test OG images** before deploying to ensure proper rendering
6. **Include alt text** for all OpenGraph images

## Related Documentation

- Internationalization
- Environment Variables
- [Next.js Metadata API](https://nextjs.org/docs/app/api-reference/functions/generate-metadata)
