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
    │   └── Application Insights
    ├── Key Vault
    ├── Storage Account
    ├── Container Apps Environment
    │   ├── Container App
    │   │   └── Managed Identity
    │   └── Container Job
    │       └── Managed Identity
    └── (other resources)
```

---

## CLI SETUP

```bash
# Install Azure CLI (macOS)
brew install azure-cli

# Login
az login

# Add Container Apps extension
az extension add --name containerapp --upgrade
```

The core `az` CLI ships with only a minimal built-in command set. Specialized services like Container Apps live in separately-installed extensions.

---

## SUBSCRIPTION

Top of the hierarchy. Subscriptions are provisioned outside the CLI (portal, enterprise agreement, etc.)

```bash
# Show active subscription
az account show

# List all subscriptions
az account list --output table

# Set active subscription
az account set --subscription "<subscription-id>"
```

---

## RESOURCE GROUP

A logical container that holds related resources (ACR, Container Apps, storage, etc.) for a project or environment. Resources share a lifecycle with their group: deleting the group deletes everything inside it.

```bash
# Create a resource group
az group create \
  --name <resource-group> \
  --location eastus

# List all resource groups
az group list --output table

# Delete a resource group (and everything inside it)
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

# List all registries (in current subscription, or scope with --resource-group)
az acr list --output table

# Delete the whole registry
az acr delete \
  --resource-group <resource-group> \
  --name <registry-name> \
  --yes
```

### IMAGES & TAGS

```bash
# List images in registry
az acr repository list --name <registry-name> --output table

# List tags for an image
az acr repository show-tags \
  --name <registry-name> \
  --repository <image-name> \
  --output table

# Delete a single image tag
az acr repository delete \
  --name <registry-name> \
  --image <image-name>:<tag> \
  --yes
```

---

## LOG ANALYTICS WORKSPACE

Central store for logs and metrics: required by the Container Apps Environment, and the backend Application Insights writes to.

```bash
# Create workspace
az monitor log-analytics workspace create \
  --resource-group <resource-group> \
  --workspace-name <workspace-name>

# List all workspaces
az monitor log-analytics workspace list \
  --resource-group <resource-group> \
  --output table

# Delete a workspace
az monitor log-analytics workspace delete \
  --resource-group <resource-group> \
  --workspace-name <workspace-name> \
  --yes
```

### GET CREDENTIALS (NEEDED FOR CONTAINER APPS ENVIRONMENT)

```bash
LOG_ANALYTICS_WORKSPACE_ID=$(az monitor log-analytics workspace show \
  --resource-group <resource-group> \
  --workspace-name <workspace-name> \
  --query customerId --output tsv)

LOG_ANALYTICS_WORKSPACE_KEY=$(az monitor log-analytics workspace get-shared-keys \
  --resource-group <resource-group> \
  --workspace-name <workspace-name> \
  --query primarySharedKey --output tsv)
```

---

## AZURE KEY VAULT

Managed store for secrets, keys, and certificates: keeps credentials out of code, config files, and environment variables.

```bash
# Create a vault (RBAC authorization model, recommended)
az keyvault create \
  --name <vault-name> \
  --resource-group <resource-group> \
  --location eastus \
  --enable-rbac-authorization true

# List all vaults
az keyvault list --output table

# Delete a vault (soft-deleted, recoverable for retention period)
az keyvault delete --name <vault-name>
```

---

## STORAGE ACCOUNT

Blob/file/queue/table storage: used for things like Container Apps file-share mounts or Container Job outputs.

```bash
# Create a storage account
az storage account create \
  --name <account-name> \
  --resource-group <resource-group> \
  --location eastus \
  --sku Standard_LRS

# List all storage accounts
az storage account list --output table

# Delete a storage account
az storage account delete \
  --name <account-name> \
  --resource-group <resource-group> \
  --yes
```

---

## CONTAINER APPS ENVIRONMENT

Shared networking and logging boundary for your apps and jobs. Requires a Log Analytics Workspace (see [[#LOG ANALYTICS WORKSPACE]] for getting `$LOG_ANALYTICS_WORKSPACE_ID` / `$LOG_ANALYTICS_WORKSPACE_KEY`).

```bash
# Create environment
az containerapp env create \
  --name <environment-name> \
  --resource-group <resource-group> \
  --location eastus \
  --logs-workspace-id $LOG_ANALYTICS_WORKSPACE_ID \
  --logs-workspace-key $LOG_ANALYTICS_WORKSPACE_KEY

# List environments
az containerapp env list --resource-group <resource-group> --output table

# Delete an environment
az containerapp env delete \
  --name <environment-name> \
  --resource-group <resource-group> \
  --yes
```

---

## CONTAINER APPS

Long-running services: HTTP servers, APIs, workers.

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

One-off or scheduled tasks: batch processing, cron jobs, pipelines.

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

Avoid storing registry credentials. Grant the app/job identity pull access to ACR instead.

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

Same pattern applies to Container Jobs: replace `containerapp` with `containerapp job`.

---

## AZURE KEY VAULT — SECRETS & ACCESS

### SECRETS

```bash
# Set a secret (creates it, or adds a new version if it already exists)
az keyvault secret set \
  --vault-name <vault-name> \
  --name <secret-name> \
  --value <secret-value>

# Get the current value of a secret
az keyvault secret show \
  --vault-name <vault-name> \
  --name <secret-name> \
  --query value --output tsv

# List all secrets in a vault (names only, not values)
az keyvault secret list --vault-name <vault-name> --output table

# Delete a secret (soft-deleted, recoverable)
az keyvault secret delete --vault-name <vault-name> --name <secret-name>
```

### GRANTING ACCESS (RBAC)

```bash
VAULT_ID=$(az keyvault show --name <vault-name> --query id --output tsv)

# Grant read access to secrets
az role assignment create \
  --assignee <principal-id-or-upn> \
  --role "Key Vault Secrets User" \
  --scope $VAULT_ID
```

### USING WITH MANAGED IDENTITY

Same pattern as ACR above: grant the Container App's identity access to the vault instead of embedding a connection string.

```bash
PRINCIPAL_ID=$(az containerapp show \
  --name <app-name> \
  --resource-group <resource-group> \
  --query identity.principalId --output tsv)

az role assignment create \
  --assignee $PRINCIPAL_ID \
  --role "Key Vault Secrets User" \
  --scope $VAULT_ID

# Reference a secret directly as a Container App env var
az containerapp secret set \
  --name <app-name> \
  --resource-group <resource-group> \
  --secrets my-secret=keyvaultref:https://<vault-name>.vault.azure.net/secrets/<secret-name>,identityref:system
```
