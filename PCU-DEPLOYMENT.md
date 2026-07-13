# Falcon MCP — Azure Container Apps + Microsoft Copilot Studio Deployment Guide

This document describes the configuration required to host the Falcon MCP server on Azure Container Apps and connect it to a Microsoft Copilot Studio agent. It covers infrastructure setup, Entra ID app registration configuration, GitHub Actions deployment, and tool configuration options.

---

## Architecture Overview

```
Microsoft Copilot Studio Agent
        │
        │  OAuth 2.0 (client app credentials)
        ▼
Power Apps Custom Connector
        │
        │  Bearer token (scoped to server app)
        ▼
Azure Container Apps (Easy Auth — validates token against server app)
        │
        │  Authenticated request
        ▼
falcon-mcp server (streamable-http transport, port 8000)
        │
        │  CrowdStrike API credentials
        ▼
CrowdStrike Falcon API
```

---

## Prerequisites

- Azure subscription with Contributor access
- Microsoft 365 Copilot licence for agent users
- CrowdStrike Falcon API client credentials with appropriate scopes
- GitHub repository (fork of CrowdStrike/falcon-mcp)
- Azure CLI installed or access to Azure Cloud Shell

---

## 1. Entra ID App Registrations

Two app registrations are required: a **server app** (the protected resource) and a **client app** (used by the Power Apps connector to obtain tokens).

### 1.1 Server App Registration

This represents the Falcon MCP Container App endpoint.

1. In Entra ID go to **App registrations → New registration**
   - Name: `falcon-mcp-server`
   - Supported account types: Single tenant
   - Click **Register**
   - Note the **Application (client) ID** — referred to as `SERVER_APP_ID`

2. **Expose an API**
   - Set Application ID URI — accept default `api://SERVER_APP_ID`
   - Add a scope:
     - Scope name: `access_as_user`
     - Who can consent: Admins and users
     - Admin consent display name: `Access Falcon MCP Server`
     - Admin consent description: `Allows access to the Falcon MCP Server on behalf of the signed-in user`
     - State: Enabled
   - Note the scope GUID that is auto-generated

3. **Authorized client applications**
   - Add the client app `CLIENT_APP_ID` → select `access_as_user` scope
   - Add the Azure API Connections service principal `fe053c5f-3692-4f14-aef2-ee34fc081cae` → select `access_as_user` scope

4. **Certificates & secrets**
   - Create a new client secret and note the value — referred to as `SERVER_APP_SECRET`
   - This is used by the Container App auth configuration

### 1.2 Client App Registration

This is what the Power Apps custom connector authenticates as when calling the server.

1. In Entra ID go to **App registrations → New registration**
   - Name: `falcon-mcp-client`
   - Supported account types: Single tenant
   - Click **Register**
   - Note the **Application (client) ID** — referred to as `CLIENT_APP_ID`

2. **Authentication**
   - Add platform → Web
   - Redirect URI: copy from the Copilot Studio MCP onboarding wizard (format: `https://global.consent.azure-apim.net/redirect/...`)
   - Save

3. **Certificates & secrets**
   - Create a new client secret — referred to as `CLIENT_SECRET`
   - Copy the value immediately, it cannot be retrieved again

4. **API permissions**
   - Add → My APIs → `falcon-mcp-server` → Delegated → `access_as_user`
   - Add → Microsoft Graph → Delegated → `openid`, `profile`, `offline_access`, `User.Read`
   - Click **Grant admin consent for [tenant]**

5. **Expose an API**
   - Set Application ID URI — accept default `api://CLIENT_APP_ID`
   - Add a scope:
     - Scope name: `access_as_user`
     - Who can consent: Admins and users
     - Admin consent display name: `Access Falcon MCP Client`
     - State: Enabled

6. **Authorized client applications**
   - Add Azure API Connections: `fe053c5f-3692-4f14-aef2-ee34fc081cae` → select `access_as_user` scope

### 1.3 Admin Consent via PowerShell

If your tenant blocks browser-based consent flows (e.g. Conditional Access requiring a managed device), grant consent programmatically from a managed device:

```powershell
Connect-MgGraph -TenantId "YOUR_TENANT_ID" -Scopes "Application.ReadWrite.All","DelegatedPermissionGrant.ReadWrite.All"

$clientSp = Get-MgServicePrincipal -Filter "appId eq 'CLIENT_APP_ID'"
$serverSp = Get-MgServicePrincipal -Filter "appId eq 'SERVER_APP_ID'"
$graphSp  = Get-MgServicePrincipal -Filter "appId eq '00000003-0000-0000-c000-000000000000'"

New-MgOauth2PermissionGrant -ClientId $clientSp.Id -ConsentType "AllPrincipals" -ResourceId $serverSp.Id -Scope "access_as_user"
New-MgOauth2PermissionGrant -ClientId $clientSp.Id -ConsentType "AllPrincipals" -ResourceId $graphSp.Id  -Scope "openid profile offline_access User.Read"
```

---

## 2. Azure Infrastructure

Run the following in Azure Cloud Shell or with the Azure CLI. Adjust `--location` and `--resource-group` to match your environment.

```bash
# Container Registry
az acr create \
  --name YOUR_ACR_NAME \
  --resource-group YOUR_RESOURCE_GROUP \
  --sku Basic \
  --admin-enabled true

# Container Apps environment
az containerapp env create \
  --name falcon-mcp-env \
  --resource-group YOUR_RESOURCE_GROUP \
  --location YOUR_REGION

# Container App (placeholder image — replaced on first GitHub Actions deploy)
az containerapp create \
  --name falcon-mcp-app \
  --resource-group YOUR_RESOURCE_GROUP \
  --environment falcon-mcp-env \
  --image mcr.microsoft.com/azuredocs/containerapps-helloworld:latest \
  --target-port 8000 \
  --ingress external \
  --min-replicas 1 \
  --registry-server YOUR_ACR_NAME.azurecr.io \
  --env-vars \
    FALCON_CLIENT_ID=YOUR_CROWDSTRIKE_CLIENT_ID \
    FALCON_CLIENT_SECRET=YOUR_CROWDSTRIKE_CLIENT_SECRET \
    FALCON_BASE_URL=https://api.crowdstrike.com

# Enable Entra ID authentication on the Container App
az containerapp auth update \
  --name falcon-mcp-app \
  --resource-group YOUR_RESOURCE_GROUP \
  --unauthenticated-client-action Return401

az containerapp auth microsoft update \
  --name falcon-mcp-app \
  --resource-group YOUR_RESOURCE_GROUP \
  --client-id SERVER_APP_ID \
  --client-secret SERVER_APP_SECRET \
  --issuer https://login.microsoftonline.com/YOUR_TENANT_ID/v2.0 \
  --allowed-audiences api://SERVER_APP_ID
```

After running, set the allowed applications in the Azure Portal:

**Container App → Authentication → Edit Microsoft provider → Allowed applications**

Add `CLIENT_APP_ID` and remove any previous GUIDs.

### Verify configuration

```bash
az containerapp auth show --name falcon-mcp-app --resource-group YOUR_RESOURCE_GROUP --output json
```

Confirm:
- `platform.enabled: true`
- `globalValidation.unauthenticatedClientAction: Return401`
- `identityProviders.azureActiveDirectory.registration.clientId: SERVER_APP_ID`
- `identityProviders.azureActiveDirectory.validation.allowedAudiences: ["api://SERVER_APP_ID"]`
- `identityProviders.azureActiveDirectory.validation.defaultAuthorizationPolicy.allowedApplications: ["CLIENT_APP_ID"]`

---

## 3. Dockerfile

The existing `Dockerfile` in this repository uses a multi-stage build. The only modification required is the `CMD` line at the end, which must pass the correct transport arguments:

```dockerfile
ENTRYPOINT ["falcon-mcp"]
CMD ["--transport", "streamable-http", "--host", "0.0.0.0", "--port", "8000"]
```

This starts the server in streamable-http mode, which is required for Azure Container Apps and Copilot Studio compatibility.

---

## 4. GitHub Actions Deployment

### 4.1 Required secrets

Add the following secrets to your GitHub repository under **Settings → Secrets and variables → Actions**:

| Secret | Description |
|---|---|
| `AZURE_CREDENTIALS` | Service principal JSON from `az ad sp create-for-rbac` |

The CrowdStrike credentials (`FALCON_CLIENT_ID`, `FALCON_CLIENT_SECRET`) are set directly as Container App environment variables rather than GitHub secrets.

### 4.2 Create the service principal

```bash
az ad sp create-for-rbac \
  --name "falcon-mcp-deploy" \
  --role contributor \
  --scopes /subscriptions/YOUR_SUBSCRIPTION_ID/resourceGroups/YOUR_RESOURCE_GROUP \
  --json-auth
```

Copy the entire JSON output into the `AZURE_CREDENTIALS` GitHub secret.

### 4.3 Workflow file

Save as `.github/workflows/deploy.yml`:

```yaml
name: Deploy Falcon MCP to Azure Container Apps

on:
  push:
    branches: [ main ]
  workflow_dispatch:

env:
  ACR_NAME: YOUR_ACR_NAME
  CONTAINER_APP_NAME: falcon-mcp-app
  RESOURCE_GROUP: YOUR_RESOURCE_GROUP
  IMAGE_NAME: falcon-mcp

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Login to Azure
        uses: azure/login@v2
        with:
          creds: ${{ secrets.AZURE_CREDENTIALS }}

      - name: Login to ACR
        run: az acr login --name ${{ env.ACR_NAME }}

      - name: Build and push image
        run: |
          docker build -t ${{ env.ACR_NAME }}.azurecr.io/${{ env.IMAGE_NAME }}:${{ github.sha }} .
          docker push ${{ env.ACR_NAME }}.azurecr.io/${{ env.IMAGE_NAME }}:${{ github.sha }}

      - name: Deploy to Container App
        run: az containerapp update --name ${{ env.CONTAINER_APP_NAME }} --resource-group ${{ env.RESOURCE_GROUP }} --image ${{ env.ACR_NAME }}.azurecr.io/${{ env.IMAGE_NAME }}:${{ github.sha }}
```

---

## 5. Copilot Studio Configuration

### 5.1 Add the MCP tool

In Copilot Studio go to your agent → **Tools → Add a tool → Model Context Protocol**:

- **Server name:** Falcon MCP
- **Server URL:** `https://YOUR_CONTAINER_APP_FQDN/mcp`
- **Authentication:** OAuth 2.0 → Manual

### 5.2 OAuth settings (Power Apps custom connector)

In Power Apps → Custom connectors → your connector → **Security tab**:

| Field | Value |
|---|---|
| Identity provider | Azure Active Directory |
| Client ID | `CLIENT_APP_ID` |
| Client secret | `CLIENT_SECRET` |
| Tenant ID | `YOUR_TENANT_ID` |
| Scope | `api://SERVER_APP_ID/access_as_user` |
| Refresh URL | `https://login.microsoftonline.com/YOUR_TENANT_ID/oauth2/v2.0/token` |

### 5.3 Agent authentication settings

In Copilot Studio → your agent → **Settings → Security → Authentication**:

- Authentication: **Authenticate with Microsoft**
- Enable Teams SSO: **On**
- Token exchange URL: `api://CLIENT_APP_ID/access_as_user`

---

## 6. Tool Configuration Options

By default falcon-mcp loads all available modules (23 modules, 112 tools as of v0.12.0). Copilot Studio limits MCP servers to 70 tools. The following options are available to manage this.

### Option A — Dynamic Mode (recommended)

Dynamic mode replaces all tools with three meta-tools:
- `falcon_list_enabled_modules` — lists available modules
- `falcon_search_tools` — discovers the right tool for a task
- `falcon_execute_tool` — executes a discovered tool

This keeps the tool count at 3 regardless of how many modules are loaded, and lets the agent discover the right tool at runtime. Enable it with:

```bash
az containerapp update \
  --name falcon-mcp-app \
  --resource-group YOUR_RESOURCE_GROUP \
  --set-env-vars FALCON_MCP_DYNAMIC=true
```

### Option B — Restrict modules

Load only specific modules using the `FALCON_MCP_MODULES` environment variable (comma-separated, no spaces):

```bash
az containerapp update \
  --name falcon-mcp-app \
  --resource-group YOUR_RESOURCE_GROUP \
  --set-env-vars FALCON_MCP_MODULES="detections,incidents,alerts,hosts,device_control,identity_protection,zero_trust_assessment"
```

#### Module reference

| Module name | Capability | Use case |
|---|---|---|
| `detections` | Search and retrieve detections | Threat detection and response |
| `incidents` | Search and manage incidents | Incident response |
| `alerts` | Search and retrieve alerts | Alert triage |
| `hosts` | Search and manage endpoints | Endpoint visibility |
| `device_control` | Device control policies | Endpoint management |
| `identity_protection` | Identity-based threat detection | Identity threat protection |
| `zero_trust_assessment` | Zero trust posture scoring | Identity and access |
| `intel` | Threat intelligence, IOCs, actors | Threat intelligence |
| `spotlight` | Vulnerability management | Vulnerability tracking |
| `discover` | Asset inventory | Asset management |
| `cloud` | Cloud workload protection | Cloud security |
| `ngsiem` | Next-gen SIEM queries | Threat hunting |
| `overwatch` | Managed detection reports | MDR integration |

For a full and current module list see the [Module Overview](https://developer.crowdstrike.com/falcon-mcp/modules/overview/) on the CrowdStrike Developer Center.

### Additional environment variables

| Variable | Default | Description |
|---|---|---|
| `FALCON_MCP_DYNAMIC` | `false` | Enable dynamic tool discovery mode |
| `FALCON_MCP_MODULES` | all | Comma-separated list of modules to load |
| `FALCON_MCP_TRANSPORT` | `stdio` | Transport: `stdio`, `sse`, `streamable-http` |
| `FALCON_MCP_HOST` | `127.0.0.1` | Host to bind to (use `0.0.0.0` for containers) |
| `FALCON_MCP_PORT` | `8000` | Port to bind to |
| `FALCON_MCP_STATELESS_HTTP` | `false` | Enable stateless HTTP mode for scaled deployments |
| `FALCON_MCP_DEBUG` | `false` | Enable debug logging |
| `FALCON_BASE_URL` | `https://api.crowdstrike.com` | CrowdStrike API base URL (adjust for non-US-1 regions) |
| `FALCON_MEMBER_CID` | — | Flight Control child CID (MSSP environments only) |

---

## 7. Keeping the Container Running

Set `--min-replicas 1` to prevent cold starts:

```bash
az containerapp update \
  --name falcon-mcp-app \
  --resource-group YOUR_RESOURCE_GROUP \
  --min-replicas 1
```

---

## 8. Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| `forbidden` on `initialize` | Wrong app GUID in Container App allowed applications | Update allowed applications in Azure Portal → Container App → Authentication |
| `notFound` on `initialize` | Wrong URL in Copilot Studio (e.g. `/map` instead of `/mcp`) | Correct the server URL in the MCP tool config |
| `LimitTools` warning | Server exposing more than 70 tools | Enable dynamic mode or restrict modules |
| Empty `[{"jsonrpc":"2.0"}]` response | Cold start or session state lost | Set `--min-replicas 1` |
| Repeated OAuth consent prompt | Admin consent not granted for all scopes | Grant consent via PowerShell (see section 1.3) |
| `AADSTS90009` error | Connector scope pointing at itself | Set scope to `api://SERVER_APP_ID/access_as_user`, not the client app URI |
| `Routes configuration is only allowed for worker runtime: custom` | Function App created with Python runtime | Not applicable for Container Apps deployment |

---

## 9. Notes

- This project is in public preview as of v0.12.0. CrowdStrike recommends avoiding production deployments until the stable 1.0 release.
- The Container App worker runtime must be set to `custom` if using Azure Functions (Flex Consumption). Azure Container Apps does not have this restriction and is the recommended hosting approach for Copilot Studio integration.
- Cost data for the Container App depends on the `--min-replicas` setting. Setting `minReplicas: 1` incurs a small ongoing cost but prevents cold start issues.
- The `mcp_extension` system key is only relevant for Azure Functions MCP extension deployments. It is not required for Azure Container Apps.
