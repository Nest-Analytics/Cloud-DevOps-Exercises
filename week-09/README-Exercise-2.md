# Exercise 2 — Docker Image + Azure Key Vault | Security & IAM: Azure Key Vault

---

## Overview

This exercise builds directly on Week 9 (Kubernetes on AKS). You will:

1. Write the Dockerfile and build the production image
2. Add Azure Key Vault to your Terraform configuration
3. Grant the service principal the permission it needs to write secrets to the vault
4. Store the three sensitive secrets (`APP-PASSWORD`, `APP-USERNAME`, `API-KEY`) in Key Vault
5. Confirm everything is accessible and the RBAC roles are correctly assigned

The non-sensitive values (`APP_TITLE`, `VITE_APP_TITLE`, `PORT`) are handled as GitHub Variables in Exercise 3 — they do not need to be in Key Vault.

---

## What You Need

- Exercise 1 completed — Kubernetes Secrets working on minikube
- The AKS cluster and Terraform state from Week 8 exercises still in place
- Azure account with an active subscription, Azure CLI authenticated (`az login`)
- Terraform installed

---

## Part 1 — Dockerfile

Before you can build or push anything, you need the Dockerfile. Create it in the root of your project:

```dockerfile
FROM --platform=$BUILDPLATFORM node:24-bookworm-slim AS build

WORKDIR /app

COPY package.json package-lock.json ./

RUN npm ci --ignore-scripts

COPY . .

RUN npm run build

FROM node:24-bookworm-slim AS runtime

WORKDIR /app

ENV NODE_ENV=production

COPY package.json package-lock.json ./

RUN npm ci --omit=dev --ignore-scripts

COPY --from=build /app/dist ./dist
COPY --from=build /app/server.js ./server.js
COPY --from=build /app/logger.js ./logger.js

EXPOSE 3000

CMD ["npm", "start"]
```

This is a multi-stage build. The `build` stage compiles the app; the `runtime` stage produces a lean production image containing only what is needed to run it. The `--platform=$BUILDPLATFORM` flag means the build stage runs on whatever platform the builder is (your laptop), while the final image is for `linux/amd64` — which is what AKS nodes require. The pipeline enforces this explicitly with `platforms: linux/amd64`.

Confirm it builds locally before continuing:

```bash
# Always build with --platform linux/amd64.
# AKS nodes run Linux AMD64. On a Mac with Apple Silicon (M1/M2/M3),
# Docker builds ARM images by default — these will fail to start on AKS.
docker build --platform linux/amd64 -t tasklineapp:local .

docker images | grep tasklineapp
```

---

## Part 2 — Grant the Service Principal Permission to Write to Key Vault

This is a step that must be done **before** running Terraform. When Terraform runs the pipeline, it authenticates as your service principal. That service principal needs `Key Vault Secrets Officer` on the vault before it can create secrets.

However, Terraform also creates the vault — so there is a chicken-and-egg: you cannot assign a role on a vault that does not exist yet.

The solution: Terraform creates the vault and immediately assigns the role in the same apply. But for the very first apply, the service principal needs `Contributor` on the subscription (which it already has from Week 8) to create the vault, and then Terraform assigns itself `Key Vault Secrets Officer` via `data.azurerm_client_config.current.object_id`. This works because the service principal is the one running Terraform, so it can assign roles to itself on resources it just created.

If you see a `(Forbidden)` error on the secrets after `terraform apply`, run apply a second time — the role assignment from the first apply propagates within about 60 seconds and the secrets will succeed on the retry.

---

## Part 3 — Terraform Files

### `terraform/variables.tf`

```hcl
variable "resource_group_name" {
  description = "Azure resource group name"
  type        = string
  default     = "learn-rg"
}

variable "location" {
  description = "Azure region"
  type        = string
  default     = "westeurope"
}

variable "aks_cluster_name" {
  description = "AKS cluster name"
  type        = string
  default     = "aks-taskline-learn"
}

variable "node_count" {
  description = "Number of AKS nodes"
  type        = number
  default     = 2
}

variable "acr_name" {
  description = "Azure Container Registry name (leave empty if not using ACR)"
  type        = string
  default     = ""
}

variable "key_vault_name" {
  description = "Azure Key Vault name (must be globally unique across Azure)"
  type        = string
  default     = "kv-tasklineapp"
}

variable "app_password" {
  description = "User password for Taskline"
  type        = string
  sensitive   = true
}

variable "app_username" {
  description = "User username for Taskline"
  type        = string
  sensitive   = true
}

variable "api_key" {
  description = "API key for Taskline"
  type        = string
  sensitive   = true
}
```

### `terraform/main.tf`

The backend block specifies where Terraform state is stored. The actual connection values are **not** hardcoded here — they are passed as `-backend-config` flags at `terraform init` time (from GitHub Secrets in the pipeline, from your shell locally). This means the same `main.tf` works for every team member and every environment without any edits.

```hcl
terraform {
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 4.0"
    }
  }

  # Backend block is intentionally empty here.
  # Connection details are passed via -backend-config flags at terraform init.
  # See Part 4 (local) and the pipeline workflow (Exercise 3) for how this works.
  backend "azurerm" {}
}

provider "azurerm" {
  features {
    resource_group {
      prevent_deletion_if_contains_resources = false
    }
  }
}

data "azurerm_client_config" "current" {}

resource "azurerm_resource_group" "main" {
  name     = var.resource_group_name
  location = var.location
}

resource "azurerm_log_analytics_workspace" "main" {
  name                = "law-tasklineapp"
  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name
  sku                 = "PerGB2018"
  retention_in_days   = 30
}

resource "azurerm_container_registry" "main" {
  count               = var.acr_name != "" ? 1 : 0
  name                = var.acr_name
  resource_group_name = azurerm_resource_group.main.name
  location            = azurerm_resource_group.main.location
  sku                 = "Basic"
  admin_enabled       = false
}

resource "azurerm_kubernetes_cluster" "main" {
  name                = var.aks_cluster_name
  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name
  dns_prefix          = var.aks_cluster_name

  default_node_pool {
    name       = "default"
    node_count = var.node_count
    vm_size    = "standard_b2ls_v2"
  }

  identity {
    type = "SystemAssigned"
  }

  oms_agent {
    log_analytics_workspace_id      = azurerm_log_analytics_workspace.main.id
    msi_auth_for_monitoring_enabled = true
  }
}

resource "azurerm_role_assignment" "aks_acr_pull" {
  count                            = var.acr_name != "" ? 1 : 0
  principal_id                     = azurerm_kubernetes_cluster.main.kubelet_identity[0].object_id
  role_definition_name             = "AcrPull"
  scope                            = azurerm_container_registry.main[0].id
  skip_service_principal_aad_check = true
}

# ── Key Vault ─────────────────────────────────────────────────────────────────

resource "azurerm_key_vault" "main" {
  name                       = var.key_vault_name
  location                   = azurerm_resource_group.main.location
  resource_group_name        = azurerm_resource_group.main.name
  tenant_id                  = data.azurerm_client_config.current.tenant_id
  sku_name                   = "standard"
  soft_delete_retention_days = 7
  purge_protection_enabled   = false
  rbac_authorization_enabled = true
}

# The service principal running Terraform needs Secrets Officer on the vault
# so it can create and update the secrets below.
# data.azurerm_client_config.current.object_id is the object ID of whichever
# identity is authenticated — your service principal in CI, your user account locally.
resource "azurerm_role_assignment" "kv_sp_secrets_officer" {
  scope                = azurerm_key_vault.main.id
  role_definition_name = "Key Vault Secrets Officer"
  principal_id         = data.azurerm_client_config.current.object_id
}

# AKS kubelet managed identity gets read-only access — least privilege
resource "azurerm_role_assignment" "kv_aks_secrets_user" {
  scope                = azurerm_key_vault.main.id
  role_definition_name = "Key Vault Secrets User"
  principal_id         = azurerm_kubernetes_cluster.main.kubelet_identity[0].object_id
}

# Secrets — values come from Terraform variables, never hardcoded
resource "azurerm_key_vault_secret" "app_password" {
  name         = "APP-PASSWORD"
  value        = var.app_password
  key_vault_id = azurerm_key_vault.main.id
  depends_on   = [azurerm_role_assignment.kv_sp_secrets_officer]
}

resource "azurerm_key_vault_secret" "app_username" {
  name         = "APP-USERNAME"
  value        = var.app_username
  key_vault_id = azurerm_key_vault.main.id
  depends_on   = [azurerm_role_assignment.kv_sp_secrets_officer]
}

resource "azurerm_key_vault_secret" "api_key" {
  name         = "API-KEY"
  value        = var.api_key
  key_vault_id = azurerm_key_vault.main.id
  depends_on   = [azurerm_role_assignment.kv_sp_secrets_officer]
}
```

### `terraform/outputs.tf`

```hcl
output "aks_cluster_name" {
  description = "Name of the AKS cluster"
  value       = azurerm_kubernetes_cluster.main.name
}

output "resource_group_name" {
  description = "Resource group containing the AKS cluster"
  value       = azurerm_resource_group.main.name
}

output "acr_login_server" {
  description = "ACR login server (N/A if not using ACR)"
  value       = var.acr_name != "" ? azurerm_container_registry.main[0].login_server : "N/A"
}
```

---

## Part 4 — GitHub Secrets and Variables

Go to your repository → Settings → Secrets and variables → Actions.

**GitHub Secrets** (sensitive — always masked in logs):

| Secret name | What it holds |
|---|---|
| `AZURE_CLIENT_ID` | Service principal client ID |
| `AZURE_CLIENT_SECRET` | Service principal client secret |
| `AZURE_SUBSCRIPTION_ID` | Your Azure subscription ID |
| `AZURE_TENANT_ID` | Your Azure tenant ID |
| `APP_USERNAME` | Your chosen username value |
| `APP_PASSWORD` | Your chosen password value |
| `API` | Your chosen API key value |
| `GHCR_TOKEN` | GitHub Personal Access Token with `read:packages` and `write:packages` scope |
| `TF_STATE_RESOURCE_GROUP` | Resource group containing the Terraform state storage account |
| `TF_STATE_STORAGE_ACCOUNT` | Storage account name for Terraform remote state |

**GitHub Variables** (non-sensitive — visible in logs, that is fine):

| Variable name | What it holds |
|---|---|
| `APP_TITLE` | `Taskline` |
| `VITE_APP_TITLE` | `Taskline` |
| `PORT` | `3000` |

> `APP_USERNAME`, `APP_PASSWORD`, and `API` are passed to Terraform as `TF_VAR_app_username`, `TF_VAR_app_password`, and `TF_VAR_api_key` in the pipeline's Terraform job — Terraform picks them up automatically and stores them in Key Vault.

---

## Part 5 — Run Terraform Locally to Verify

Test locally before the pipeline. The `-backend-config` flags supply the storage account details that the empty `backend "azurerm" {}` block expects:

```bash
cd terraform

export ARM_CLIENT_ID=YOUR_CLIENT_ID
export ARM_CLIENT_SECRET=YOUR_CLIENT_SECRET
export ARM_SUBSCRIPTION_ID=YOUR_SUBSCRIPTION_ID
export ARM_TENANT_ID=YOUR_TENANT_ID
export TF_VAR_app_password=<your-password>
export TF_VAR_app_username=<your-username>
export TF_VAR_api_key=<your-api-key>

terraform init \
  -backend-config="resource_group_name=YOUR_TFSTATE_RG" \
  -backend-config="storage_account_name=YOUR_TFSTATE_STORAGE" \
  -backend-config="container_name=tfstate1" \
  -backend-config="key=tasklineapp.terraform.tfstate"

terraform plan
terraform apply
```

If `apply` fails with a Key Vault permissions error on the secrets, wait 60 seconds and run `terraform apply` again — the `kv_sp_secrets_officer` role assignment needs a moment to propagate before the secret resources can use it.

---

## Part 6 — Verify in the CLI and Portal

```bash
# List secrets (names only — values not shown)
az keyvault secret list --vault-name kv-tasklineapp -o table

# Read a value to confirm it stored correctly
az keyvault secret show \
  --vault-name kv-tasklineapp \
  --name APP-PASSWORD \
  --query value -o tsv
```

In the Portal: Key Vault → **Secrets** — confirm `APP-PASSWORD`, `APP-USERNAME`, and `API-KEY` are listed and enabled. Key Vault → **Access control (IAM)** — confirm the role assignments: your service principal has `Key Vault Secrets Officer`, the AKS kubelet identity has `Key Vault Secrets User`.

---

## Submitting Your Work

See `SUBMISSION.md` for what to include and how to submit.

---

## Hints

- Key Vault names must be globally unique across all of Azure. If `kv-tasklineapp` is taken, add your initials: `kv-tasklineapp-san`.
- The empty `backend "azurerm" {}` block is correct — do not add values inside it. The `-backend-config` flags at `terraform init` supply everything. If you hardcode values in the block and push to GitHub, Terraform will try to use those values in CI instead of the secrets, causing failures.
- If `terraform apply` errors on the secrets with `(Forbidden)`, run it again after 60 seconds. RBAC propagation in Azure is eventually consistent — the role exists immediately but takes a short time to be enforced.
- `sensitive = true` on variables prevents Terraform from printing values in plan output or logs. Always mark credential variables as sensitive.
- The Dockerfile must be in the repository root before the pipeline can build the image in Exercise 3.
