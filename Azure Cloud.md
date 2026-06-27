---
tags: [azure, cloud, containers, devops, cli]
---
# Azure Cloud

## RESOURCE HIERARCHY

```
Subscription
└── Resource Group
    ├── Azure Container Registry (ACR)
    ├── Log Analytics Workspace
    ├── Container Apps Environment
    │   ├── Container App
    │   └── Container Job
    └── (other resources)
```

---

## CLI SETUP

```bash
# Install Azure CLI (macOS)
brew install azure-cli

# Login
az login

# Show active subscription
az account show

# List all subscriptions
az account list --output table

# Set active subscription
az account set --subscription "<subscription-id>"

# Add Container Apps extension
az extension add --name containerapp --upgrade
```

---

## RESOURCE GROUP

```bash
az group create \
  --name <resource-group> \
  --location eastus

az group list --output table
az group delete --name <resource-group> --yes
```

---

## AZURE CONTAINER REGISTRY (ACR)

Stores your Docker images.

```bash
# Create registry
az acr create \
  --resource-group <resource-group> \
  --name <registry-name> \
  --sku Basic

# Login to registry
az acr login --name <registry-name>

# Build and push image directly in ACR (no local Docker needed)
az acr build \
  --registry <registry-name> \
  --image <image-name>:<tag> \
  --file Dockerfile \
  .

# List images in registry
az acr repository list --name <registry-name> --output table

# List tags for an image
az acr repository show-tags \
  --name <registry-name> \
  --repository <image-name> \
  --output table
```

---

## CONTAINER APPS ENVIRONMENT

Shared networking and logging boundary for your apps and jobs.

```bash
# Create Log Analytics workspace (required)
az monitor log-analytics workspace create \
  --resource-group <resource-group> \
  --workspace-name <workspace-name>

# Get workspace credentials
LOG_ANALYTICS_WORKSPACE_ID=$(az monitor log-analytics workspace show \
  --resource-group <resource-group> \
  --workspace-name <workspace-name> \
  --query customerId --output tsv)

LOG_ANALYTICS_WORKSPACE_KEY=$(az monitor log-analytics workspace get-shared-keys \
  --resource-group <resource-group> \
  --workspace-name <workspace-name> \
  --query primarySharedKey --output tsv)

# Create environment
az containerapp env create \
  --name <environment-name> \
  --resource-group <resource-group> \
  --location eastus \
  --logs-workspace-id $LOG_ANALYTICS_WORKSPACE_ID \
  --logs-workspace-key $LOG_ANALYTICS_WORKSPACE_KEY

# List environments
az containerapp env list --resource-group <resource-group> --output table
```

---

## CONTAINER APPS

Long-running services — HTTP servers, APIs, workers.

### CREATE

```bash
az containerapp create \
  --name <app-name> \
  --resource-group <resource-group> \
  --environment <environment-name> \
  --image <registry-name>.azurecr.io/<image-name>:<tag> \
  --registry-server <registry-name>.azurecr.io \
  --target-port 8080 \
  --ingress external \
  --min-replicas 1 \
  --max-replicas 3 \
  --cpu 0.5 \
  --memory 1.0Gi \
  --env-vars KEY=value KEY2=secretref:my-secret
```

### UPDATE

```bash
# Deploy a new image
az containerapp update \
  --name <app-name> \
  --resource-group <resource-group> \
  --image <registry-name>.azurecr.io/<image-name>:<new-tag>

# Scale replicas
az containerapp update \
  --name <app-name> \
  --resource-group <resource-group> \
  --min-replicas 0 \
  --max-replicas 5
```

### INSPECT

```bash
# Show app details and FQDN
az containerapp show \
  --name <app-name> \
  --resource-group <resource-group>

# Get public URL
az containerapp show \
  --name <app-name> \
  --resource-group <resource-group> \
  --query properties.configuration.ingress.fqdn \
  --output tsv

# Stream live logs
az containerapp logs show \
  --name <app-name> \
  --resource-group <resource-group> \
  --follow

# List all apps
az containerapp list --resource-group <resource-group> --output table
```

### SECRETS

```bash
# Add a secret
az containerapp secret set \
  --name <app-name> \
  --resource-group <resource-group> \
  --secrets my-secret=<value>

# Reference secret as env var
az containerapp update \
  --name <app-name> \
  --resource-group <resource-group> \
  --set-env-vars MY_VAR=secretref:my-secret
```

---

## CONTAINER JOBS

One-off or scheduled tasks — batch processing, cron jobs, pipelines.

### CREATE (MANUAL TRIGGER)

```bash
az containerapp job create \
  --name <job-name> \
  --resource-group <resource-group> \
  --environment <environment-name> \
  --trigger-type Manual \
  --image <registry-name>.azurecr.io/<image-name>:<tag> \
  --registry-server <registry-name>.azurecr.io \
  --cpu 1.0 \
  --memory 2.0Gi \
  --replica-timeout 1800 \
  --env-vars KEY=value
```

### CREATE (SCHEDULED — CRON)

```bash
az containerapp job create \
  --name <job-name> \
  --resource-group <resource-group> \
  --environment <environment-name> \
  --trigger-type Schedule \
  --cron-expression "0 9 * * *" \
  --image <registry-name>.azurecr.io/<image-name>:<tag> \
  --registry-server <registry-name>.azurecr.io \
  --cpu 1.0 \
  --memory 2.0Gi \
  --replica-timeout 3600
```

### RUN AND INSPECT

```bash
# Trigger a manual job execution
az containerapp job start \
  --name <job-name> \
  --resource-group <resource-group>

# List executions
az containerapp job execution list \
  --name <job-name> \
  --resource-group <resource-group> \
  --output table

# Show a specific execution
az containerapp job execution show \
  --name <job-name> \
  --resource-group <resource-group> \
  --job-execution-name <execution-name>

# Stream logs for an execution
az containerapp job logs show \
  --name <job-name> \
  --resource-group <resource-group> \
  --execution <execution-name> \
  --follow

# Update job image
az containerapp job update \
  --name <job-name> \
  --resource-group <resource-group> \
  --image <registry-name>.azurecr.io/<image-name>:<new-tag>

# List all jobs
az containerapp job list --resource-group <resource-group> --output table
```

---

## MANAGED IDENTITY (RECOMMENDED FOR ACR AUTH)

Avoid storing registry credentials — grant the app/job identity pull access to ACR instead.

```bash
# Enable system-assigned identity on a Container App
az containerapp identity assign \
  --name <app-name> \
  --resource-group <resource-group> \
  --system-assigned

# Get the identity principal ID
PRINCIPAL_ID=$(az containerapp show \
  --name <app-name> \
  --resource-group <resource-group> \
  --query identity.principalId --output tsv)

# Get ACR resource ID
ACR_ID=$(az acr show \
  --name <registry-name> \
  --resource-group <resource-group> \
  --query id --output tsv)

# Grant AcrPull role to the identity
az role assignment create \
  --assignee $PRINCIPAL_ID \
  --role AcrPull \
  --scope $ACR_ID
```

Same pattern applies to Container Jobs — replace `containerapp` with `containerapp job`.
