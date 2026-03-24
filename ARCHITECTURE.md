# Hudu MCP Server — Architecture & Implementation Reference

This document explains everything about how this server works, why each decision was made, and how to recreate it from scratch. Written after a full implementation session so nothing is assumed.

---

## What This Is

A **Model Context Protocol (MCP) server** that wraps the Hudu IT documentation API and exposes it as a set of tools to Claude. Deployed as a Docker container behind Caddy (reverse proxy with automatic HTTPS), authenticated via Azure AD OAuth 2.1.

**End result:** Claude.ai (web) and Claude Code (CLI) can call Hudu API tools — search companies, assets, articles, passwords, networks, etc. — authenticated through your Azure AD tenant.

---

## Architecture Overview

```
Claude.ai / Claude Code
        │
        │ HTTPS (443)
        ▼
   [Caddy container]          ← automatic Let's Encrypt TLS
        │
        │ HTTP (3100, internal)
        ▼
[hudu-mcp-server container]   ← Node.js / TypeScript MCP server
        │
        ├── OAuth proxy endpoints (/authorize, /token, /register)
        │         │
        │         ▼
        │   Azure AD (Entra ID)
        │
        └── Tool execution
                  │
                  ▼
         Hudu API (external)
```

**Why Caddy instead of Traefik:** Traefik v3.x uses Docker API version 1.24 by default when connecting to the Docker socket. This server's Docker daemon requires API ≥ 1.40 and returns an error. The `DOCKER_API_VERSION` environment variable does not fix it in Traefik v3. Caddy requires no Docker socket — routes are defined statically in a `Caddyfile`. It also handles Let's Encrypt automatically with zero config.

---

## Authentication Flow

### The Core Problem with Claude.ai OAuth

Claude.ai's custom connectors implement an **older version of the MCP OAuth 2.1 spec** (approximately 2025-03-26) with several confirmed deviations from the current spec:

| Behavior | What spec says | What Claude.ai web actually does |
|----------|---------------|----------------------------------|
| `authorization_endpoint` | Use the value from AS metadata | **Ignores it.** Always constructs `https://your-server/authorize` |
| `token_endpoint` | Use the value from AS metadata | **Ignores it.** Always constructs `https://your-server/token` |
| `registration_endpoint` | Use the full path from AS metadata | **Strips to root.** Always POSTs to `https://your-server/register` |
| AS discovery from `authorization_servers` | Follow the issuer URL to the external AS | **Ignores external AS.** Still builds endpoints on your server domain |

**Practical consequence:** Even if you point `authorization_servers` directly at Azure AD in your protected resource metadata, Claude.ai web will never talk to Azure AD directly for the OAuth flow. It always calls `/authorize`, `/token`, and `/register` on your MCP server's domain. Your server must proxy these to Azure AD.

### The RFC 8414 Issuer Rule

RFC 8414 (Authorization Server Metadata) requires that the `issuer` field in `/.well-known/oauth-authorization-server` **must exactly match the URL prefix used to fetch that document**. If you serve the document from `https://mcp.yourserver.com/.well-known/oauth-authorization-server`, the `issuer` must be `https://mcp.yourserver.com`.

Our original implementation set `issuer` to Azure AD's issuer (`https://login.microsoftonline.com/{tenant}/v2.0`) while serving from our server URL. Spec-compliant clients must discard a response where the issuer doesn't match. This caused Claude.ai to silently fail during discovery.

### How the OAuth Flow Actually Works

```
1. Claude.ai → POST https://mcp.v1corp.com/mcp (no token)
   ← 401  WWW-Authenticate: Bearer resource_metadata="https://mcp.v1corp.com/.well-known/oauth-protected-resource"

2. Claude.ai → GET https://mcp.v1corp.com/.well-known/oauth-protected-resource
   ← { resource, authorization_servers: ["https://mcp.v1corp.com"], ... }

3. Claude.ai → GET https://mcp.v1corp.com/.well-known/oauth-authorization-server
   ← { issuer: "https://mcp.v1corp.com",
       authorization_endpoint: "https://mcp.v1corp.com/authorize",
       token_endpoint: "https://mcp.v1corp.com/token",
       registration_endpoint: "https://mcp.v1corp.com/register", ... }

4. Claude.ai → POST https://mcp.v1corp.com/register
      { redirect_uris: ["https://claude.ai/api/mcp/auth_callback"], ... }
   ← { client_id: "<azure-app-client-id>", ... }

5. Claude.ai → browser opens:
   GET https://mcp.v1corp.com/authorize?response_type=code&client_id=...&redirect_uri=https://claude.ai/api/mcp/auth_callback&code_challenge=...

6. Our /authorize → 302 redirect to Azure AD:
   GET https://login.microsoftonline.com/{tenant}/oauth2/v2.0/authorize?
       client_id={azure-client-id}&scope=api://{client-id}/access_as_user openid ...&...

7. User logs in at Azure AD
   Azure AD → 302 redirect to https://claude.ai/api/mcp/auth_callback?code=...

8. Claude.ai → POST https://mcp.v1corp.com/token
      { grant_type=authorization_code, code=..., code_verifier=..., redirect_uri=... }

9. Our /token → proxies to Azure AD:
   POST https://login.microsoftonline.com/{tenant}/oauth2/v2.0/token
   ← { access_token, refresh_token, expires_in, ... }
   → returns token response to Claude.ai

10. Claude.ai → POST https://mcp.v1corp.com/mcp
        Authorization: Bearer <azure-access-token>
    ← 200 (tools respond)
```

### Why the Redirect URI Must Be Under "Mobile and Desktop Applications"

In Azure AD, platform types enforce different token issuance rules:
- **Web**: Tokens are sender-constrained. Requires a client secret for token exchange.
- **SPA (Single-page application)**: Issues tokens with `Sec-Fetch-Site` and origin binding. Designed for browser JS apps. Token binding breaks programmatic OAuth clients.
- **Mobile and desktop applications**: Public client. Supports PKCE without a client secret. No token binding. This is what Claude.ai's flow requires.

Azure AD only allows a redirect URI to be registered under one platform type. If `https://claude.ai/api/mcp/auth_callback` is registered under SPA, the token exchange will fail. It must be under **Mobile and desktop applications**.

---

## Server Components

### OAuth Proxy Endpoints (`src/server.ts`)

All endpoints are only active when `MCP_OAUTH_ENABLED=true`.

#### `GET /.well-known/oauth-protected-resource`
RFC 9728 document. Points clients to our server as the Authorization Server.
- `resource` includes the `/mcp` path: `https://mcp.v1corp.com/mcp`
- `authorization_servers` points to ourselves: `["https://mcp.v1corp.com"]`

#### `GET /.well-known/oauth-authorization-server`
RFC 8414 document. **Issuer is our server URL** (not Azure AD). Endpoints point to our proxy routes.

#### `POST /register`
Dynamic Client Registration (RFC 7591). Returns our Azure AD app's `client_id`. Azure AD doesn't support open DCR, so this endpoint returns the pre-registered app credentials. Claude.ai uses the returned `client_id` for all subsequent OAuth requests.

#### `GET /authorize`
Receives Claude.ai's authorization request, rewrites it for Azure AD (substitutes `client_id`, sets correct `scope`), and redirects (302) to Azure AD's authorization endpoint. The `redirect_uri` (`https://claude.ai/api/mcp/auth_callback`) passes through unchanged — Azure AD redirects the user back to Claude.ai directly.

Key transformation:
- `client_id`: replaced with `AZURE_CLIENT_ID` (Claude.ai sends whatever we returned from `/register`)
- `scope`: replaced with `api://{AZURE_CLIENT_ID}/access_as_user openid profile email offline_access`
- `resource`: dropped (Azure AD v2 uses `scope`, not `resource`)
- `code_challenge` / `code_challenge_method`: passed through unchanged (PKCE)

#### `POST /token`
Receives Claude.ai's token exchange request, proxies to Azure AD's `/token` endpoint with the same rewrites as `/authorize`. Returns the Azure AD token response directly to Claude.ai.

#### Bearer token validation middleware (on `/mcp` only)
Validates Azure AD JWTs using the tenant's JWKS endpoint. Tries multiple audiences (`api://client_id`, `client_id`, configured `AZURE_AUDIENCE`) to handle different token types. On failure, logs the token's actual `aud` claim for diagnostics.

#### Static Bearer token (alternative auth)
When `MCP_OAUTH_ENABLED=false` and `MCP_BEARER_TOKEN` is set, a simpler static token check is used instead of OAuth. Used for Claude Code CLI and Claude Desktop where OAuth is not required.

---

## Azure AD App Registration

### Required Configuration

| Setting | Value |
|---------|-------|
| App type | Single tenant |
| Platform | **Mobile and desktop applications** |
| Redirect URI | `https://claude.ai/api/mcp/auth_callback` |
| Client secret | **None** (public client, PKCE only) |
| Allow public client flows | **Yes** |
| `accessTokenAcceptedVersion` (Manifest) | `2` (must be added manually — not present by default) |

### API Exposure (required for `api://` audience)

1. **Expose an API** → Application ID URI: `api://{client-id}` (accept default)
2. Add scope: `access_as_user` (Admin + users consent, Enabled)

This creates the audience `api://{client-id}` that our server validates tokens against.

### Manifest Change

In the app Manifest, `accessTokenAcceptedVersion` is not present by default. Add it manually at the root level:

```json
"signInAudience": "AzureADMyOrg",
"accessTokenAcceptedVersion": 2,
```

Value must be the number `2`, not the string `"2"`. This forces v2 tokens (required for our issuer validation).

### API Permissions

`openid`, `profile`, `email` (Microsoft Graph, delegated) — these are usually pre-granted. Grant admin consent if prompted.

---

## Infrastructure

### Docker Compose

Two services:

**caddy** (`caddy:2-alpine`):
- Ports 80 and 443
- Mounts `Caddyfile` read-only
- Named volumes for cert persistence (`caddy-data`, `caddy-config`)
- No Docker socket needed

**hudu-mcp-server** (custom build):
- Internal only (no published ports — Caddy proxies to it over the internal bridge network)
- Bind mount `./logs` → requires `sudo chown -R 1001:1001 logs` on host (container runs as uid 1001)
- All config via environment variables from `.env`

### Caddyfile

```
{$MCP_HOSTNAME} {
    reverse_proxy hudu-mcp-server:3100
    header { ... security headers ... }
}
```

Caddy automatically obtains and renews Let's Encrypt certificates via TLS-ALPN-01 challenge (uses port 443, not port 80). Both ports 80 and 443 must be open in Azure NSG.

### Dockerfile

Multi-stage-style: installs deps, builds TypeScript, prunes dev deps, runs as non-root user `hudu` (uid 1001). Node 20 Alpine base.

---

## Environment Variables

```env
# Hudu API
HUDU_BASE_URL=https://your-company.huducloud.com
HUDU_API_KEY=your-api-key

# Infrastructure
MCP_SERVER_PORT=3100
MCP_HOSTNAME=mcp.yourcompany.com
ACME_EMAIL=you@yourcompany.com

# OAuth 2.1 (for Claude.ai web connectors)
MCP_OAUTH_ENABLED=true
MCP_SERVER_URL=https://mcp.yourcompany.com
AZURE_TENANT_ID=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
AZURE_CLIENT_ID=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
AZURE_AUDIENCE=api://xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx

# Static Bearer token (for Claude Code CLI / Claude Desktop)
# Use this OR MCP_OAUTH_ENABLED=true, not both
# Generate: openssl rand -hex 32
MCP_BEARER_TOKEN=your-secure-token
```

---

## Connecting Clients

### Claude.ai Web (Custom Connector)

Settings → Integrations → Add custom integration:
- URL: `https://mcp.yourcompany.com/mcp`
- Auth: OAuth 2.0

The first connection attempt completes the OAuth flow. The redirect URI `https://claude.ai/api/mcp/auth_callback` must be pre-registered in Azure AD under Mobile and desktop applications.

### Claude Code CLI

```bash
claude mcp add --transport http hudu https://mcp.yourcompany.com/mcp \
  --header "Authorization: Bearer <MCP_BEARER_TOKEN>"
```

Set `MCP_OAUTH_ENABLED=false` and `MCP_BEARER_TOKEN=<token>` in `.env` for this mode.

### Claude Desktop

Claude Desktop's `claude_desktop_config.json` **only supports local STDIO MCP servers**. HTTP/remote servers are not configurable via the JSON file. Use the Integrations UI in Claude.ai web, or use Claude Code CLI.

---

## Deployment Steps (From Scratch)

### 1. Azure AD

1. Portal → Azure Active Directory → App registrations → New registration
2. Name it, single tenant, register
3. **Authentication** → Add platform → Mobile and desktop applications → add `https://claude.ai/api/mcp/auth_callback` → Save
4. **Authentication** → Advanced settings → Allow public client flows → **Yes** → Save
5. **Manifest** → add `"accessTokenAcceptedVersion": 2` after `"signInAudience"` → Save
6. **Expose an API** → Set Application ID URI (`api://{client-id}`) → Add scope `access_as_user` → Save
7. **API permissions** → Add `openid`, `profile`, `email` from Microsoft Graph → Grant admin consent
8. Copy **Application (client) ID** and **Directory (tenant) ID**

### 2. Server

```bash
# Prerequisites: Docker + Compose v2, ports 80+443 open, DNS A record pointing to server

git clone <repo> hudu-mcp
cd hudu-mcp

# Set permissions for log volume
mkdir -p logs
sudo chown -R 1001:1001 logs

# Configure
cp .env.example .env
nano .env  # fill in all values

# Deploy
docker compose up -d --build

# Verify
curl https://mcp.yourcompany.com/health
curl https://mcp.yourcompany.com/.well-known/oauth-protected-resource
curl https://mcp.yourcompany.com/.well-known/oauth-authorization-server
```

### 3. Claude.ai

1. Settings → Integrations → Add custom integration
2. Enter URL and select OAuth 2.0
3. Click Connect → Microsoft login appears → authenticate
4. Connection shows as active

---

## Troubleshooting

### "Couldn't reach the MCP server" in Claude.ai
- Check that `/authorize` and `/token` endpoints exist on the server
- Check server logs: `docker compose logs hudu-mcp-server`
- Verify `issuer` in `/.well-known/oauth-authorization-server` matches your server URL (not Azure AD)
- Verify redirect URI is under **Mobile and desktop applications** in Azure AD (not SPA)

### OAuth tab opens but Azure AD shows AADSTS50011 (redirect URI mismatch)
The redirect URI in the request doesn't match what's registered. `https://claude.ai/api/mcp/auth_callback` must be registered in Azure AD under Mobile and desktop applications.

### OAuth completes but token validation fails
Check logs for `Bearer token validation failed` — the log entry shows `tokenAudience` (what Azure AD put in the token) and `triedAudiences` (what we validated against). Set `AZURE_AUDIENCE` in `.env` to match the `tokenAudience` value exactly.

### Server crashes on startup with EACCES on logs
```bash
sudo chown -R 1001:1001 logs
docker compose restart hudu-mcp-server
```

### Caddy shows unhealthy
Check `docker compose logs caddy`. Most common causes:
- Port 80 or 443 not open in Azure NSG
- DNS A record not pointing to the server
- Certificate rate limit hit (Let's Encrypt limits: 5 certs per domain per week)

### Traefik Docker API error (if switching back to Traefik)
Don't. The host Docker daemon requires API ≥ 1.40; Traefik v3's Go Docker SDK defaults to 1.24 and `DOCKER_API_VERSION` env var does not fix it. Use Caddy.

---

## Key Files

| File | Purpose |
|------|---------|
| `src/server.ts` | MCP HTTP server, all OAuth proxy endpoints, JWT validation |
| `src/hudu-client.ts` | Hudu API client (typed methods) |
| `src/tools/working-index.ts` | All 47 registered tools |
| `docker-compose.yml` | Caddy + MCP server containers |
| `Caddyfile` | Reverse proxy config + security headers |
| `Dockerfile` | Node 20 Alpine, non-root build |
| `.env.example` | All configurable variables with descriptions |
| `SETUP.md` | Step-by-step first-time setup guide |

---

## What Would Break This in the Future

- **Claude.ai fixes their OAuth implementation** to respect `authorization_endpoint`/`token_endpoint` from AS metadata — our proxy endpoints would still work fine (they'd just not be called), so nothing breaks.
- **Azure AD changes token format** — update `AZURE_AUDIENCE` in `.env` to match new audience value.
- **Let's Encrypt rate limits** — if you recreate the server frequently, you may hit 5 certs/week/domain. Use `caddy:2-alpine` with staging ACME for testing.
- **MCP spec version bump** — check `@modelcontextprotocol/sdk` for breaking changes. The `CLAUDE.md` requires always using the latest SDK version.
- **`accessTokenAcceptedVersion` in manifest** — if Azure resets this during app updates, tokens will revert to v1 format and issuer validation will fail.
