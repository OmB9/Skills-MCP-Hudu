# Hudu MCP Server — Full Setup Guide

End-to-end instructions for deploying the server and connecting it to Claude.ai with Azure AD OAuth.

---

## Prerequisites

Before you start, you need:

- **A Linux server** (Ubuntu 22.04+ recommended) with:
  - Docker + Docker Compose v2 installed
  - Ports **80** and **443** open in your firewall/security group
- **A domain name** (e.g. `mcp.yourcompany.com`) with an **A record** pointing to your server's public IP
- **A Hudu instance** with an API key
- **An Azure account** with permission to register apps in Azure AD (Entra ID)
- **A Claude.ai account** with access to Integrations (requires Claude Pro/Team/Enterprise)

---

## Part 1 — Azure AD App Registration

### 1.1 Create the app

1. Go to [portal.azure.com](https://portal.azure.com) → **Azure Active Directory** → **App registrations** → **New registration**
2. Fill in:
   - **Name**: `Hudu MCP Server` (or anything descriptive)
   - **Supported account types**: *Accounts in this organizational directory only* (single tenant)
   - **Redirect URI**: leave blank for now — you'll add it later
3. Click **Register**
4. On the app overview page, copy and save:
   - **Application (client) ID** → this is your `AZURE_CLIENT_ID`
   - **Directory (tenant) ID** → this is your `AZURE_TENANT_ID`

### 1.2 Configure as a public client (SPA — enables PKCE)

1. In the app, go to **Authentication** → **Add a platform** → **Single-page application**
2. Leave Redirect URIs blank for now
3. Under **Advanced settings**, set **Allow public client flows** to **Yes**
4. Click **Save**

### 1.3 Set token version to v2.0

1. Go to **Manifest**
2. The `"accessTokenAcceptedVersion"` property does **not** exist by default — you need to add it manually
3. Find the `"signInAudience"` line and add the new property directly after it:
   ```json
   "signInAudience": "AzureADMyOrg",
   "accessTokenAcceptedVersion": 2,
   ```
   The value must be the number `2`, not the string `"2"`. Make sure there are no trailing commas on the line immediately before it.
4. Click **Save**

### 1.4 Expose an API (creates an access token audience)

1. Go to **Expose an API**
2. Click **Add** next to *Application ID URI* — accept the default `api://<client-id>`
3. Click **Add a scope**:
   - Scope name: `access_as_user`
   - Who can consent: *Admins and users*
   - Admin consent display name: `Access Hudu MCP as user`
   - Admin consent description: `Allows the app to access the Hudu MCP server on behalf of the signed-in user.`
   - State: **Enabled**
4. Click **Add scope**
5. Your `AZURE_AUDIENCE` will be: `api://<your-client-id>`

### 1.5 Grant API permissions

1. Go to **API permissions** → **Add a permission** → **Microsoft Graph** → **Delegated permissions**
2. Add: `openid`, `profile`, `email`
3. Click **Grant admin consent** (if you have admin rights) — otherwise ask your Azure admin

---

## Part 2 — Server Setup

### 2.1 Clone the repo

```bash
git clone <repo-url> hudu-mcp
cd hudu-mcp
```

### 2.2 Create your `.env` file

```bash
cp .env.example .env
nano .env   # or use any editor
```

Fill in all values:

```env
# ── Hudu ──────────────────────────────────────────
HUDU_BASE_URL=https://your-company.huducloud.com
HUDU_API_KEY=your-hudu-api-key

# ── Server ────────────────────────────────────────
MCP_SERVER_PORT=3100
NODE_ENV=production
LOG_LEVEL=info

# ── Traefik / HTTPS ───────────────────────────────
MCP_HOSTNAME=mcp.yourcompany.com        # must match your DNS A record
ACME_EMAIL=you@yourcompany.com          # for Let's Encrypt notifications

# ── OAuth 2.1 (Azure AD) ─────────────────────────
MCP_OAUTH_ENABLED=true
MCP_SERVER_URL=https://mcp.yourcompany.com   # same as MCP_HOSTNAME with https://
AZURE_TENANT_ID=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
AZURE_CLIENT_ID=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
AZURE_AUDIENCE=api://xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
```

### 2.3 Create the logs directory

```bash
mkdir -p logs
```

### 2.4 Build and start

```bash
docker compose up -d --build
```

This will:
- Build the MCP server image
- Start Traefik (which automatically fetches a Let's Encrypt certificate for your domain)
- Start the MCP server

> **Note:** The first certificate issuance takes 30–60 seconds. Traefik must be able to receive a request on port 80 from Let's Encrypt's servers.

### 2.5 Verify everything is running

```bash
# Check containers are up
docker compose ps

# Check logs for errors
docker compose logs -f hudu-mcp-server

# Test health endpoint
curl https://mcp.yourcompany.com/health
# Expected: {"status":"healthy","timestamp":"...","version":"1.1.0","transport":"streamable-http"}

# Test OAuth discovery endpoint
curl https://mcp.yourcompany.com/.well-known/oauth-protected-resource
# Expected: {"resource":"https://mcp.yourcompany.com","authorization_servers":[...],...}

# Test that /mcp requires auth
curl -X POST https://mcp.yourcompany.com/mcp -H "Content-Type: application/json" -d '{}'
# Expected: 401 with WWW-Authenticate header
```

---

## Part 3 — Connect to Claude.ai

### 3.1 Add the custom connector

1. Go to [claude.ai](https://claude.ai) → **Settings** (bottom-left) → **Integrations**
2. Click **Add custom integration** (or similar — the label may vary by account type)
3. Fill in:
   - **Name**: `Hudu`
   - **URL**: `https://mcp.yourcompany.com/mcp`
   - **Authentication**: OAuth 2.0
4. Click **Connect** / **Add**

### 3.2 Capture the redirect URI (first attempt)

Claude.ai will call your `/register` endpoint during the connection attempt. **Check your server logs immediately:**

```bash
docker compose logs hudu-mcp-server | grep "OAuth Dynamic Client Registration"
```

You'll see something like:

```json
{
  "message": "OAuth Dynamic Client Registration",
  "redirect_uris": ["https://claude.ai/api/mcp/auth_callback"],
  ...
}
```

Copy the `redirect_uris` value.

### 3.3 Add the redirect URI to Azure AD

1. Back in Azure Portal → your **Hudu MCP Server** app registration → **Authentication**
2. Under **Single-page application**, click **Add URI**
3. Paste the redirect URI from the logs (e.g. `https://claude.ai/api/mcp/auth_callback`)
4. Click **Save**

### 3.4 Connect again

1. Go back to Claude.ai → **Settings** → **Integrations**
2. Remove the previous (failed) connection if present
3. Add the connector again with the same URL
4. This time, a browser tab will open → Microsoft login → sign in with your Azure AD account
5. Approve the permissions prompt if shown
6. The tab closes and Claude.ai shows the connection as active

### 3.5 Test it

In a Claude.ai conversation, try:

```
Search for all companies in Hudu
```

or

```
Use hudu_search_company_information to find companies matching "your-company-name"
```

You should see Claude calling the tool and returning results from your Hudu instance.

---

## Troubleshooting

### Certificate not issued / HTTPS not working

- Confirm your DNS A record resolves to your server: `nslookup mcp.yourcompany.com`
- Confirm port 80 is open: `curl http://mcp.yourcompany.com` from an external machine
- Check Traefik logs: `docker compose logs traefik`

### 401 on /mcp after auth

- Confirm `MCP_OAUTH_ENABLED=true` in your `.env`
- Confirm `AZURE_AUDIENCE` exactly matches `api://<your-client-id>`
- Confirm token version is v2 in Azure AD manifest (`"accessTokenAcceptedVersion": 2`)
- Check: `docker compose logs hudu-mcp-server | grep "validation failed"`

### OAuth tab opens but shows an Azure error

- **AADSTS50011** (redirect URI mismatch): The redirect URI in Claude.ai's request doesn't match what's registered in Azure AD. Re-check step 3.3.
- **AADSTS65001** (consent not granted): Go to Azure AD → your app → API permissions → Grant admin consent.
- **AADSTS700016** (app not found): Double-check `AZURE_CLIENT_ID` and `AZURE_TENANT_ID` in your `.env`.

### Tools not showing in Claude.ai

```bash
# Rebuild and restart
docker compose down
docker compose up -d --build
docker compose logs -f hudu-mcp-server
```

### Check running status

```bash
docker compose ps                          # container status
docker compose logs traefik                # Traefik / cert issues
docker compose logs hudu-mcp-server        # MCP server issues
curl https://mcp.yourcompany.com/health    # liveness
```

---

## Updating

```bash
git pull
docker compose down
docker compose up -d --build
```

Traefik will auto-renew certificates — no action needed.
