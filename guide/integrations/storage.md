---
title: "Storage"
---

# Storage

JumpSaaS uses **better-upload** for file uploads with Cloudflare R2 (default) or AWS S3 as the storage backend.

## Setup

### Storage Providers

- **Cloudflare R2** (Default): S3-compatible storage with zero egress fees and a built-in `cloudflare()` helper
- **AWS S3**: Production-ready object storage (requires custom client configuration)

### Cloudflare R2

#### Environment Variables

Add these to your `.env` file:

```bash
CLOUDFLARE_ACCOUNT_ID=your-cloudflare-account-id
STORAGE_BUCKET=your-bucket-name
STORAGE_ACCESS_KEY_ID=your-r2-access-key-id
STORAGE_SECRET_ACCESS_KEY=your-r2-secret-access-key

# IMPORTANT: You must configure one of these public URL options:

# Option A: R2.dev public subdomain (from your bucket's public access settings)
R2_PUBLIC_URL_DOMAIN=https://pub-8fbc3a6e8f854c94b2a0f839652c1103.r2.dev

# Option B: Custom public URL (R2 custom domain)
# STORAGE_PUBLIC_URL=https://cdn.yourdomain.com
```

#### Step 1: Find Your Account ID

1. Go to your Cloudflare Dashboard
2. Look at the URL: `https://dash.cloudflare.com/{YOUR_ACCOUNT_ID}/r2`
3. Copy the account ID from the URL

#### Step 2: Create R2 Bucket

1. In Cloudflare Dashboard, navigate to **R2 Object Storage**
2. Click **Create bucket**
3. Enter your bucket name (e.g., `jumpsaas`)
4. Choose a location hint (optional)
5. Click **Create bucket**

#### Step 3: Configure CORS Policy

CORS must be configured to allow browser uploads via signed URLs.

Navigate to: Bucket → Settings → CORS Policy

Add this CORS configuration:

```json
[
  {
    "AllowedOrigins": [
      "http://localhost:3000",
      "https://yourdomain.com",
      "https://*.yourdomain.com"
    ],
    "AllowedMethods": [
      "GET",
      "PUT",
      "POST",
      "DELETE",
      "HEAD"
    ],
    "AllowedHeaders": [
      "*"
    ],
    "ExposeHeaders": [
      "ETag"
    ],
    "MaxAgeSeconds": 3600
  }
]
```

Important:
- Replace `https://yourdomain.com` with your actual production domain
- Include all subdomains if needed (e.g., `www`, `app`, `cdn`)
- Keep `localhost:3000` for local testing, or remove it in production-only buckets
- `AllowedHeaders: ["*"]` is required for better-upload signed URL uploads

#### Step 4: Create API Token

1. Go to your R2 bucket list
2. Click **Manage R2 API Tokens**
3. Click **Create API token**
4. Configure the token:
   - **Token name:** Enter a descriptive name (e.g., `jumpsaas-production`)
   - **Permissions:** Select **Object Read & Write**
   - **Apply to buckets:** Select **Apply to specific buckets only** and choose your bucket
   - **TTL:** No expiry (or set expiry if rotating tokens)
5. Click **Create API token**
6. Copy and save both credentials immediately (shown only once):
   - Access Key ID
   - Secret Access Key

#### Step 5: Enable Public Access

To make uploaded files publicly accessible, choose one option:

**Option A: R2.dev Subdomain (Easiest)**

1. Go to your R2 bucket → Settings → **Public Access**
2. Click **Allow Access**
3. Enable the public R2.dev subdomain
4. Copy the R2.dev subdomain URL shown in settings (e.g., `https://pub-8fbc3a6e8f854c94b2a0f839652c1103.r2.dev`)
5. Add to your `.env` file: `R2_PUBLIC_URL_DOMAIN=https://pub-8fbc3a6e8f854c94b2a0f839652c1103.r2.dev`

**Note:** The R2.dev subdomain uses a bucket-specific public domain ID (not your account ID). Copy the exact URL from your bucket settings.

**Option B: Custom Domain (Recommended for Production)**

1. Go to your R2 bucket → Settings → **Public Access**
2. Click **Connect Domain**
3. Enter your domain (e.g., `cdn.yourdomain.com`)
4. Add the CNAME record to your DNS:
   ```
   CNAME  cdn  <your-bucket-id>.r2.cloudflarestorage.com
   ```
5. Wait for DNS propagation (a few minutes)
6. Set in `.env.production`: `STORAGE_PUBLIC_URL=https://cdn.yourdomain.com`

Benefits of custom domain: branded URLs, better SEO, no R2.dev rate limits, easier provider migration.

#### Step 6: Environment Variable Checklist

**Production `.env.production`:**

```bash
# R2 Configuration
CLOUDFLARE_ACCOUNT_ID=abc123def456                    # From dashboard URL
STORAGE_BUCKET=jumpsaas-prod                           # Your bucket name
STORAGE_ACCESS_KEY_ID=xxx                              # API token access key
STORAGE_SECRET_ACCESS_KEY=yyy                          # API token secret key

# Public URL (choose ONE)
# Option A: Custom domain (recommended)
STORAGE_PUBLIC_URL=https://cdn.yourdomain.com

# Option B: R2.dev subdomain
# R2_PUBLIC_URL_DOMAIN=https://pub-abc123.r2.dev
```

**Build-time note:** Public variables used for upload configuration are baked into the client bundle at build time — `NEXT_PUBLIC_` prefix in Next.js, `VITE_` prefix in TanStack. Changes require a full rebuild; set them correctly in `.env.production` before building.

#### Step 7: Set Bucket Lifecycle Rules (Optional)

Automatically delete incomplete multipart uploads to save costs.

1. Navigate to bucket → Settings → **Lifecycle Rules**
2. Add rule:
   - **Name:** Delete incomplete uploads
   - **Action:** Abort Incomplete Multipart Uploads
   - **Days after initiation:** 7
3. Click **Add Rule**

#### Security Best Practices

- Use separate buckets for dev/staging/production
- Use separate API tokens per environment
- Apply token permissions to specific buckets only
- Rotate API tokens periodically
- Never commit credentials to version control

### AWS S3

To use AWS S3 instead of R2, update your upload route with an S3 client.

#### Environment Variables

```bash
STORAGE_BUCKET=your-bucket-name
STORAGE_ACCESS_KEY_ID=your-access-key-id
STORAGE_SECRET_ACCESS_KEY=your-secret-access-key
AWS_REGION=us-east-1

# Optional: Custom public URL (e.g., CloudFront)
STORAGE_PUBLIC_URL=https://d111111abcdef8.cloudfront.net
```

#### Create S3 Bucket

```bash
aws s3 mb s3://your-bucket-name --region us-east-1
```

Or via the AWS Console:
1. Go to AWS S3 Console
2. Create a new bucket
3. Choose your preferred region
4. Disable "Block all public access" for public read access

#### Configure CORS

Create `cors.json`:

```json
{
  "CORSRules": [
    {
      "AllowedOrigins": ["https://yourdomain.com"],
      "AllowedMethods": ["GET", "PUT", "POST", "DELETE", "HEAD"],
      "AllowedHeaders": ["*"],
      "ExposeHeaders": ["ETag"],
      "MaxAgeSeconds": 3600
    }
  ]
}
```

Apply via CLI:

```bash
aws s3api put-bucket-cors --bucket your-bucket-name --cors-configuration file://cors.json
```

Or via the Console: Bucket → Permissions → Cross-origin resource sharing (CORS) → Edit.

#### Configure Bucket Policy for Public Read

Create `policy.json`:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "PublicReadGetObject",
      "Effect": "Allow",
      "Principal": "*",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::your-bucket-name/*"
    }
  ]
}
```

Apply via CLI:

```bash
aws s3api put-bucket-policy --bucket your-bucket-name --policy file://policy.json
```

Replace `your-bucket-name` with your actual bucket name.

#### Create IAM User

1. Go to AWS IAM → Users → Add User
2. User name: `jumpsaas-uploads`
3. Access type: Programmatic access
4. Attach this inline policy:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:PutObject",
        "s3:GetObject",
        "s3:DeleteObject"
      ],
      "Resource": "arn:aws:s3:::your-bucket-name/*"
    }
  ]
}
```

5. Save the Access Key ID and Secret Access Key to your `.env` file

#### Configure CloudFront CDN (Optional)

For faster global delivery:
1. Create a CloudFront distribution
2. Set origin to your S3 bucket
3. Set `STORAGE_PUBLIC_URL` to your CloudFront URL

#### Update Upload Route for S3

Replace the R2 client with an S3 client in your upload route:

```typescript
import { S3Client } from '@aws-sdk/client-s3';

const s3Client = new S3Client({
  region: process.env.AWS_REGION || 'us-east-1',
  credentials: {
    accessKeyId: process.env.STORAGE_ACCESS_KEY_ID!,
    secretAccessKey: process.env.STORAGE_SECRET_ACCESS_KEY!,
  },
});

const router: Router = {
  client: s3Client,
  bucketName: process.env.STORAGE_BUCKET || 'your-bucket',
  // ... rest of configuration
};
```

If using S3, update the upload helper in `src/lib/saas/upload/server.ts`:

```typescript
import { S3Client } from "@aws-sdk/client-s3";

export function createS3Client(): S3Client {
  const region = process.env.AWS_REGION || "us-east-1";
  const accessKeyId = process.env.AWS_ACCESS_KEY_ID;
  const secretAccessKey = process.env.AWS_SECRET_ACCESS_KEY;

  if (!accessKeyId || !secretAccessKey) {
    throw new Error("Missing AWS credentials");
  }

  return new S3Client({
    region,
    credentials: {
      accessKeyId,
      secretAccessKey,
    },
  });
}
```

Also update the public URL generation in `onAfterSignedUrl`:

```typescript
onAfterSignedUrl: async ({ file, metadata }) => {
  const publicUrl = process.env.STORAGE_PUBLIC_URL;
  const fileUrl = publicUrl
    ? `${publicUrl}/${file.objectKey}`
    : `https://${process.env.STORAGE_BUCKET}.s3.${process.env.AWS_REGION}.amazonaws.com/${file.objectKey}`;

  // ... rest of code
};
```

### Testing Your Configuration

After setting up your storage provider:

1. Start your development server: `pnpm dev`
2. Sign in to your application
3. Navigate to Settings → Profile
4. Click "Change Avatar" and upload an image
5. Verify the image appears immediately
6. Check your storage provider to confirm the file was uploaded to the `avatars/` folder

### Cost Reference

**Cloudflare R2 (as of 2024):**
- Storage: $0.015/GB/month (first 10GB free)
- Class A operations (PUT, POST): $4.50/million (first 1M free/month)
- Class B operations (GET, HEAD): Free
- Egress (bandwidth): Free — major advantage over S3

**AWS S3 (as of 2024):**
- Storage: $0.023/GB/month (first 50GB)
- PUT requests: $0.005/1,000 requests
- GET requests: $0.0004/1,000 requests
- Data transfer OUT: $0.09/GB (first 1GB free/month)

R2 is significantly cheaper for high-bandwidth applications due to free egress.

---

## Using in Code

### Upload Helper Functions

The template provides reusable helper functions in `src/lib/saas/upload/server.ts`.

```typescript
import {
  createR2Client,
  getBucketName,
  getPublicUrl,
} from '@/lib/saas/upload/server';
```

#### `createR2Client()`

Creates and returns a configured R2 client using better-upload's cloudflare helper.

**Returns:** `S3Client` — Configured Cloudflare R2 client
**Throws:** `Error` if required environment variables are missing

```typescript
const r2Client = createR2Client();
```

#### `getBucketName()`

Returns the configured storage bucket name from environment variables.

**Returns:** `string` — Bucket name (defaults to `"jumpsaas"` if not set)

```typescript
const bucketName = getBucketName(); // "jumpsaas" or custom name
```

#### `getPublicUrl(objectKey)`

Generates the public URL for an uploaded file. Automatically uses custom domain if `STORAGE_PUBLIC_URL` is set, otherwise uses R2.dev subdomain.

**Parameters:**
- `objectKey` (string) — The object key/path in the bucket (e.g., `"avatars/user-123-1234567890.jpg"`)

**Returns:** `string` — Public URL for the file

```typescript
const url = getPublicUrl('avatars/user-123.jpg');
// With STORAGE_PUBLIC_URL: "https://cdn.yourdomain.com/avatars/user-123.jpg"
// Without: "https://pub-<account-id>.r2.dev/avatars/user-123.jpg"
```

These helpers automatically handle environment variable validation, custom domain support, R2.dev subdomain fallback, and missing configuration errors.

### Route Structure

The template uses a nested route structure for organizing different upload types:

```
src/app/api/upload/
├── avatar/
│   └── route.ts          # POST /api/upload/avatar
├── media/
│   └── route.ts          # POST /api/upload/media
└── file/
    └── route.ts          # POST /api/upload/file
```

Each endpoint has its own configuration, auth, and file size limits.

### Basic Upload Route

**Next.js** (`src/app/api/upload/{type}/route.ts`):

```typescript
// src/app/api/upload/{type}/route.ts
import {
  createUploadRouteHandler,
  route,
  type Router,
} from 'better-upload/server';
import { auth } from '@/lib/saas/auth/server';
import { headers } from 'next/headers';
import {
  createR2Client,
  getBucketName,
  getPublicUrl,
} from '@/lib/saas/upload/server';

const router: Router = {
  client: createR2Client(),
  bucketName: getBucketName(),
  routes: {
    [routeName]: route({
      fileTypes: ['image/*'],
      maxFileSize: 5 * 1024 * 1024, // 5MB
      onBeforeUpload: async () => {
        // 1. Authenticate user
        const session = await auth.api.getSession({ headers: await headers() });
        if (!session) throw new Error('Unauthorized');

        // 2. Generate unique file name
        const timestamp = Date.now();
        const fileName = `folder/${session.user.id}-${timestamp}`;

        // 3. Return file info and metadata
        return {
          objectInfo: { key: fileName },
          metadata: { userId: session.user.id },
        };
      },
      onAfterSignedUrl: async ({ file, metadata }) => {
        // 4. Generate public URL
        const fileUrl = getPublicUrl(file.objectKey);

        // 5. Save to database
        // await db.insert(table).values({ url: fileUrl, ... });

        // 6. Return metadata with URL
        return {
          metadata: {
            ...metadata,
            fileUrl,
          },
        };
      },
    }),
  },
};

export const { POST } = createUploadRouteHandler(router);
```

**TanStack** (`src/routes/api/upload/$type.ts`):

```typescript
import { createAPIFileRoute } from "@tanstack/react-start/api";
import { getWebRequest } from "@tanstack/react-start/server";
import {
  createUploadRouteHandler,
  route,
  type Router,
} from "better-upload/server";
import { auth } from "@/lib/saas/auth/server";
import {
  createR2Client,
  getBucketName,
  getPublicUrl,
} from "@/lib/saas/upload/server";

const router: Router = {
  client: createR2Client(),
  bucketName: getBucketName(),
  routes: {
    [routeName]: route({
      fileTypes: ["image/*"],
      maxFileSize: 5 * 1024 * 1024,
      onBeforeUpload: async () => {
        const request = getWebRequest();
        const session = await auth.api.getSession({ headers: request.headers });
        if (!session) throw new Error("Unauthorized");

        const timestamp = Date.now();
        const fileName = `folder/${session.user.id}-${timestamp}`;
        return {
          objectInfo: { key: fileName },
          metadata: { userId: session.user.id },
        };
      },
      onAfterSignedUrl: async ({ file, metadata }) => {
        const fileUrl = getPublicUrl(file.objectKey);
        return { metadata: { ...metadata, fileUrl } };
      },
    }),
  },
};

const handler = createUploadRouteHandler(router);

export const APIRoute = createAPIFileRoute("/api/upload/$type")({
  POST: () => handler,
});
```

The `better-upload/server` helpers (`createUploadRouteHandler`, `route`, `createR2Client`, etc.) and all R2/S3 configuration are identical between frameworks — only the route file structure and header access differ.

### Example: Avatar Upload

> **Next.js** example. For TanStack, apply the same handler logic using `createAPIFileRoute` and `getWebRequest()` as shown in the [Basic Upload Route](#basic-upload-route) section above.

**File:** `src/app/api/upload/avatar/route.ts`
**Endpoint:** `POST /api/upload/avatar`

```typescript
import {
  createUploadRouteHandler,
  route,
  type Router,
} from 'better-upload/server';
import { auth } from '@/lib/saas/auth/server';
import { headers } from 'next/headers';
import { db } from '@/db';
import { user } from '@/db/schema';
import { eq } from 'drizzle-orm';
import {
  createR2Client,
  getBucketName,
  getPublicUrl,
} from '@/lib/saas/upload/server';

const router: Router = {
  client: createR2Client(),
  bucketName: getBucketName(),
  routes: {
    avatar: route({
      fileTypes: ['image/*'],
      maxFileSize: 5 * 1024 * 1024, // 5MB
      onBeforeUpload: async () => {
        const session = await auth.api.getSession({ headers: await headers() });
        if (!session) throw new Error('Unauthorized');

        const timestamp = Date.now();
        const fileName = `avatars/${session.user.id}-${timestamp}`;

        return {
          objectInfo: { key: fileName },
          metadata: { userId: session.user.id },
        };
      },
      onAfterSignedUrl: async ({ file, metadata }) => {
        const fileUrl = getPublicUrl(file.objectKey);

        await db
          .update(user)
          .set({ image: fileUrl })
          .where(eq(user.id, metadata.userId));

        return {
          metadata: {
            ...metadata,
            fileUrl,
          },
        };
      },
    }),
  },
};

export const { POST } = createUploadRouteHandler(router);
```

### Example: Media Gallery Upload

> **Next.js** example. For TanStack, apply the same handler logic using `createAPIFileRoute` and `getWebRequest()` as shown in the [Basic Upload Route](#basic-upload-route) section above.

**File:** `src/app/api/upload/media/route.ts`
**Endpoint:** `POST /api/upload/media`

```typescript
import {
  createUploadRouteHandler,
  route,
  type Router,
} from 'better-upload/server';
import { auth } from '@/lib/saas/auth/server';
import { headers } from 'next/headers';
import {
  createR2Client,
  getBucketName,
  getPublicUrl,
} from '@/lib/saas/upload/server';

const router: Router = {
  client: createR2Client(),
  bucketName: getBucketName(),
  routes: {
    media: route({
      fileTypes: ['image/*', 'video/*'],
      maxFileSize: 50 * 1024 * 1024, // 50MB
      multipleFiles: true,
      maxFiles: 10,
      onBeforeUpload: async () => {
        const session = await auth.api.getSession({ headers: await headers() });
        if (!session) throw new Error('Unauthorized');

        const timestamp = Date.now();
        const fileName = `media/${session.user.id}/${timestamp}`;

        return {
          objectInfo: { key: fileName },
          metadata: { userId: session.user.id },
        };
      },
      onAfterSignedUrl: async ({ file, metadata }) => {
        const fileUrl = getPublicUrl(file.objectKey);

        // Save to your media database table
        // await db.insert(media).values({
        //   userId: metadata.userId,
        //   url: fileUrl,
        //   type: file.type,
        //   size: file.size,
        // });

        return { metadata: { ...metadata, fileUrl } };
      },
    }),
  },
};

export const { POST } = createUploadRouteHandler(router);
```

### Client-Side Usage

#### Single File Upload

```typescript
import { useUploadFile } from "better-upload/client";

const { upload, isPending, isSuccess, isError } = useUploadFile({
  route: "avatar",
  api: "/api/upload/avatar",
  onUploadComplete: ({ metadata }) => {
    console.log("Uploaded:", metadata.fileUrl);
  },
});

const handleFileChange = (e: React.ChangeEvent<HTMLInputElement>) => {
  const file = e.target.files?.[0];
  if (file) upload(file);
};
```

#### Multiple File Upload

```typescript
import { useUploadFiles } from "better-upload/client";

const { upload, isPending, uploadedFiles } = useUploadFiles({
  route: "media",
  api: "/api/upload/media",
  onUploadComplete: ({ metadata }) => {
    console.log("All files uploaded:", metadata);
  },
});

const handleFilesChange = (e: React.ChangeEvent<HTMLInputElement>) => {
  const files = Array.from(e.target.files || []);
  upload(files);
};
```

### Configuration Tips

#### Custom File Naming

```typescript
onBeforeUpload: async () => {
  const session = await auth.api.getSession({ headers: await headers() });
  if (!session) throw new Error('Unauthorized');

  const date = new Date().toISOString().split('T')[0]; // YYYY-MM-DD
  const random = Math.random().toString(36).substring(7);
  const fileName = `uploads/${date}/${session.user.id}-${random}`;

  return {
    objectInfo: { key: fileName },
    metadata: { userId: session.user.id },
  };
},
```

#### File Type Validation

```typescript
avatar: route({
  fileTypes: ['image/jpeg', 'image/png', 'image/webp'], // Specific types
  // or
  fileTypes: ['image/*'],                                // Wildcard
  // or
  fileTypes: ['image/*', 'video/mp4', 'application/pdf'], // Multiple types

  maxFileSize: 5 * 1024 * 1024, // 5MB
}),
```

#### Multiple File Configuration

```typescript
media: route({
  multipleFiles: true,
  maxFiles: 10,
  fileTypes: ['image/*', 'video/*'],
  maxFileSize: 50 * 1024 * 1024, // 50MB per file
}),
```

### Error Handling

#### Client-Side

```typescript
const { upload, isError, error } = useUploadFile({
  route: "avatar",
  api: "/api/upload/avatar",
  onUploadFail: (error) => {
    console.error("Upload failed:", error);
  },
});

{isError && <p className="text-red-600">Upload failed. Please try again.</p>}
```

#### Server-Side

```typescript
onBeforeUpload: async () => {
  const session = await auth.api.getSession({ headers: await headers() });

  if (!session) {
    throw new Error('Unauthorized: Please sign in to upload files');
  }

  const userQuota = await checkUserStorageQuota(session.user.id);
  if (userQuota.exceeded) {
    throw new Error('Storage quota exceeded. Please upgrade your plan.');
  }

  // ... rest of logic
},
```

### Security Best Practices

1. **Always authenticate** — check session in `onBeforeUpload` and throw if missing
2. **Validate file types** — use `fileTypes` to restrict accepted MIME types
3. **Set size limits** — use `maxFileSize` to prevent oversized uploads
4. **Use unique file names** — include `userId` and timestamp to avoid collisions
5. **Implement rate limiting** — in middleware or auth logic
6. **Scan for malware** — optional, recommended in production

---

## Troubleshooting

### CORS Errors

**Symptoms:** Browser console shows CORS errors, upload fails.

**Solutions:**
- Verify CORS policy includes your domain with the correct protocol (`https://`)
- Ensure `AllowedMethods` includes `PUT` and `POST`
- Ensure `AllowedHeaders: ["*"]` is set (required for better-upload signed URLs)
- Check `ExposeHeaders` includes `ETag`
- Clear browser cache and retry

### 403 Forbidden Errors

**Symptoms:** Uploads fail with 403, files not accessible.

**Solutions:**
- Verify API token has Object Read & Write permissions
- Check the bucket name matches the `STORAGE_BUCKET` env var
- Verify CORS policy is correctly configured
- For R2: verify public access is enabled on the bucket
- For S3: check IAM user has `PutObject` permission and bucket policy allows public `GetObject`

### Files Upload But Aren't Accessible

**Symptoms:** Upload succeeds, but public URL returns 404.

**Solutions:**
- Verify public access is enabled on the bucket
- Check `STORAGE_PUBLIC_URL` or `R2_PUBLIC_URL_DOMAIN` is correct
- Test the URL in incognito mode (rules out browser cache)
- For custom domains: verify DNS propagation is complete

### Upload Fails Silently

**Symptoms:** Uploads fail without visible errors.

**Solutions:**
- Check server logs for error messages
- Verify `DATABASE_URL` is correct
- Ensure the user is authenticated
- Check that file size does not exceed the configured limit

### Old Files Not Deleted

**Symptoms:** R2 bucket fills with orphaned files.

**Solutions:**
- Check server logs for deletion errors
- Ensure API token has delete permissions
- Set up a lifecycle rule to clean up old files periodically

### "Missing required R2 configuration"

**Solution:** Ensure all required environment variables are set in `.env`. See the [environment variable checklist](#step-6-environment-variable-checklist) above.

---

## Additional Resources

- [better-upload Documentation](https://better-upload.com/docs)
- [Cloudflare R2 Documentation](https://developers.cloudflare.com/r2/)
- [AWS S3 Documentation](https://docs.aws.amazon.com/s3/)
