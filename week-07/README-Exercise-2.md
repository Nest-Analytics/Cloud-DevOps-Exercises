# Exercise 2 — Deploy the Taskline App to Azure with Terraform and GitHub Actions | Docker + Terraform + YAML (CI/CD)

---

## Overview

This exercise brings together everything from the last three weeks. You will:

1. Build and push your Docker image to Docker Hub — built for the correct platform
2. Write Terraform to provision Azure infrastructure (resource group, Log Analytics, Container Apps environment, Container App)
3. Write a GitHub Actions workflow that runs on every push — building the image, pushing it to Docker Hub, and applying your Terraform to deploy it to Azure

When it is working, pushing a code change to your `main` branch should automatically result in a live, updated app at a public Azure URL.

---

## ⚠️ Platform Note — Read This First

Azure Container Apps runs on **Linux AMD64** infrastructure. If you are on a Mac with Apple Silicon (M1, M2, M3), Docker builds ARM images by default — these will fail to start on Azure.

**Always build with `--platform linux/amd64`** when building an image intended for Azure:

```bash
docker build --platform linux/amd64 -t your-dockerhub-username/taskline:v1 .
docker push your-dockerhub-username/taskline:v1
```

The GitHub Actions pipeline handles this automatically (the `ubuntu-latest` runner is AMD64), but for any manual builds from your local machine, the `--platform` flag is required.

---

## What You Need

- Exercise 1 completed (working Dockerfile in the repo)
- A Docker Hub account — [hub.docker.com](https://hub.docker.com)
- An Azure account with an active subscription
- A GitHub repository for the taskline app
- The following secrets configured in your GitHub repository settings (Settings → Secrets and variables → Actions):

| Secret name | What it contains |
|---|---|
| `DOCKERHUB_USERNAME` | Your Docker Hub username |
| `DOCKERHUB_TOKEN` | A Docker Hub access token (not your password — generate one at hub.docker.com → Account Settings → Security) |
| `AZURE_CLIENT_ID` | Azure service principal client ID |
| `AZURE_CLIENT_SECRET` | Azure service principal client secret |
| `AZURE_SUBSCRIPTION_ID` | Your Azure subscription ID |
| `AZURE_TENANT_ID` | Your Azure tenant ID |
| `API_KEY` | API key used by the taskline app |
| `DB_PASSWORD` | Database password used by the taskline app |

> **How to create an Azure service principal:**
> ```bash
> az ad sp create-for-rbac \
>   --name "sp-taskline-deploy" \
>   --role Contributor \
>   --scopes /subscriptions/YOUR_SUBSCRIPTION_ID \
>   --sdk-auth
> ```
> The output JSON gives you `clientId`, `clientSecret`, `subscriptionId`, and `tenantId`. Add each to GitHub Secrets.

---

## Repository Structure

Your repository should look like this when complete:

```
taskline-app/
├── src/                          ← taskline app source (provided)
├── package.json
├── Dockerfile                    ← from Exercise 1
├── .dockerignore                 ← from Exercise 1
├── terraform/
│   ├── main.tf
│   ├── variables.tf
│   └── outputs.tf
└── .github/
    └── workflows/
        └── deploy.yml
```

---

## Part 1 — Terraform

Create the `terraform/` folder in your repository and write the following files.

### `terraform/variables.tf`

```hcl
variable "resource_group_name" {
  description = "The name of the resource group"
  type        = string
  default     = "rg-taskline"
}

variable "location" {
  description = "The Azure region to deploy the resources to"
  type        = string
  default     = "westeurope"
}

variable "docker_image" {
  description = "Docker image to deploy, e.g. hellosanmi/taskline:v1"
  type        = string
}

variable "api_key" {
  description = "The API key to be used by the application"
  type        = string
  sensitive   = true
}

variable "db_password" {
  description = "The database password"
  type        = string
  sensitive   = true
}
```

---

### `terraform/main.tf`

```hcl
# Define required providers and their versions
terraform {
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 3.0"
    }
  }
}

# Configure the Azure provider
provider "azurerm" {
  features {
    resource_group {
      prevent_deletion_if_contains_resources = false
    }
  }
}

# Create a Resource Group to hold all resources
resource "azurerm_resource_group" "main" {
  name     = var.resource_group_name
  location = var.location
}

# Create a Log Analytics Workspace for storing container logs
resource "azurerm_log_analytics_workspace" "main" {
  name                = "analytics-taskline"
  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name
  sku                 = "PerGB2018"
  retention_in_days   = 30
}

# Create the Container App Environment (the underlying managed infrastructure)
resource "azurerm_container_app_environment" "main" {
  name                       = "env-taskline"
  location                   = azurerm_resource_group.main.location
  resource_group_name        = azurerm_resource_group.main.name
  log_analytics_workspace_id = azurerm_log_analytics_workspace.main.id
}

# Deploy the actual Container App with its container image and ingress settings
resource "azurerm_container_app" "main" {
  name                         = "taskline"
  container_app_environment_id = azurerm_container_app_environment.main.id
  resource_group_name          = azurerm_resource_group.main.name
  revision_mode                = "Single"

  # Define sensitive values here
  secret {
    name  = "api-key"
    value = var.api_key
  }

  secret {
    name  = "db-password"
    value = var.db_password
  }

  template {
    container {
      name   = "taskline"
      image  = var.docker_image
      cpu    = 0.25
      memory = "0.5Gi"

      # App environment variables
      env {
        name  = "APP_TITLE"
        value = "Taskline"
      }

      env {
        name  = "VITE_APP_TITLE"
        value = "Taskline"
      }

      env {
        name  = "PORT"
        value = "3000"
      }

      # Reference secrets as environment variables
      env {
        name        = "API_KEY"
        secret_name = "api-key"
      }

      env {
        name        = "DB_PASSWORD"
        secret_name = "db-password"
      }
    }
  }

  ingress {
    external_enabled = true
    target_port      = 3000
    traffic_weight {
      percentage      = 100
      latest_revision = true
    }
  }
}
```

---

### `terraform/outputs.tf`

```hcl
output "app_url" {
  description = "Public URL of the deployed taskline app"
  value       = "https://${azurerm_container_app.main.ingress[0].fqdn}"
}
```

---

## Part 2 — GitHub Actions Workflow

Create `.github/workflows/deploy.yml`:

```yaml
name: Build, Push and Deploy

on:
  push:
    branches:
      - main

jobs:
  build-and-push:
    name: Build and Push Docker Image
    runs-on: ubuntu-latest

    outputs:
      image_tag: ${{ steps.meta.outputs.image_tag }}

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Log in to Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}

      - name: Generate image tag
        id: meta
        run: |
          IMAGE_TAG="${{ secrets.DOCKERHUB_USERNAME }}/taskline:${{ github.sha }}"
          echo "image_tag=${IMAGE_TAG}" >> $GITHUB_OUTPUT

      - name: Build and push image
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.image_tag }}
          platforms: linux/amd64

  deploy:
    name: Deploy to Azure with Terraform
    runs-on: ubuntu-latest
    needs: build-and-push

    env:
      ARM_CLIENT_ID: ${{ secrets.AZURE_CLIENT_ID }}
      ARM_CLIENT_SECRET: ${{ secrets.AZURE_CLIENT_SECRET }}
      ARM_SUBSCRIPTION_ID: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
      ARM_TENANT_ID: ${{ secrets.AZURE_TENANT_ID }}

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Set up Terraform
        uses: hashicorp/setup-terraform@v3

      - name: Terraform Init
        working-directory: terraform
        run: terraform init

      - name: Terraform Plan
        working-directory: terraform
        run: |
          terraform plan \
            -var="docker_image=${{ needs.build-and-push.outputs.image_tag }}" \
            -var="api_key=${{ secrets.API_KEY }}" \
            -var="db_password=${{ secrets.DB_PASSWORD }}" \
            -out=tfplan

      - name: Terraform Apply
        working-directory: terraform
        run: terraform apply -auto-approve tfplan

      - name: Print app URL
        working-directory: terraform
        run: terraform output app_url
```

> **Note:** The pipeline builds with `platforms: linux/amd64` — this ensures the image runs correctly on Azure regardless of what machine triggered the push.

---

## How It Works — End to End

When you push to `main`:

1. **build-and-push job** checks out your code, logs in to Docker Hub, builds the image for `linux/amd64` tagged with the git commit SHA, and pushes it
2. **deploy job** runs after — it authenticates to Azure using your service principal secrets, runs `terraform init` and `terraform plan` with the new image tag and app secrets passed as variables, then applies the plan
3. Terraform provisions (or updates) the resource group, Log Analytics workspace, Container Apps environment, and Container App pointing at the new image
4. The workflow prints the public URL as the final step

---

## Verifying It Worked

After the workflow completes:

1. Go to your GitHub repository → **Actions** tab
2. Click the latest workflow run and confirm both jobs show green ticks
3. In the **deploy** job logs, find the `Print app URL` step — copy the URL
4. Open the URL in your browser — the taskline app should be live

---

## Submitting Your Work

Push all of the following to your `main` branch:

```
├── Dockerfile
├── .dockerignore
├── terraform/
│   ├── main.tf
│   ├── variables.tf
│   └── outputs.tf
└── .github/
    └── workflows/
        └── deploy.yml
```

Also include in your repository root a file named `Acceptance-Exercise.md` with:

```markdown
## Exercise 2 Submission

- **GitHub repo URL:** https://github.com/YOUR_USERNAME/YOUR_REPO
- **Live app URL:** https://YOUR_APP.azurecontainerapps.io
- **Successful Actions run URL:** https://github.com/YOUR_USERNAME/YOUR_REPO/actions/runs/XXXXXXX
```

---

## Hints

- **On a Mac with Apple Silicon?** Always build locally with `--platform linux/amd64`. Skipping this flag means your image is ARM64 and will not start on Azure.
- If the **build-and-push** job fails at login, double-check that `DOCKERHUB_TOKEN` is an access token, not your account password. Generate one at Docker Hub → Account Settings → Security → New Access Token.
- If the **deploy** job fails at `terraform init`, make sure `working-directory: terraform` is set on every Terraform step, not just the first.
- If Terraform fails with an authentication error, check that all four `ARM_*` environment variables are set and match the service principal output exactly.
- If Terraform fails with a variable error for `api_key` or `db_password`, make sure those secrets are added to GitHub Secrets and the `-var` flags are present in the plan step.
- The first time the pipeline runs, Terraform creates all resources from scratch — this takes 2–3 minutes. Subsequent runs update only what has changed.
- `revision_mode = "Single"` means Terraform replaces the running revision when the image changes. This is correct for this exercise.
- Do not commit secrets into your repository. All credentials must be in GitHub Secrets, never in `.yml` or `.tf` files.
