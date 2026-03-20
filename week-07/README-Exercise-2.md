# Exercise 2 — Deploy the Game App to Azure with Terraform and GitHub Actions - Docker + Terraform + YAML (CI/CD)

---

## Overview

This exercise brings together everything from the last three weeks. You will:

1. Push your Docker image to Docker Hub
2. Write Terraform to provision Azure infrastructure (resource group, Container Apps environment, Container App)
3. Write a GitHub Actions workflow that runs on every push — building the image, pushing it to Docker Hub, and applying your Terraform to deploy it to Azure

When it is working, pushing a code change to your `main` branch should automatically result in a live, updated app at a public Azure URL.

---

## What You Need

- Exercise 1 completed (working Dockerfile in the repo)
- A Docker Hub account — [hub.docker.com](https://hub.docker.com)
- An Azure account with an active subscription
- A GitHub repository for the game app
- The following secrets configured in your GitHub repository settings (Settings → Secrets and variables → Actions):

| Secret name | What it contains |
|---|---|
| `DOCKERHUB_USERNAME` | Your Docker Hub username |
| `DOCKERHUB_TOKEN` | A Docker Hub access token (not your password — generate one at hub.docker.com → Account Settings → Security) |
| `AZURE_CLIENT_ID` | Azure service principal client ID |
| `AZURE_CLIENT_SECRET` | Azure service principal client secret |
| `AZURE_SUBSCRIPTION_ID` | Your Azure subscription ID |
| `AZURE_TENANT_ID` | Your Azure tenant ID |

> **How to create an Azure service principal:**
> ```bash
> az ad sp create-for-rbac \
>   --name "sp-gameapp-deploy" \
>   --role Contributor \
>   --scopes /subscriptions/YOUR_SUBSCRIPTION_ID \
>   --sdk-auth
> ```
> The output JSON gives you `clientId`, `clientSecret`, `subscriptionId`, and `tenantId`. Add each to GitHub Secrets.

---

## Repository Structure

Your repository should look like this when complete:

```
game-app/
├── src/                          ← game app source (provided)
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
  description = "Name of the Azure resource group"
  type        = string
  default     = "rg-gameapp"
}

variable "location" {
  description = "Azure region"
  type        = string
  default     = "EastUS"
}

variable "container_app_env_name" {
  description = "Name of the Container Apps environment"
  type        = string
  default     = "env-gameapp"
}

variable "container_app_name" {
  description = "Name of the Container App"
  type        = string
  default     = "gameapp"
}

variable "docker_image" {
  description = "Full Docker image reference including tag"
  type        = string
}
```

---

### `terraform/main.tf`

```hcl
terraform {
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 3.0"
    }
  }
}

provider "azurerm" {
  features {}
}

resource "azurerm_resource_group" "main" {
  name     = var.resource_group_name
  location = var.location
}

resource "azurerm_container_app_environment" "main" {
  name                = var.container_app_env_name
  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name
}

resource "azurerm_container_app" "main" {
  name                         = var.container_app_name
  container_app_environment_id = azurerm_container_app_environment.main.id
  resource_group_name          = azurerm_resource_group.main.name
  revision_mode                = "Single"

  template {
    container {
      name   = var.container_app_name
      image  = var.docker_image
      cpu    = 0.25
      memory = "0.5Gi"
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
  description = "Public URL of the deployed game app"
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
          IMAGE_TAG="${{ secrets.DOCKERHUB_USERNAME }}/gameapp:${{ github.sha }}"
          echo "image_tag=${IMAGE_TAG}" >> $GITHUB_OUTPUT

      - name: Build and push image
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.image_tag }}

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
            -out=tfplan

      - name: Terraform Apply
        working-directory: terraform
        run: terraform apply -auto-approve tfplan

      - name: Print app URL
        working-directory: terraform
        run: terraform output app_url
```

---

## How It Works — End to End

When you push to `main`:

1. **build-and-push job** checks out your code, logs in to Docker Hub, builds the image tagged with the git commit SHA, and pushes it
2. **deploy job** runs after — it authenticates to Azure using your service principal secrets, runs `terraform init` and `terraform plan` with the new image tag passed as a variable, then applies the plan
3. Terraform provisions (or updates) the resource group, Container Apps environment, and Container App pointing at the new image
4. The workflow prints the public URL as the final step

---

## Verifying It Worked

After the workflow completes:

1. Go to your GitHub repository → **Actions** tab
2. Click the latest workflow run and confirm both jobs show green ticks
3. In the **deploy** job logs, find the `Print app URL` step — copy the URL
4. Open the URL in your browser — the game should be live

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

Also include in your repository root a file named `SUBMISSION.md` with:

```markdown
## Exercise 2 Submission

- **GitHub repo URL:** https://github.com/YOUR_USERNAME/YOUR_REPO
- **Live app URL:** https://YOUR_APP.azurecontainerapps.io
- **Successful Actions run URL:** https://github.com/YOUR_USERNAME/YOUR_REPO/actions/runs/XXXXXXX
```

---

## Hints

- If the **build-and-push** job fails at login, double-check that `DOCKERHUB_TOKEN` is an access token, not your account password. Generate a new one at Docker Hub → Account Settings → Security → New Access Token.
- If the **deploy** job fails at `terraform init`, make sure the `working-directory: terraform` is set correctly on every Terraform step.
- If Terraform fails with an authentication error, check that all four `ARM_*` environment variables are set and match the service principal output exactly.
- The first time the pipeline runs, Terraform creates all resources from scratch — this takes 2–3 minutes. Subsequent runs update only what has changed.
- `revision_mode = "Single"` means Terraform replaces the running revision when the image changes. This is correct for this exercise.
- Do not commit secrets into your repository. All credentials must be in GitHub Secrets, never in `.yml` or `.tf` files.
