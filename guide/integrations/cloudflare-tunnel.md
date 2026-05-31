---
title: "Cloudflare Tunnel for Local Development"
---

# Cloudflare Tunnel for Local Development

> **Next.js only.** This guide is most relevant to the `jumpsaas-nextjs` template (particularly the `cloudflare` branch). TanStack users can also use Cloudflare Tunnel for local webhook testing, but the Cloudflare Workers deployment target is not available in the TanStack variant.

Set up a permanent development URL using Cloudflare Tunnel. This creates a stable HTTPS URL that doesn't change between sessions, making it perfect for webhook integrations (Stripe, Slack, etc.) that require consistent callback URLs.

## Overview

**Benefits:**
- [YOURS] Permanent HTTPS URL (e.g., `https://dev.yourdomain.com`)
- [YOURS] Configure webhooks once - never change URLs again
- [YOURS] No ngrok sessions that expire
- [YOURS] Works with any port/service

**Use Cases:**
- Stripe webhook testing
- Slack app development
- OAuth callback URLs
- Any external service requiring webhooks

## Prerequisites

- Cloudflare account with your domain configured
- `cloudflared` CLI installed
  ```bash
  brew install cloudflare/cloudflare/cloudflared
  ```

## Setup Process

### Step 1: Login to Cloudflare

```bash
cloudflared tunnel login
```

- Opens browser automatically
- Select your domain from the list
- Authenticates your CLI

### Step 2: Create Named Tunnel

```bash
cloudflared tunnel create <tunnel-name>
```

**Example:**
```bash
cloudflared tunnel create myapp-dev
```

**Output:**
```
Tunnel credentials written to /Users/username/.cloudflared/<UUID>.json
Created tunnel myapp-dev with id <UUID>
```

**[CAUTION] IMPORTANT:** Copy the UUID from the output - you'll need it in the next step.

### Step 3: Create Config File

```bash
cat > ~/.cloudflared/config.yml << 'EOF'
tunnel: <tunnel-name>
credentials-file: /Users/username/.cloudflared/YOUR-UUID-HERE.json

ingress:
  - hostname: <subdomain>.yourdomain.com
    service: http://localhost:3000
  - service: http_status:404
EOF
```

**Edit the config file** to replace placeholders:

```bash
nano ~/.cloudflared/config.yml
```

**Example final config:**
```yaml
tunnel: myapp-dev
credentials-file: /Users/username/.cloudflared/a1b2c3d4-e5f6-7890-abcd-ef1234567890.json

ingress:
  - hostname: dev.jumpsaas.com
    service: http://localhost:3000
  - service: http_status:404
```

**Configuration Options:**

| Field | Description | Example |
|-------|-------------|---------|
| `tunnel` | Your tunnel name | `myapp-dev` |
| `credentials-file` | Path to UUID.json file | `~/.cloudflared/<UUID>.json` |
| `hostname` | Your public URL | `dev.yourdomain.com` |
| `service` | Local server to tunnel to | `http://localhost:3000` |

**Port Options:**
- Next.js dev server: `http://localhost:3000`
- Custom port: `http://localhost:8080`
- Different service: `http://localhost:5000`

### Step 4: Create DNS Record

```bash
cloudflared tunnel route dns <tunnel-name> <subdomain>.yourdomain.com
```

**Example:**
```bash
cloudflared tunnel route dns myapp-dev dev.jumpsaas.com
```

**Output:**
```
Created CNAME record for dev.jumpsaas.com
```

This automatically creates a DNS record in your Cloudflare dashboard:
- **Type**: CNAME
- **Name**: `dev` (or your chosen subdomain)
- **Target**: `<UUID>.cfargotunnel.com`

### Step 5: Start the Tunnel

```bash
cloudflared tunnel run <tunnel-name>
```

**Example:**
```bash
cloudflared tunnel run myapp-dev
```

**Keep this terminal running!** You should see:
```
Connection registered
Registered tunnel connection
```

Your permanent URL is now active: **`https://<subdomain>.yourdomain.com`** [YOURS]

## Daily Development Workflow

You need two things running:
1. **Cloudflare tunnel** - Routes traffic to your local server
2. **Local dev server** - Your application (Next.js, etc.)

### Option A: Manual Start (Two Terminals)

**Terminal 1** - Start tunnel:
```bash
cloudflared tunnel run <tunnel-name>
```

**Terminal 2** - Start dev server:
```bash
pnpm dev
```

### Option B: Automated Script (Recommended)

Create `dev-tunnel.sh` in project root:

```bash
#!/bin/bash

TUNNEL_NAME="myapp-dev"  # Change this to your tunnel name
DEV_URL="dev.yourdomain.com"  # Change this to your URL

echo "🚇 Starting Cloudflare Tunnel: $DEV_URL"
cloudflared tunnel run $TUNNEL_NAME &
TUNNEL_PID=$!

echo "⏳ Waiting for tunnel to connect..."
sleep 3

echo "🚀 Starting Next.js dev server..."
pnpm dev

# Clean up tunnel on exit
trap "echo '🛑 Stopping tunnel...'; kill $TUNNEL_PID" EXIT
```

Make executable:
```bash
chmod +x dev-tunnel.sh
```

Run:
```bash
./dev-tunnel.sh
```

Press `Ctrl+C` to stop both services.

## Environment Variables

Update your `.env` to use the tunnel URL:

```bash
NEXT_PUBLIC_APP_URL=https://dev.yourdomain.com
```

**[CAUTION] Important:** Restart dev server after changing `NEXT_PUBLIC_APP_URL` (it's baked into the client bundle at build time).

## Troubleshooting

### Error 1033 - Tunnel Connection Failed

**Symptoms:** Browser shows "Cloudflare Tunnel error 1033"

**Solutions:**

1. **Check dev server is running:**
   ```bash
   curl http://localhost:3000
   ```
   Should return HTML, not "Connection refused"

2. **Check tunnel is running:**
   ```bash
   ps aux | grep cloudflared
   ```
   Should show `cloudflared tunnel run <tunnel-name>`

3. **Restart both services:**
   ```bash
   # Stop tunnel (Ctrl+C in tunnel terminal)
   # Stop dev server (Ctrl+C in dev terminal)

   # Start tunnel first
   cloudflared tunnel run <tunnel-name>

   # Wait for "Connection registered", then start dev server
   pnpm dev
   ```

4. **Verify tunnel config:**
   ```bash
   cat ~/.cloudflared/config.yml
   ```
   Should have correct UUID and hostname

### DNS Not Resolving

**Check DNS propagation:**
```bash
nslookup dev.yourdomain.com
```

Should return Cloudflare IP (e.g., `198.41.192.167`)

**If not working:**
- Wait 1-2 minutes for DNS propagation
- Check Cloudflare Dashboard → Your Domain → DNS
- Look for CNAME: `dev` (or your subdomain) → `<UUID>.cfargotunnel.com`

### Wrong Port

If tunnel connects but shows wrong service:

1. **Check config file port:**
   ```bash
   cat ~/.cloudflared/config.yml
   ```
   Should match your dev server port

2. **Update if needed:**
   ```bash
   nano ~/.cloudflared/config.yml
   # Change: service: http://localhost:3000
   # To:     service: http://localhost:8080  (or your port)
   ```

3. **Restart tunnel:**
   ```bash
   cloudflared tunnel run <tunnel-name>
   ```

## Useful Commands

```bash
# List all tunnels
cloudflared tunnel list

# Show tunnel info
cloudflared tunnel info <tunnel-name>

# Delete tunnel (if needed)
cloudflared tunnel delete <tunnel-name>

# Check tunnel status with debug logs
cloudflared tunnel run <tunnel-name> --loglevel debug

# Stop tunnel
# Just press Ctrl+C in the tunnel terminal
```

## Multiple Tunnels

You can create different tunnels for different projects:

```bash
# Create tunnels
cloudflared tunnel create project1-dev
cloudflared tunnel create project2-dev

# Configure different domains/subdomains
# project1-dev.yourdomain.com → localhost:3000
# project2-dev.yourdomain.com → localhost:4000

# Run specific tunnel
cloudflared tunnel run project1-dev
```

## Production Notes

**You don't need tunnels in production.** Cloudflare Tunnel is for local development only.

In production:
- Use direct domain URLs (e.g., `https://www.yourdomain.com`)
- Configure webhooks to production URLs
- Remove tunnel references from deployment configs

## Additional Resources

- [Cloudflare Tunnel Documentation](https://developers.cloudflare.com/cloudflare-one/connections/connect-apps/)
- [Tunnel Configuration Reference](https://developers.cloudflare.com/cloudflare-one/connections/connect-apps/install-and-setup/tunnel-guide/local/)
- [DNS Configuration](https://developers.cloudflare.com/dns/)

---

**Last Updated:** 2026-02-15
