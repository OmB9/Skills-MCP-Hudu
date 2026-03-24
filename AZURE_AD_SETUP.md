# Azure AD App Registration — Hudu MCP Server

Step-by-step guide for creating the Azure AD app registration that backs the MCP server's OAuth 2.1 integration with Claude.ai.

---

## Overview

The MCP server acts as an OAuth 2.1 proxy between Claude.ai and Azure AD. Claude.ai talks exclusively to the MCP server's `/authorize`, `/token`, and `/register` endpoints — the server then proxies those requests to Azure AD on Claude.ai's behalf.

The app registration needs to be configured as a **public client** (no client secret, PKCE only) because Claude.ai's OAuth flow does not supply a client secret during token exchange.

---

## Prerequisites

- Azure account with **Application Administrator** (or higher) role in your tenant
- Your MCP server's public hostname (e.g. `mcp.yourcompany.com`)

---

## Step 1: Create the App Registration

1. Go to [portal.azure.com](https://portal.azure.com) → **Azure Active Directory** → **App registrations** → **New registration**
2. Fill in:
   - **Name**: `Hudu MCP Server` (or any descriptive name)
   - **Supported account types**: *Accounts in this organizational directory only* (single tenant)
   - **Redirect URI**: leave blank — you'll add it after the first Claude.ai connection attempt
3. Click **Register**
4. On the overview page, copy and save:
   - **Application (client) ID** → `AZURE_CLIENT_ID` in your `.env`
   - **Directory (tenant) ID** → `AZURE_TENANT_ID` in your `.env`

---

## Step 2: Configure Platform as Mobile and Desktop Applications

> **Critical:** The redirect URI must be registered under **Mobile and desktop applications**, NOT SPA or Web.
>
> - **SPA**: Issues sender-constrained tokens bound to the browser origin — breaks Claude.ai's programmatic token exchange.
> - **Web**: Requires a client secret for token exchange — incompatible with PKCE-only public clients.
> - **Mobile and desktop applications**: Public client, supports PKCE without a secret, no token binding. This is what Claude.ai requires.

1. Go to **Authentication** → **Add a platform** → **Mobile and desktop applications**
2. Leave Redirect URIs blank for now (you'll add it in Step 6)
3. Under **Advanced settings**, set **Allow public client flows** to **Yes**
4. Click **Save**

---

## Step 3: Set Token Version to v2.0

Azure AD app registrations default to v1 tokens. The MCP server validates v2 tokens (different issuer URL format). This must be set manually in the manifest.

1. Go to **Manifest**
2. `"accessTokenAcceptedVersion"` does **not** exist by default — add it manually
3. Find the `"signInAudience"` line and add the new property directly after it:
   ```json
   "signInAudience": "AzureADMyOrg",
   "accessTokenAcceptedVersion": 2,
   ```
   The value must be the number `2`, not the string `"2"`.
4. Click **Save**

---

## Step 4: Expose an API (creates the `api://` audience)

This creates the `api://<client-id>` audience that the MCP server validates tokens against.

1. Go to **Expose an API**
2. Click **Add** next to *Application ID URI* — accept the default `api://<client-id>`
3. Click **Add a scope**:
   - **Scope name**: `access_as_user`
   - **Who can consent**: Admins and users
   - **Admin consent display name**: `Access Hudu MCP as user`
   - **Admin consent description**: `Allows the app to access the Hudu MCP server on behalf of the signed-in user.`
   - **State**: Enabled
4. Click **Add scope**

Your `AZURE_AUDIENCE` will be: `api://<your-client-id>`

---

## Step 5: Grant API Permissions

1. Go to **API permissions** → **Add a permission** → **Microsoft Graph** → **Delegated permissions**
2. Add: `openid`, `profile`, `email`
3. Click **Grant admin consent** (if you have admin rights)

---

## Step 6: Add the Redirect URI (after first Claude.ai connection attempt)

Claude.ai's redirect URI is not known until the first connection attempt. The MCP server logs it when Claude.ai calls `/register`.

**After your first Claude.ai connection attempt:**

```bash
docker compose logs hudu-mcp-server | grep "OAuth Dynamic Client Registration"
```

You'll see the `redirect_uris` value — typically `https://claude.ai/api/mcp/auth_callback`.

To add it:
1. Go to **Authentication**
2. Under **Mobile and desktop applications**, click **Add URI**
3. Paste the URI from the logs
4. Click **Save**

Then retry the connection in Claude.ai.

---

## Step 7: Gather Configuration Values

```env
AZURE_TENANT_ID=<Directory (tenant) ID from Step 1>
AZURE_CLIENT_ID=<Application (client) ID from Step 1>
AZURE_AUDIENCE=api://<Application (client) ID from Step 1>
```

No client secret is needed — this is a public client.

---

## Configuration Summary

| Setting | Value |
|---------|-------|
| App type | Single tenant |
| Platform | **Mobile and desktop applications** |
| Redirect URI | `https://claude.ai/api/mcp/auth_callback` (added after first attempt) |
| Client secret | **None** (public client, PKCE only) |
| Allow public client flows | **Yes** |
| `accessTokenAcceptedVersion` (Manifest) | `2` |
| API scope | `api://<client-id>/access_as_user` |
| Graph permissions | `openid`, `profile`, `email` |

---

## Troubleshooting

### AADSTS50011 — redirect URI mismatch
The URI in Claude.ai's request doesn't match what's registered. Re-check the value from the server logs and confirm it's saved under **Mobile and desktop applications** (not SPA or Web).

### AADSTS65001 — consent not granted
Go to **API permissions** → **Grant admin consent**.

### AADSTS700016 — application not found
Double-check `AZURE_CLIENT_ID` and `AZURE_TENANT_ID` in your `.env`.

### Token validation fails after successful OAuth
Check server logs for `Bearer token validation failed` — the log entry shows `tokenAudience` (what Azure AD put in the token). Set `AZURE_AUDIENCE` in `.env` to match that value exactly.

### `accessTokenAcceptedVersion` resets after manifest edit
Azure occasionally resets this during other manifest changes. If token issuer validation starts failing, re-check the manifest and add the property again.
