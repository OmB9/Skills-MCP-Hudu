# Skills-MCP-Hudu — Azure Container Apps Deployment Guide

**OAuth 2.1 + Entra ID · All secrets in Key Vault · Claude.ai web only**

---

## What This Guide Covers

This guide deploys the Skills-MCP-Hudu server to **Azure Container Apps (ACA)** using:

- **OAuth 2.1 with Entra ID** — Claude.ai web authenticates via your Azure AD tenant. No shared bearer tokens.
- **Key Vault for all config** — every environment variable, including static values, lives in Key Vault. Nothing is hardcoded or stored in shell history.
- **Managed Identity** — the Container App pulls secrets from Key Vault and images from ACR with no stored credentials.
- **Log Analytics** — all container logs stream automatically to Azure Monitor. Live tailing and historical queries included.

### What This Guide Does NOT Cover

- **Claude Code CLI / Claude Desktop** — these clients require a bearer token. OAuth-only mode does not support them. If you need CLI access, see the `MCP_BEARER_TOKEN` section in [README.md](README.md).
- **GitHub Actions CI/CD** — image builds and pushes are manual in this guide. Automating that is straightforward once the infrastructure is in place.

---

## Architecture

```
Claude.ai (web)
     │
     │  HTTPS  ← ACA provides TLS automatically, no Caddy needed
     ▼
Azure Container Apps (external ingress)
  {app}.{env}.{region}.azurecontainerapps.io  (or your custom domain)
     │
     ├── GET  /.well-known/oauth-protected-resource  ← RFC 9728
     ├── GET  /.well-known/oauth-authorization-server ← RFC 8414
     ├── POST /register                              ← RFC 7591 (returns Azure client_id)
     ├── GET  /authorize  → redirects to Entra ID   ← OAuth proxy
     ├── POST /token      → proxies to Entra ID     ← OAuth proxy
     └── POST /mcp        ← MCP JSON-RPC (JWT-validated)
          │
          ▼
     Key Vault (all secrets via Managed Identity)
          │
          ▼
     Hudu REST API
```

**Why no Caddy?** ACA terminates TLS and provides HTTPS automatically via a managed certificate on every app. Caddy is only needed for self-hosted deployments where you manage your own reverse proxy.

---

## Prerequisites

- **Azure CLI** installed and logged in (`az login`)
- **Docker Desktop** running
- **Azure AD app registration** completed — follow [AZURE_AD_SETUP.md](AZURE_AD_SETUP.md) before continuing. You need:
  - `AZURE_CLIENT_ID` (Application ID)
  - `AZURE_TENANT_ID` (Directory ID)
  - Redirect URI `https://claude.ai/api/mcp/auth_callback` registered under **Mobile and desktop applications**
  - `accessTokenAcceptedVersion: 2` set in the app manifest
  - API scope `api://{client-id}/access_as_user` created

```powershell
# Verify you're logged in to the right subscription
az account show
az account set --subscription "YOUR_SUBSCRIPTION_NAME_OR_ID"
```

---

## Variables

Set these once. Every subsequent command references them — no copy-pasting GUIDs.

```powershell
# === Customize these ===
$RESOURCE_GROUP       = "rg-mcp-hudu"
$LOCATION             = "eastus"           # eastus, eastus2, westus2, etc.
$ACR_NAME             = "acrmcphudu"       # Globally unique, alphanumeric only, 5-50 chars
$KEYVAULT_NAME        = "kv-mcp-hudu"      # Globally unique, 3-24 chars
$ACA_ENV_NAME         = "aca-env-mcp"
$ACA_APP_NAME         = "skills-mcp-hudu"
$IDENTITY_NAME        = "id-mcp-hudu"
$LAW_NAME             = "law-mcp-hudu"     # Log Analytics workspace

# === From your Azure AD app registration (AZURE_AD_SETUP.md) ===
$AZURE_TENANT_ID      = "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
$AZURE_CLIENT_ID      = "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
$AZURE_AUDIENCE       = "api://$AZURE_CLIENT_ID"

# === Your Hudu instance ===
$HUDU_BASE_URL        = "https://v1corp.huducloud.com"
$HUDU_API_KEY         = "BWxrqmqGvZNLY4DUk5zAWH9H"
```

> **Note:** `$AZURE_AUDIENCE` is constructed from `$AZURE_CLIENT_ID`. If token validation fails later, check the server logs for the actual `aud` claim and set this to match exactly.

---

## Phase 1 — Azure AD Pre-check

Before deploying infrastructure, confirm your app registration is correctly configured. A misconfigured app registration is the most common cause of OAuth failures.

```powershell
# Verify the app registration exists
az ad app show --id $AZURE_CLIENT_ID --query "{name:displayName, id:appId}" --output table

# Confirm the redirect URI is registered under Mobile/Desktop (not SPA or Web)
# Look for: "https://claude.ai/api/mcp/auth_callback" under mobileAndDesktopApplication
az ad app show --id $AZURE_CLIENT_ID --query "publicClient.redirectUris" --output json

# Confirm accessTokenAcceptedVersion is 2 in the manifest
# (You need to check this in the portal: App Registration → Manifest)
# It should read: "accessTokenAcceptedVersion": 2
# If it reads null or 1, update it before proceeding.
```

**Expected redirect URI output:**
```json
[
  "https://claude.ai/api/mcp/auth_callback"
]
```

If the redirect URI is missing, go to **Azure portal → App registrations → Authentication → Mobile and desktop applications → Add URI**.

---

## Phase 2 — Resource Group, ACR, and Key Vault

```powershell
# --- Resource Group ---
az group create `
  --name $RESOURCE_GROUP `
  --location $LOCATION

# --- Azure Container Registry ---
# admin-enabled false: we use Managed Identity to pull images, not admin credentials
az acr create `
  --resource-group $RESOURCE_GROUP `
  --name $ACR_NAME `
  --sku Basic `
  --admin-enabled false

# Save the login server URL for later (e.g. acrmcphudu.azurecr.io)
$ACR_LOGIN_SERVER = az acr show `
  --name $ACR_NAME `
  --query loginServer `
  --output tsv

Write-Host "ACR login server: $ACR_LOGIN_SERVER"

# --- Key Vault ---
# enable-rbac-authorization true: use Azure RBAC instead of legacy access policies.
# This is the recommended model and works cleanly with Managed Identity.
az keyvault create `
  --name $KEYVAULT_NAME `
  --resource-group $RESOURCE_GROUP `
  --location $LOCATION `
  --enable-rbac-authorization true

# Save the Key Vault resource ID for role assignments
$KV_RESOURCE_ID = az keyvault show `
  --name $KEYVAULT_NAME `
  --query id `
  --output tsv
```

---

## Phase 3 — Log Analytics Workspace and Managed Identity

### Log Analytics

ACA streams all container stdout/stderr to Log Analytics automatically when a workspace is attached to the Container Apps Environment. The MCP server logs every tool call, OAuth event, and error to stdout via Winston — all of it will appear here.

```powershell
az monitor log-analytics workspace create `
  --resource-group $RESOURCE_GROUP `
  --workspace-name $LAW_NAME `
  --location $LOCATION

# Capture the workspace ID and key — needed when creating the ACA Environment
$LAW_CUSTOMER_ID = az monitor log-analytics workspace show `
  --resource-group $RESOURCE_GROUP `
  --workspace-name $LAW_NAME `
  --query customerId `
  --output tsv

$LAW_KEY = az monitor log-analytics workspace get-shared-keys `
  --resource-group $RESOURCE_GROUP `
  --workspace-name $LAW_NAME `
  --query primarySharedKey `
  --output tsv
```

### Managed Identity

The Container App uses this identity to pull images from ACR and read secrets from Key Vault. No credentials are stored anywhere.

```powershell
az identity create `
  --name $IDENTITY_NAME `
  --resource-group $RESOURCE_GROUP

# The principal ID is used for role assignments
$IDENTITY_PRINCIPAL_ID = az identity show `
  --name $IDENTITY_NAME `
  --resource-group $RESOURCE_GROUP `
  --query principalId `
  --output tsv

# The resource ID is used when attaching the identity to the Container App
$IDENTITY_RESOURCE_ID = az identity show `
  --name $IDENTITY_NAME `
  --resource-group $RESOURCE_GROUP `
  --query id `
  --output tsv

# --- Role: Key Vault Secrets User ---
# Allows the identity to read secret values (not manage secrets)
az role assignment create `
  --assignee $IDENTITY_PRINCIPAL_ID `
  --role "Key Vault Secrets User" `
  --scope $KV_RESOURCE_ID

# --- Role: AcrPull ---
# Allows the identity to pull container images from ACR
$ACR_RESOURCE_ID = az acr show `
  --name $ACR_NAME `
  --query id `
  --output tsv

az role assignment create `
  --assignee $IDENTITY_PRINCIPAL_ID `
  --role "AcrPull" `
  --scope $ACR_RESOURCE_ID

Write-Host "Identity principal ID: $IDENTITY_PRINCIPAL_ID"
Write-Host "Identity resource ID:  $IDENTITY_RESOURCE_ID"
```

> Role assignments can take 1–2 minutes to propagate. If you hit permission errors in later phases, wait a moment and retry.

---

## Phase 4 — Container Apps Environment

The environment is the shared network and logging boundary for all Container Apps within it.

```powershell
# Install or upgrade the Container Apps extension
az extension add --name containerapp --upgrade

az containerapp env create `
  --name $ACA_ENV_NAME `
  --resource-group $RESOURCE_GROUP `
  --location $LOCATION `
  --logs-workspace-id $LAW_CUSTOMER_ID `
  --logs-workspace-key $LAW_KEY
```

This creates the environment and wires it to your Log Analytics workspace. All container logs will flow there automatically.

---

## Phase 5 — Store All Secrets in Key Vault

Every configuration value lives in Key Vault — no plaintext values anywhere else. The Container App will reference all of these via `secretref:`.

```powershell
# --- Hudu ---
az keyvault secret set --vault-name $KEYVAULT_NAME --name "HUDU-API-KEY"    --value $HUDU_API_KEY
az keyvault secret set --vault-name $KEYVAULT_NAME --name "HUDU-BASE-URL"   --value $HUDU_BASE_URL
az keyvault secret set --vault-name $KEYVAULT_NAME --name "HUDU-TIMEOUT"    --value "30000"
az keyvault secret set --vault-name $KEYVAULT_NAME --name "HUDU-ALLOWED-COMPANY-IDS" --value "ALL"

# --- Azure AD / OAuth ---
az keyvault secret set --vault-name $KEYVAULT_NAME --name "AZURE-TENANT-ID" --value $AZURE_TENANT_ID
az keyvault secret set --vault-name $KEYVAULT_NAME --name "AZURE-CLIENT-ID" --value $AZURE_CLIENT_ID
az keyvault secret set --vault-name $KEYVAULT_NAME --name "AZURE-AUDIENCE"  --value $AZURE_AUDIENCE

# --- MCP Server ---
az keyvault secret set --vault-name $KEYVAULT_NAME --name "MCP-OAUTH-ENABLED" --value "true"
az keyvault secret set --vault-name $KEYVAULT_NAME --name "MCP-SERVER-PORT"   --value "3100"

# --- Runtime ---
az keyvault secret set --vault-name $KEYVAULT_NAME --name "NODE-ENV"   --value "production"
az keyvault secret set --vault-name $KEYVAULT_NAME --name "LOG-LEVEL"  --value "info"

# Note: MCP-SERVER-URL is set in Phase 8 after the ACA FQDN is known.
```

### Retrieve Secret URIs

ACA Key Vault references require the versioned secret URI. Retrieve all of them now:

```powershell
$KV_URI_HUDU_KEY    = az keyvault secret show --vault-name $KEYVAULT_NAME --name "HUDU-API-KEY"             --query id --output tsv
$KV_URI_HUDU_URL    = az keyvault secret show --vault-name $KEYVAULT_NAME --name "HUDU-BASE-URL"            --query id --output tsv
$KV_URI_TIMEOUT     = az keyvault secret show --vault-name $KEYVAULT_NAME --name "HUDU-TIMEOUT"             --query id --output tsv
$KV_URI_COMPANY_IDS = az keyvault secret show --vault-name $KEYVAULT_NAME --name "HUDU-ALLOWED-COMPANY-IDS" --query id --output tsv
$KV_URI_TENANT_ID   = az keyvault secret show --vault-name $KEYVAULT_NAME --name "AZURE-TENANT-ID"          --query id --output tsv
$KV_URI_CLIENT_ID   = az keyvault secret show --vault-name $KEYVAULT_NAME --name "AZURE-CLIENT-ID"          --query id --output tsv
$KV_URI_AUDIENCE    = az keyvault secret show --vault-name $KEYVAULT_NAME --name "AZURE-AUDIENCE"           --query id --output tsv
$KV_URI_OAUTH       = az keyvault secret show --vault-name $KEYVAULT_NAME --name "MCP-OAUTH-ENABLED"        --query id --output tsv
$KV_URI_PORT        = az keyvault secret show --vault-name $KEYVAULT_NAME --name "MCP-SERVER-PORT"          --query id --output tsv
$KV_URI_NODE_ENV    = az keyvault secret show --vault-name $KEYVAULT_NAME --name "NODE-ENV"                 --query id --output tsv
$KV_URI_LOG_LEVEL   = az keyvault secret show --vault-name $KEYVAULT_NAME --name "LOG-LEVEL"               --query id --output tsv
```

---

## Phase 6 — Build and Push the Docker Image

```powershell
# Authenticate Docker to your ACR using your Azure login (no admin password needed)
az acr login --name $ACR_NAME

# Build from the repo root
docker build -t "$ACR_LOGIN_SERVER/skills-mcp-hudu:latest" .

# Push to ACR
docker push "$ACR_LOGIN_SERVER/skills-mcp-hudu:latest"

# Verify the image is there
az acr repository list --name $ACR_NAME --output table
```

---

## Phase 7 — Deploy Container App (Step 1: without MCP_SERVER_URL)

### Why two steps?

`MCP_SERVER_URL` must be set to the Container App's public HTTPS URL — it appears in the OAuth discovery documents that Claude.ai reads. But ACA generates this URL only after the app is created.

In this step, we create the app with all secrets except `MCP_SERVER_URL`. The server starts and passes health checks, but OAuth will not work yet because the discovery endpoints return `http://localhost:3100` as the server URL. **Do not connect Claude.ai yet.**

```powershell
az containerapp create `
  --name $ACA_APP_NAME `
  --resource-group $RESOURCE_GROUP `
  --environment $ACA_ENV_NAME `
  --image "$ACR_LOGIN_SERVER/skills-mcp-hudu:latest" `
  --registry-server $ACR_LOGIN_SERVER `
  --registry-identity $IDENTITY_RESOURCE_ID `
  --user-assigned $IDENTITY_RESOURCE_ID `
  --target-port 3100 `
  --ingress external `
  --min-replicas 1 `
  --max-replicas 3 `
  --cpu 0.5 `
  --memory 1.0Gi `
  --secrets `
    "hudu-api-key=keyvaultref:$KV_URI_HUDU_KEY,identityref:$IDENTITY_RESOURCE_ID" `
    "hudu-base-url=keyvaultref:$KV_URI_HUDU_URL,identityref:$IDENTITY_RESOURCE_ID" `
    "hudu-timeout=keyvaultref:$KV_URI_TIMEOUT,identityref:$IDENTITY_RESOURCE_ID" `
    "hudu-company-ids=keyvaultref:$KV_URI_COMPANY_IDS,identityref:$IDENTITY_RESOURCE_ID" `
    "azure-tenant-id=keyvaultref:$KV_URI_TENANT_ID,identityref:$IDENTITY_RESOURCE_ID" `
    "azure-client-id=keyvaultref:$KV_URI_CLIENT_ID,identityref:$IDENTITY_RESOURCE_ID" `
    "azure-audience=keyvaultref:$KV_URI_AUDIENCE,identityref:$IDENTITY_RESOURCE_ID" `
    "mcp-oauth-enabled=keyvaultref:$KV_URI_OAUTH,identityref:$IDENTITY_RESOURCE_ID" `
    "mcp-server-port=keyvaultref:$KV_URI_PORT,identityref:$IDENTITY_RESOURCE_ID" `
    "node-env=keyvaultref:$KV_URI_NODE_ENV,identityref:$IDENTITY_RESOURCE_ID" `
    "log-level=keyvaultref:$KV_URI_LOG_LEVEL,identityref:$IDENTITY_RESOURCE_ID" `
  --env-vars `
    "HUDU_API_KEY=secretref:hudu-api-key" `
    "HUDU_BASE_URL=secretref:hudu-base-url" `
    "HUDU_TIMEOUT=secretref:hudu-timeout" `
    "HUDU_ALLOWED_COMPANY_IDS=secretref:hudu-company-ids" `
    "AZURE_TENANT_ID=secretref:azure-tenant-id" `
    "AZURE_CLIENT_ID=secretref:azure-client-id" `
    "AZURE_AUDIENCE=secretref:azure-audience" `
    "MCP_OAUTH_ENABLED=secretref:mcp-oauth-enabled" `
    "MCP_SERVER_PORT=secretref:mcp-server-port" `
    "NODE_ENV=secretref:node-env" `
    "LOG_LEVEL=secretref:log-level"
```

### Confirm it started

```powershell
# Get the public FQDN — save this, you'll need it throughout
$MCP_FQDN = az containerapp show `
  --name $ACA_APP_NAME `
  --resource-group $RESOURCE_GROUP `
  --query properties.configuration.ingress.fqdn `
  --output tsv

Write-Host "Container App URL: https://$MCP_FQDN"

# Health check — should return {"status":"healthy","transport":"streamable-http",...}
Invoke-RestMethod -Uri "https://$MCP_FQDN/health"
```

If the health check returns 200, the container is running and Key Vault access is working. Proceed to Phase 8.

---

## Phase 8 — Set MCP_SERVER_URL (Step 2: complete OAuth config)

Now that we have the FQDN, store it in Key Vault and wire it into the Container App. This is what makes the OAuth discovery endpoints return the correct URLs.

```powershell
$MCP_SERVER_URL = "https://$MCP_FQDN"

# Store in Key Vault
az keyvault secret set `
  --vault-name $KEYVAULT_NAME `
  --name "MCP-SERVER-URL" `
  --value $MCP_SERVER_URL

# Get the versioned URI
$KV_URI_MCP_URL = az keyvault secret show `
  --vault-name $KEYVAULT_NAME `
  --name "MCP-SERVER-URL" `
  --query id `
  --output tsv

# Add the secret reference to the Container App
az containerapp secret set `
  --name $ACA_APP_NAME `
  --resource-group $RESOURCE_GROUP `
  --secrets "mcp-server-url=keyvaultref:$KV_URI_MCP_URL,identityref:$IDENTITY_RESOURCE_ID"

# Wire the env var to the new secret — this triggers a new revision automatically
az containerapp update `
  --name $ACA_APP_NAME `
  --resource-group $RESOURCE_GROUP `
  --set-env-vars "MCP_SERVER_URL=secretref:mcp-server-url"

Write-Host "MCP Server URL: $MCP_SERVER_URL"
Write-Host "MCP Endpoint:   $MCP_SERVER_URL/mcp"
```

### Verify the OAuth discovery documents

```powershell
# Protected resource — authorization_servers must equal ["https://{your-fqdn}"]
$pr = Invoke-RestMethod -Uri "https://$MCP_FQDN/.well-known/oauth-protected-resource"
Write-Host "authorization_servers: $($pr.authorization_servers)"
# Expected: ["https://skills-mcp-hudu.{env}.{region}.azurecontainerapps.io"]

# Authorization server metadata — issuer must equal "https://{your-fqdn}"
$as = Invoke-RestMethod -Uri "https://$MCP_FQDN/.well-known/oauth-authorization-server"
Write-Host "issuer:                $($as.issuer)"
Write-Host "authorization_endpoint: $($as.authorization_endpoint)"
Write-Host "token_endpoint:         $($as.token_endpoint)"
# Expected: all three contain your ACA FQDN, not login.microsoftonline.com
```

If `issuer` still shows `http://localhost:3100`, the new revision hasn't finished deploying. Wait 30 seconds and retry.

---

## Phase 9 — Connect Claude.ai

1. Go to **Claude.ai → Settings → Integrations → Add custom integration**
2. Enter URL: `https://{your-fqdn}/mcp`
3. Select **OAuth 2.0**
4. Click **Connect** — a Microsoft login page should appear
5. Sign in with your Entra ID account
6. The integration should show as **Connected**

**Test it:**
```
Hudu, list all companies
```

---

## Optional: Custom Domain

If you want a predictable, branded URL (e.g. `mcp.yourcompany.com`) instead of the ACA-generated FQDN:

```powershell
# After Phase 7, add your custom hostname to the Container App
az containerapp hostname add `
  --name $ACA_APP_NAME `
  --resource-group $RESOURCE_GROUP `
  --hostname "mcp.yourcompany.com"

# The command will output a verification token and CNAME target.
# In your DNS provider, create:
#   CNAME  mcp.yourcompany.com  →  {your-fqdn}.azurecontainerapps.io
#   TXT    asuid.mcp            →  {verification-token from above}

# Once DNS propagates, bind the managed certificate
az containerapp hostname bind `
  --name $ACA_APP_NAME `
  --resource-group $RESOURCE_GROUP `
  --hostname "mcp.yourcompany.com" `
  --environment $ACA_ENV_NAME `
  --validation-method CNAME
```

Then update `MCP-SERVER-URL` in Key Vault to your custom domain:

```powershell
az keyvault secret set `
  --vault-name $KEYVAULT_NAME `
  --name "MCP-SERVER-URL" `
  --value "https://mcp.yourcompany.com"

# The Container App picks up the new secret value on next restart.
# Update the Container App to trigger a new revision:
az containerapp update `
  --name $ACA_APP_NAME `
  --resource-group $RESOURCE_GROUP `
  --image "$ACR_LOGIN_SERVER/skills-mcp-hudu:latest"
```

> **Advantage of a custom domain:** The URL doesn't change if you tear down and redeploy. Useful for sharing the endpoint with a team — they only need to update their client config once.

---

## Maintenance

### View Live Logs

```powershell
# Stream live from the Container App
az containerapp logs show `
  --name $ACA_APP_NAME `
  --resource-group $RESOURCE_GROUP `
  --follow

# Or query Log Analytics for historical logs
# In the Azure portal: Log Analytics → $LAW_NAME → Logs
# Sample Kusto query:
# ContainerAppConsoleLogs_CL
# | where ContainerAppName_s == "skills-mcp-hudu"
# | project TimeGenerated, Log_s
# | order by TimeGenerated desc
# | take 100
```

### Update to a New Image Version

```powershell
cd /path/to/Skills-MCP-Hudu
git pull

az acr login --name $ACR_NAME
docker build -t "$ACR_LOGIN_SERVER/skills-mcp-hudu:latest" .
docker push "$ACR_LOGIN_SERVER/skills-mcp-hudu:latest"

# Force a new revision with the updated image
az containerapp update `
  --name $ACA_APP_NAME `
  --resource-group $RESOURCE_GROUP `
  --image "$ACR_LOGIN_SERVER/skills-mcp-hudu:latest"
```

### Rotate the Hudu API Key

```powershell
# Set the new value in Key Vault
az keyvault secret set `
  --vault-name $KEYVAULT_NAME `
  --name "HUDU-API-KEY" `
  --value "NEW_API_KEY_HERE"

# The Container App reads secrets on startup. Force a restart by updating the image:
az containerapp update `
  --name $ACA_APP_NAME `
  --resource-group $RESOURCE_GROUP `
  --image "$ACR_LOGIN_SERVER/skills-mcp-hudu:latest"
```

### Restrict Access to Specific Companies (Multi-Tenant)

```powershell
# Allow only specific Hudu company IDs (comma-separated)
az keyvault secret set `
  --vault-name $KEYVAULT_NAME `
  --name "HUDU-ALLOWED-COMPANY-IDS" `
  --value "123,456,789"

az containerapp update `
  --name $ACA_APP_NAME `
  --resource-group $RESOURCE_GROUP `
  --image "$ACR_LOGIN_SERVER/skills-mcp-hudu:latest"
```

### Teardown

```powershell
# Remove all resources at once — Key Vault uses soft delete by default (90-day recovery)
az group delete --name $RESOURCE_GROUP --yes --no-wait

# If you want to permanently delete the Key Vault immediately:
az keyvault purge --name $KEYVAULT_NAME --location $LOCATION
```

---

## Troubleshooting

### "Couldn't reach the MCP server" in Claude.ai

Check that `MCP_SERVER_URL` is set correctly:

```powershell
$as = Invoke-RestMethod -Uri "https://$MCP_FQDN/.well-known/oauth-authorization-server"
$as.issuer
# Must equal "https://{your-fqdn}", not "http://localhost:3100"
```

If it shows `localhost`, `MCP-SERVER-URL` in Key Vault wasn't picked up. Check:
1. The secret exists: `az keyvault secret show --vault-name $KEYVAULT_NAME --name "MCP-SERVER-URL"`
2. The Container App has the env var: `az containerapp show --name $ACA_APP_NAME --resource-group $RESOURCE_GROUP --query "properties.template.containers[0].env"`
3. Force a new revision: `az containerapp update --name $ACA_APP_NAME --resource-group $RESOURCE_GROUP --image "$ACR_LOGIN_SERVER/skills-mcp-hudu:latest"`

### OAuth Opens but Shows AADSTS50011 (Redirect URI Mismatch)

The URI `https://claude.ai/api/mcp/auth_callback` is not registered in your Azure AD app.

Go to: **Azure portal → App registrations → {your app} → Authentication → Mobile and desktop applications → Add URI**

Add exactly: `https://claude.ai/api/mcp/auth_callback`

### OAuth Completes but Token Validation Fails

Check server logs for the actual audience in the token:

```powershell
az containerapp logs show `
  --name $ACA_APP_NAME `
  --resource-group $RESOURCE_GROUP `
  --follow
# Look for: "Bearer token validation failed" with "tokenAudience" field
```

Update `AZURE-AUDIENCE` in Key Vault to match the `tokenAudience` value exactly, then restart:

```powershell
az keyvault secret set `
  --vault-name $KEYVAULT_NAME `
  --name "AZURE-AUDIENCE" `
  --value "THE_ACTUAL_AUDIENCE_FROM_LOGS"

az containerapp update `
  --name $ACA_APP_NAME `
  --resource-group $RESOURCE_GROUP `
  --image "$ACR_LOGIN_SERVER/skills-mcp-hudu:latest"
```

### Container Fails to Start (Key Vault Access Denied)

The Managed Identity doesn't have the `Key Vault Secrets User` role yet (role assignments can take a few minutes to propagate).

```powershell
# Verify the role assignment exists
az role assignment list `
  --assignee $IDENTITY_PRINCIPAL_ID `
  --role "Key Vault Secrets User" `
  --scope $KV_RESOURCE_ID `
  --output table
```

If the assignment is missing, re-run the role assignment from Phase 3 and wait 2 minutes before redeploying.

### `accessTokenAcceptedVersion` Resets

Azure occasionally resets this during manifest edits. If token issuer validation starts failing after working previously, re-check the app manifest in the portal and add `"accessTokenAcceptedVersion": 2` again.

---

## Cost Estimate

| Resource | Tier | Approx. Monthly |
|---|---|---|
| Azure Container Apps | Consumption (1 replica, 0.5 vCPU / 1 GB) | ~$10–15 |
| Azure Container Registry | Basic | ~$5 |
| Azure Key Vault | Standard (< 10K ops/mo) | ~$0.03 |
| Log Analytics | Pay-per-GB (low volume) | ~$2–5 |
| **Total** | | **~$17–25/month** |

---

## Security Summary

| Control | Implementation |
|---|---|
| No secrets in source control | All values in Key Vault |
| No secrets in shell history | Variables set once, referenced via `$KV_URI_*` |
| No admin credentials for ACR | Managed Identity with AcrPull role |
| Transport encryption | ACA enforces HTTPS, no Caddy needed |
| Authentication | Short-lived Azure AD JWTs (OAuth 2.1 with PKCE) |
| No shared bearer tokens | OAuth-only; no long-lived static secret |
| Audit trail | Log Analytics captures all tool calls and auth events |
| Company scope control | `HUDU_ALLOWED_COMPANY_IDS` in Key Vault |

---

*Based on Skills-MCP-Hudu v1.1.0 — March 2026*
*OAuth proxy implementation: [ARCHITECTURE.md](ARCHITECTURE.md)*
*Azure AD setup: [AZURE_AD_SETUP.md](AZURE_AD_SETUP.md)*
