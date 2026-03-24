# User Onboarding Guide
## Connecting to the Hudu MCP Server

**Time required:** 5–10 minutes

---

## Choose Your Client

There are two supported ways to connect:

| Client | Auth Method | Setup |
|--------|-------------|-------|
| **Claude.ai web** (Pro/Team/Enterprise) | OAuth 2.0 (Azure AD) | See Part A |
| **Claude Code CLI** | Static Bearer token | See Part B |

> **Claude Desktop (desktop app):** Claude Desktop's `claude_desktop_config.json` only supports local STDIO servers. This server uses HTTP transport. Use Claude.ai web or Claude Code CLI instead.

---

## Part A — Claude.ai Web (OAuth 2.0)

### Step 1: Add the custom connector

1. Go to [claude.ai](https://claude.ai) → **Settings** (bottom-left) → **Integrations**
2. Click **Add custom integration**
3. Fill in:
   - **Name**: `Hudu`
   - **URL**: `https://<your-mcp-hostname>/mcp`
   - **Authentication**: OAuth 2.0
4. Click **Connect**

### Step 2: Sign in with Microsoft

A browser tab will open and redirect you to the Microsoft login page. Sign in with your organizational account (the Azure AD tenant associated with the MCP server).

After signing in, the tab closes and Claude.ai shows the integration as active.

### Step 3: Test it

In a Claude.ai conversation, try:

```
Search for all companies in Hudu
```

or

```
Use hudu_search_company_information to find companies matching "acme"
```

Claude should call the tool and return results from your Hudu instance.

---

## Part B — Claude Code CLI (Bearer Token)

### Step 1: Get your Bearer token

Ask your administrator for the `MCP_BEARER_TOKEN` value from the server's `.env` file.

### Step 2: Add the server

```bash
claude mcp add --transport http hudu https://<your-mcp-hostname>/mcp \
  --header "Authorization: Bearer <your-token>"
```

### Step 3: Test it

```bash
claude
```

Then in the chat:

```
Use hudu_search_company_information to list all companies
```

---

## Troubleshooting

### Claude.ai shows "Couldn't reach the MCP server"
- Verify the URL is `https://<hostname>/mcp` (not just the hostname)
- Confirm the server is running: `curl https://<hostname>/health`
- Check server logs: `docker compose logs hudu-mcp-server`

### OAuth tab opens but shows an Azure error
- **AADSTS50011** (redirect URI mismatch): The callback URL hasn't been registered in Azure AD yet. Tell your administrator to check `SETUP.md` Step 3.3.
- **AADSTS65001** (consent not granted): Admin consent is required. Contact your Azure AD administrator.
- **AADSTS700016** (app not found): Wrong tenant — confirm you're signing in with the correct organizational account.

### "Invalid token" or 401 after OAuth completes
- Your session token may have expired. Remove the integration in Claude.ai Settings → Integrations and reconnect.

### Claude Code CLI: "unauthorized"
- Verify the Bearer token matches `MCP_BEARER_TOKEN` in the server's `.env`
- Check that `MCP_OAUTH_ENABLED=false` is set (Bearer token mode and OAuth mode are mutually exclusive)

### No Hudu tools visible
- Confirm the integration is showing as active (green) in Claude.ai Settings → Integrations
- Try asking: "What Hudu tools are available?"
- Rebuild if needed: `docker compose down && docker compose up -d --build`

---

## Need Help?

Contact your IT administrator and provide:
- The error message you're seeing
- Whether the issue occurs in Claude.ai web or Claude Code CLI
- Output of: `curl https://<your-mcp-hostname>/health`
