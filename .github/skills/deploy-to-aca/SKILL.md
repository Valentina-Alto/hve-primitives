---
name: deploy-to-aca
description: "Generate all scripts and infrastructure-as-code to deploy an application to Azure Container Apps. USE WHEN: containerizing an app, creating Dockerfiles, scaffolding Bicep/IaC for Azure Container Apps, setting up ACR, or deploying with azd. Ensures secure defaults, Managed Identity, and repeatable deployments."
argument-hint: "Point to the application root or describe the app to deploy"
---

# Deploy to Azure Container Apps

## When to Use

- User asks to deploy an application to Azure Container Apps
- User asks to containerize an existing app and host it on Azure
- User asks to scaffold infrastructure-as-code for Container Apps
- User asks to set up a CI/CD pipeline targeting Container Apps
- User asks to create a Dockerfile and deployment scripts for their project

## Architecture Pattern

Every deployment MUST produce this structure alongside the existing application code:

```
project_root/
├── Dockerfile                  # Multi-stage build for the application
├── .dockerignore               # Excludes dev artifacts from the image
├── infra/
│   ├── main.bicep              # Orchestrator — wires all modules together
│   ├── main.parameters.json    # Environment-specific parameter values
│   ├── modules/
│   │   ├── container-app.bicep       # Container App + ingress + scaling
│   │   ├── container-registry.bicep  # Azure Container Registry
│   │   ├── container-app-env.bicep   # Container Apps Environment + Log Analytics
│   │   └── managed-identity.bicep    # User-assigned Managed Identity + role assignments
│   └── abbreviations.json     # Naming convention abbreviations (optional)
├── azure.yaml                  # azd project definition (if using azd)
└── scripts/
    └── deploy.ps1              # One-command deployment script (optional)
```

## Key Principles

1. **Infrastructure as Code first** — Always generate Bicep files under `infra/`. Never rely on ad-hoc `az` CLI commands for resource creation.
2. **Modular Bicep** — Each Azure resource type gets its own module file under `infra/modules/`. The `main.bicep` orchestrates them.
3. **Secure by default** — Use Managed Identity for ACR pull (no admin credentials). Disable anonymous pull on ACR. Use HTTPS ingress. Never hardcode secrets.
4. **Repeatable deployments** — Prefer `azd up` when possible. All configuration is parameterized. No manual portal steps.
5. **Test locally first** — If Docker is available, validate the image locally before pushing to Azure.
6. **Port alignment** — The Container App `targetPort` MUST match the port the application listens on.

## Step-by-Step Procedure

### Step 1 — Analyze the Application

Read the project to determine:

| What to find | Where to look | Why it matters |
|--------------|--------------|----------------|
| Language / runtime | `package.json`, `requirements.txt`, `*.csproj`, `go.mod` | Determines Dockerfile base image |
| Entry point | `app.py`, `server.js`, `Program.cs`, `main.go` | Determines `CMD` in Dockerfile |
| Listening port | Source code (`app.listen`, `EXPOSE`, `--port`) | Must match Container App `targetPort` |
| Environment variables | `.env`, `appsettings.json`, config files | Become Container App env vars or secrets |
| Dependencies | Lock files, `requirements.txt`, `package-lock.json` | Copied into Docker build stage |
| Static assets | `static/`, `public/`, `wwwroot/` | May need a build step or volume |

**Decision point:** If the project already has a `Dockerfile`, inspect it and adapt rather than overwriting. If it has `azure.yaml`, this may already be an azd project — confirm before regenerating.

### Step 2 — Create the Dockerfile

Generate a multi-stage Dockerfile appropriate for the detected runtime:

**Python (Flask/FastAPI):**
```dockerfile
FROM python:3.12-slim AS base
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
EXPOSE 8000
CMD ["gunicorn", "--bind", "0.0.0.0:8000", "app:app"]
```

**Node.js:**
```dockerfile
FROM node:20-alpine AS build
WORKDIR /app
COPY package*.json ./
RUN npm ci --production
COPY . .
EXPOSE 3000
CMD ["node", "server.js"]
```

**ASP.NET:**
```dockerfile
FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
WORKDIR /src
COPY *.csproj ./
RUN dotnet restore
COPY . .
RUN dotnet publish -c Release -o /app/publish

FROM mcr.microsoft.com/dotnet/aspnet:8.0
WORKDIR /app
COPY --from=build /app/publish .
EXPOSE 8080
ENTRYPOINT ["dotnet", "<ProjectName>.dll"]
```

**Critical rules for Dockerfiles:**
- Always use a specific version tag for the base image — never `latest`
- Use `--no-cache-dir` (pip) or `npm ci --production` to minimize image size
- Copy dependency files first, then source — leverages Docker layer caching
- The `EXPOSE` port MUST match the app's actual listening port

Also create a `.dockerignore`:
```
.git
.github
.env
__pycache__
node_modules
*.pyc
.vscode
infra/
scripts/
```

### Step 3 — Scaffold Bicep Infrastructure

Create `infra/main.bicep` as the orchestrator:

```bicep
targetScope = 'resourceGroup'

@description('Primary location for all resources')
param location string = resourceGroup().location

@description('Base name for all resources')
param appName string

@description('Container image to deploy (e.g. myregistry.azurecr.io/myapp:latest)')
param containerImage string

@description('Port the container listens on')
param targetPort int = 8000

// ── Managed Identity ──
module identity 'modules/managed-identity.bicep' = {
  name: 'identity'
  params: {
    location: location
    appName: appName
  }
}

// ── Container Registry ──
module acr 'modules/container-registry.bicep' = {
  name: 'acr'
  params: {
    location: location
    appName: appName
    principalId: identity.outputs.principalId
  }
}

// ── Container Apps Environment ──
module env 'modules/container-app-env.bicep' = {
  name: 'env'
  params: {
    location: location
    appName: appName
  }
}

// ── Container App ──
module app 'modules/container-app.bicep' = {
  name: 'app'
  params: {
    location: location
    appName: appName
    containerImage: containerImage
    targetPort: targetPort
    environmentId: env.outputs.environmentId
    userAssignedIdentityId: identity.outputs.identityId
    registryServer: acr.outputs.loginServer
  }
}

output appUrl string = app.outputs.fqdn
output acrLoginServer string = acr.outputs.loginServer
```

Create each module under `infra/modules/`:

**`managed-identity.bicep`** — User-assigned Managed Identity:
```bicep
param location string
param appName string

resource identity 'Microsoft.ManagedIdentity/userAssignedIdentities@2023-01-31' = {
  name: 'id-${appName}'
  location: location
}

output identityId string = identity.id
output principalId string = identity.properties.principalId
output clientId string = identity.properties.clientId
```

**`container-registry.bicep`** — ACR with AcrPull role for the identity:
```bicep
param location string
param appName string
param principalId string

resource acr 'Microsoft.ContainerRegistry/registries@2023-11-01-preview' = {
  name: replace('acr${appName}', '-', '')
  location: location
  sku: { name: 'Basic' }
  properties: {
    adminUserEnabled: false
    anonymousPullEnabled: false
  }
}

// Grant AcrPull to the managed identity
resource acrPull 'Microsoft.Authorization/roleAssignments@2022-04-01' = {
  name: guid(acr.id, principalId, 'acrpull')
  scope: acr
  properties: {
    roleDefinitionId: subscriptionResourceId('Microsoft.Authorization/roleDefinitions', '7f951dda-4ed3-4680-a7ca-43fe172d538d')
    principalId: principalId
    principalType: 'ServicePrincipal'
  }
}

output loginServer string = acr.properties.loginServer
output acrName string = acr.name
```

**`container-app-env.bicep`** — Environment + Log Analytics:
```bicep
param location string
param appName string

resource logAnalytics 'Microsoft.OperationalInsights/workspaces@2023-09-01' = {
  name: 'log-${appName}'
  location: location
  properties: {
    sku: { name: 'PerGB2018' }
    retentionInDays: 30
  }
}

resource env 'Microsoft.App/managedEnvironments@2024-03-01' = {
  name: 'cae-${appName}'
  location: location
  properties: {
    appLogsConfiguration: {
      destination: 'log-analytics'
      logAnalyticsConfiguration: {
        customerId: logAnalytics.properties.customerId
        sharedKey: logAnalytics.listKeys().primarySharedKey
      }
    }
  }
}

output environmentId string = env.id
```

**`container-app.bicep`** — The Container App itself:
```bicep
param location string
param appName string
param containerImage string
param targetPort int
param environmentId string
param userAssignedIdentityId string
param registryServer string

resource app 'Microsoft.App/containerApps@2024-03-01' = {
  name: 'ca-${appName}'
  location: location
  identity: {
    type: 'UserAssigned'
    userAssignedIdentities: {
      '${userAssignedIdentityId}': {}
    }
  }
  properties: {
    managedEnvironmentId: environmentId
    configuration: {
      ingress: {
        external: true
        targetPort: targetPort
        transport: 'http'
        allowInsecure: false
      }
      registries: [
        {
          server: registryServer
          identity: userAssignedIdentityId
        }
      ]
    }
    template: {
      containers: [
        {
          name: appName
          image: containerImage
          resources: {
            cpu: json('0.5')
            memory: '1.0Gi'
          }
        }
      ]
      scale: {
        minReplicas: 0
        maxReplicas: 3
        rules: [
          {
            name: 'http-scaling'
            http: {
              metadata: {
                concurrentRequests: '50'
              }
            }
          }
        ]
      }
    }
  }
}

output fqdn string = app.properties.configuration.ingress.fqdn
```

**Critical rules for Bicep:**
- Always use the latest stable API version for each resource type
- ACR: `adminUserEnabled: false`, `anonymousPullEnabled: false`
- Use User-Assigned Managed Identity + AcrPull role — never ACR admin credentials
- Ingress: `allowInsecure: false`
- All names parameterized, no hardcoded values
- Place everything under `infra/` folder

### Step 4 — Create Parameters File

Create `infra/main.parameters.json`:

```json
{
  "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentParameters.json#",
  "contentVersion": "1.0.0.0",
  "parameters": {
    "appName": {
      "value": "<APP_NAME>"
    },
    "containerImage": {
      "value": "<ACR_LOGIN_SERVER>/<APP_NAME>:latest"
    },
    "targetPort": {
      "value": 8000
    }
  }
}
```

Replace `<APP_NAME>` with the actual application name (lowercase, alphanumeric + hyphens). Replace `<ACR_LOGIN_SERVER>` after ACR is created, or use the `azd` flow which handles this automatically.

### Step 5 — Create azd Project Definition (recommended)

If using Azure Developer CLI, create `azure.yaml` at the project root:

```yaml
name: <app-name>
services:
  app:
    host: containerapp
    project: .
    docker:
      path: Dockerfile
```

This enables `azd up` for a single-command deploy flow.

**Decision point:** If the user explicitly prefers raw `az` CLI or has an existing CI/CD pipeline, skip `azure.yaml` and proceed to the deployment script in Step 6.

### Step 6 — Create Deployment Script (if not using azd)

Create `scripts/deploy.ps1` for a manual deployment path:

```powershell
# deploy.ps1 — Deploy to Azure Container Apps
# Usage: ./scripts/deploy.ps1 -AppName myapp -ResourceGroup rg-myapp -Location eastus

param(
    [Parameter(Mandatory)] [string] $AppName,
    [Parameter(Mandatory)] [string] $ResourceGroup,
    [string] $Location = "eastus"
)

$ErrorActionPreference = "Stop"

# 1. Create resource group
Write-Host "Creating resource group '$ResourceGroup'..."
az group create --name $ResourceGroup --location $Location --output none

# 2. Deploy infrastructure
Write-Host "Deploying infrastructure (Bicep)..."
$deployment = az deployment group create `
    --resource-group $ResourceGroup `
    --template-file infra/main.bicep `
    --parameters infra/main.parameters.json `
    --parameters appName=$AppName `
    --query "properties.outputs" -o json | ConvertFrom-Json

$acrLoginServer = $deployment.acrLoginServer.value
$appUrl = $deployment.appUrl.value

# 3. Build and push container image
Write-Host "Building and pushing image to $acrLoginServer..."
az acr build --registry ($acrLoginServer -replace '\.azurecr\.io','') `
    --image "${AppName}:latest" .

# 4. Update Container App with the built image
Write-Host "Updating Container App with new image..."
az containerapp update `
    --name "ca-$AppName" `
    --resource-group $ResourceGroup `
    --image "${acrLoginServer}/${AppName}:latest"

Write-Host ""
Write-Host "Deployed successfully!" -ForegroundColor Green
Write-Host "App URL: https://$appUrl"
Write-Host "Portal:  https://portal.azure.com/#@/resource/subscriptions/.../resourceGroups/$ResourceGroup"
```

### Step 7 — Local Validation

Before deploying to Azure, validate locally if Docker is available:

```powershell
# Build
docker build -t <app-name>:local .

# Run (use the same port the app listens on)
docker run --rm -p 8000:8000 --env-file .env <app-name>:local

# Test
curl http://localhost:8000/
```

If the local test fails, fix the Dockerfile before proceeding to Azure deployment.

### Step 8 — Deploy

**If using azd (recommended):**
```powershell
# Preview first
azd provision --preview

# Deploy
azd up
```

**If using the deployment script:**
```powershell
./scripts/deploy.ps1 -AppName myapp -ResourceGroup rg-myapp -Location eastus
```

**If using raw Bicep:**
```powershell
# Validate first
az deployment group what-if `
    --resource-group rg-myapp `
    --template-file infra/main.bicep `
    --parameters infra/main.parameters.json

# Deploy
az deployment group create `
    --resource-group rg-myapp `
    --template-file infra/main.bicep `
    --parameters infra/main.parameters.json
```

After deployment, always test the returned URL and provide it to the user.

## Checklist

Before finishing, verify:
- [ ] `Dockerfile` uses a specific base image tag (not `latest`)
- [ ] `Dockerfile` EXPOSE port matches the app's actual listening port
- [ ] `.dockerignore` excludes `.env`, `.git`, `node_modules`, `__pycache__`, `infra/`
- [ ] All Bicep files are under `infra/` with modules in `infra/modules/`
- [ ] ACR has `adminUserEnabled: false` and `anonymousPullEnabled: false`
- [ ] Authentication uses User-Assigned Managed Identity + AcrPull role
- [ ] Container App `targetPort` matches Dockerfile EXPOSE port
- [ ] Ingress has `allowInsecure: false`
- [ ] No secrets or credentials are hardcoded anywhere
- [ ] Parameters file has no real values committed (use placeholders or `azd` env)
- [ ] Local Docker build + run was tested (if Docker available)
- [ ] Deployment was validated with `--preview` or `what-if` before applying

## Common Mistakes to Avoid

| Mistake | Fix |
|---------|-----|
| Port mismatch between app, Dockerfile, and Container App | Trace the port from source code → EXPOSE → `targetPort` — all three must match |
| Using ACR admin credentials | Use Managed Identity + AcrPull role assignment |
| Hardcoded ACR login server in Bicep | Pass as parameter or use Bicep module output |
| Missing `.dockerignore` → huge image with `.git/` | Always create `.dockerignore` excluding dev artifacts |
| Using `latest` base image tag | Pin to specific version: `python:3.12-slim`, `node:20-alpine` |
| Deploying without `what-if` / `--preview` | Always validate before applying infrastructure changes |
| Secrets in `main.parameters.json` committed to git | Use `azd` env vars, Key Vault references, or `.gitignore` the params file |
| `allowInsecure: true` on ingress | Always set to `false` — enforce HTTPS |
