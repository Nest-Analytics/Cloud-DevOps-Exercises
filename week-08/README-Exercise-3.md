# Exercise 3 — Deploy to AKS with Terraform and GitHub Actions - Kubernetes: Terraform + GitHub Actions + AKS

---

## Overview

This is the capstone exercise for this week. You will combine everything from the last three weeks:

- **Docker** — your image from Exercise 1, pushed to a registry in Exercise 2
- **Terraform** — provision the AKS cluster and (if using ACR) the container registry
- **Kubernetes YAML** — the manifests that define your Deployment and Service
- **GitHub Actions** — a pipeline that builds the image, pushes it to your registry, applies Terraform, and deploys updated manifests to AKS on every push to `main`

When it is working, pushing a code change to `main` automatically results in a live, updated app accessible at a public AKS IP.

---

## Prerequisites

- Exercise 1 completed — manifests written and tested on minikube
- Exercise 2 completed — image pushed to your chosen registry
- GitHub repository set up
- Azure account with an active subscription
- The following secrets configured in GitHub repository settings (Settings → Secrets and variables → Actions):

| Secret name | What it contains |
|---|---|
| `DOCKERHUB_USERNAME` | Docker Hub username — required if using Docker Hub or GHCR |
| `DOCKERHUB_TOKEN` | Docker Hub access token — required if using Docker Hub |
| `GHCR_TOKEN` | GitHub PAT with `read/write:packages` — required if using GHCR |
| `AZURE_CLIENT_ID` | Azure service principal client ID |
| `AZURE_CLIENT_SECRET` | Azure service principal client secret |
| `AZURE_SUBSCRIPTION_ID` | Your Azure subscription ID |
| `AZURE_TENANT_ID` | Your Azure tenant ID |

> Only add the registry secrets for the registry you chose in Exercise 2.

**Create the Azure service principal:**

```bash
az ad sp create-for-rbac \
  --name "sp-tasklineapp-k8s" \
  --role Contributor \
  --scopes /subscriptions/YOUR_SUBSCRIPTION_ID \
  --sdk-auth
```

The output JSON gives you `clientId`, `clientSecret`, `subscriptionId`, and `tenantId`.

---

## Repository Structure

Your repository should look like this when complete:

```
tasklineapp/
├── src/                        ← app source
├── package.json
├── Dockerfile
├── .dockerignore
├── k8s/
│   ├── deployment.yaml         ← minikube version (Exercise 1)
│   ├── service.yaml            ← minikube version (Exercise 1)
│   └── aks/
│       ├── deployment.yaml     ← AKS version (this exercise)
│       └── service.yaml        ← AKS version (this exercise)
├── terraform/
│   ├── main.tf
│   ├── variables.tf
│   └── outputs.tf
├── .github/
│   └── workflows/
│       └── deploy.yml
└── SUBMISSION.md
```

---

## Part 1 — AKS Kubernetes Manifests

The AKS manifests are almost identical to your minikube ones from Exercise 1. Two things change:

1. The image reference uses the full registry path from Exercise 2
2. `imagePullPolicy: Never` is removed — AKS pulls from the registry
3. Service type changes from `NodePort` to `LoadBalancer`

### `k8s/aks/deployment.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: tasklineapp
  labels:
    app: tasklineapp
spec:
  replicas: 2
  selector:
    matchLabels:
      app: tasklineapp
  template:
    metadata:
      labels:
        app: tasklineapp
    spec:
      containers:
        - name: tasklineapp
          # Use the image reference from your Exercise 2 registry:
          # ACR:        acrtasklineapp.azurecr.io/tasklineapp:v1
          # GHCR:       ghcr.io/YOUR_USERNAME/tasklineapp:v1
          # Docker Hub: YOUR_USERNAME/tasklineapp:v1
          image: YOUR_REGISTRY/tasklineapp:v1
          ports:
            - containerPort: 3000
          resources:
            requests:
              cpu: "100m"
              memory: "128Mi"
            limits:
              cpu: "250m"
              memory: "256Mi"
```

### `k8s/aks/service.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: tasklineapp-service
spec:
  type: LoadBalancer
  selector:
    app: tasklineapp
  ports:
    - protocol: TCP
      port: 80
      targetPort: 3000
```

`LoadBalancer` on AKS provisions a real Azure load balancer and assigns a public external IP. Your users connect on port 80; traffic is forwarded to the container on port 3000.

---

## Part 2 — Terraform

Create the `terraform/` folder with three files. This configuration provisions the resource group, ACR (if you chose ACR), and the AKS cluster.

### `terraform/variables.tf`

```hcl
variable "resource_group_name" {
  description = "Azure resource group name"
  type        = string
  default     = "rg-tasklineapp"
}

variable "location" {
  description = "Azure region"
  type        = string
  default     = "westeurope"
}

variable "aks_cluster_name" {
  description = "AKS cluster name"
  type        = string
  default     = "aks-tasklineapp"
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
```

You must create that storage account/container once in Azure first, and fill in this details in the backend "azurerm"

### `terraform/main.tf`

```hcl
terraform {
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 3.0"
    }
  }
  backend "azurerm" {
    resource_group_name  = "YOUR_TFSTATE_RG"
    storage_account_name = "YOURTFSTATESTORAGE"
    container_name       = "tfstate"
    key                  = "tasklineapp.terraform.tfstate"
  }
}

provider "azurerm" {
  features {
    resource_group {
      prevent_deletion_if_contains_resources = false
    }
  }
}

resource "azurerm_resource_group" "main" {
  name     = var.resource_group_name
  location = var.location
}

# ACR — only created if acr_name variable is provided
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
    vm_size    = "Standard_B2s"
  }

  identity {
    type = "SystemAssigned"
  }
}

# Grant AKS permission to pull from ACR (only if ACR is being used)
resource "azurerm_role_assignment" "aks_acr_pull" {
  count                            = var.acr_name != "" ? 1 : 0
  principal_id                     = azurerm_kubernetes_cluster.main.kubelet_identity[0].object_id
  role_definition_name             = "AcrPull"
  scope                            = azurerm_container_registry.main[0].id
  skip_service_principal_aad_check = true
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
  description = "ACR login server (empty if ACR not used)"
  value       = var.acr_name != "" ? azurerm_container_registry.main[0].login_server : "N/A"
}
```

---

## Part 3 — GitHub Actions Workflow

Create `.github/workflows/deploy.yml`.

The pipeline has three jobs: build and push the image, provision infrastructure with Terraform, then deploy the Kubernetes manifests.

```yaml
name: Build, Push, Provision and Deploy

on:
  push:
    branches:
      - main

jobs:

  # ── JOB 1: Build and push the Docker image ──────────────────────────────────
  build-and-push:
    name: Build and Push Image
    runs-on: ubuntu-latest

    outputs:
      image_tag: ${{ steps.tag.outputs.image_tag }}

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      # --- ACR login (use this block if using ACR) ---
      - name: Log in to ACR
        if: ${{ vars.REGISTRY == 'acr' }}
        uses: azure/docker-login@v1
        with:
          login-server: ${{ secrets.ACR_LOGIN_SERVER }}
          username: ${{ secrets.AZURE_CLIENT_ID }}
          password: ${{ secrets.AZURE_CLIENT_SECRET }}

      # --- Docker Hub login (use this block if using Docker Hub) ---
      - name: Log in to Docker Hub
        if: ${{ vars.REGISTRY == 'dockerhub' }}
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}

      # --- GHCR login (use this block if using GHCR) ---
      - name: Log in to GHCR
        if: ${{ vars.REGISTRY == 'ghcr' }}
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GHCR_TOKEN }}

      - name: Generate image tag
        id: tag
        run: |
          # Replace YOUR_REGISTRY_PATH with your full image path prefix:
          # ACR:        acrtasklineapp.azurecr.io/tasklineapp
          # Docker Hub: YOUR_DOCKERHUB_USERNAME/tasklineapp
          # GHCR:       ghcr.io/YOUR_GITHUB_USERNAME/tasklineapp
          IMAGE_TAG="YOUR_REGISTRY_PATH:${{ github.sha }}"
          echo "image_tag=${IMAGE_TAG}" >> $GITHUB_OUTPUT

      - name: Build and push image
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ${{ steps.tag.outputs.image_tag }}
          platforms: linux/amd64

  # ── JOB 2: Provision infrastructure with Terraform ──────────────────────────
  terraform:
    name: Provision AKS with Terraform
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
        run: terraform plan -out=tfplan

      - name: Terraform Apply
        working-directory: terraform
        run: terraform apply -auto-approve tfplan

  # ── JOB 3: Deploy manifests to AKS ──────────────────────────────────────────
  deploy:
    name: Deploy to AKS
    runs-on: ubuntu-latest
    needs: [build-and-push, terraform]

    env:
      ARM_CLIENT_ID: ${{ secrets.AZURE_CLIENT_ID }}
      ARM_CLIENT_SECRET: ${{ secrets.AZURE_CLIENT_SECRET }}
      ARM_SUBSCRIPTION_ID: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
      ARM_TENANT_ID: ${{ secrets.AZURE_TENANT_ID }}

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Log in to Azure
        uses: azure/login@v1
        with:
          creds: >-
            {
              "clientId": "${{ secrets.AZURE_CLIENT_ID }}",
              "clientSecret": "${{ secrets.AZURE_CLIENT_SECRET }}",
              "subscriptionId": "${{ secrets.AZURE_SUBSCRIPTION_ID }}",
              "tenantId": "${{ secrets.AZURE_TENANT_ID }}"
            }

      - name: Get AKS credentials
        run: |
          az aks get-credentials \
            --resource-group rg-tasklineapp \
            --name aks-tasklineapp \
            --overwrite-existing

      - name: Update image in deployment manifest
        run: |
          sed -i "s|YOUR_REGISTRY_PATH:v1|${{ needs.build-and-push.outputs.image_tag }}|g" \
            k8s/aks/deployment.yaml

      - name: Apply Kubernetes manifests
        run: |
          kubectl apply -f k8s/aks/deployment.yaml
          kubectl apply -f k8s/aks/service.yaml

      - name: Wait for rollout
        run: kubectl rollout status deployment/tasklineapp --timeout=120s

      - name: Get external IP
        run: |
          echo "Waiting for external IP..."
          for i in {1..20}; do
            IP=$(kubectl get svc tasklineapp-service \
              -o jsonpath='{.status.loadBalancer.ingress[0].ip}' 2>/dev/null)
            if [ -n "$IP" ]; then
              echo "App is live at: http://${IP}"
              break
            fi
            echo "Still waiting... (${i}/20)"
            sleep 15
          done
```

> **Registry variable:** Add a GitHub Actions variable (not a secret) called `REGISTRY` set to `acr`, `dockerhub`, or `ghcr` so the correct login step activates. Go to Settings → Secrets and variables → Actions → Variables tab.

---

## How the Pipeline Works End to End

When you push to `main`:

1. **build-and-push** — builds the image for `linux/amd64`, tags it with the git commit SHA, and pushes to your registry
2. **terraform** — runs `terraform init`, `plan`, and `apply` to provision or update the resource group, ACR (if used), and AKS cluster. On the first run this takes 5–8 minutes. Subsequent runs are fast — Terraform only changes what has changed.
3. **deploy** — authenticates to Azure, pulls AKS credentials, replaces the image tag in the manifest with the SHA-tagged image from Job 1, applies both manifests to AKS, waits for the rollout, then prints the external IP

---

## Verifying It Worked

1. Go to your GitHub repository → **Actions** tab
2. Click the latest workflow run — confirm all three jobs show green ticks
3. In the **deploy** job logs, find the `Get external IP` step — copy the IP
4. Open `http://<EXTERNAL-IP>` in your browser — the app should be live

---

## Submitting Your Work

Push all of the following to `main`:

```
├── Dockerfile
├── .dockerignore
├── k8s/
│   ├── deployment.yaml         ← minikube (Exercise 1)
│   ├── service.yaml            ← minikube (Exercise 1)
│   └── aks/
│       ├── deployment.yaml     ← AKS (this exercise)
│       └── service.yaml        ← AKS (this exercise)
├── terraform/
│   ├── main.tf
│   ├── variables.tf
│   └── outputs.tf
└── .github/
    └── workflows/
        └── deploy.yml
```

Add an Exercise 3 section to `SUBMISSION.md`:

```markdown
## Exercise 3 Submission

- **GitHub repo URL:** https://github.com/YOUR_USERNAME/YOUR_REPO
- **Registry used:** ACR / GHCR / Docker Hub
- **Full image reference:** YOUR_REGISTRY/tasklineapp:SHA
- **Live app URL:** http://EXTERNAL_IP
- **Successful Actions run URL:** https://github.com/YOUR_USERNAME/YOUR_REPO/actions/runs/XXXXXXX
- **Screenshot:** screenshot-aks.png
```

Also include `screenshot-aks.png` — a browser screenshot showing the app at the AKS external IP.

---

## Hints

- **Terraform first run takes time.** AKS cluster creation takes 5–8 minutes. The pipeline will appear to hang at the Terraform Apply step — this is normal.
- **`sed` image replacement.** The deploy job uses `sed` to swap the placeholder image tag in `deployment.yaml` with the real SHA tag. Make sure `YOUR_REGISTRY_PATH:v1` in the manifest matches exactly what the `sed` command targets.
- **Terraform on second run.** If the AKS cluster already exists from a previous run, Terraform will detect no changes and the apply will complete in seconds. This is correct behaviour.
- **ACR attach.** The `azurerm_role_assignment` resource in `main.tf` grants AKS the `AcrPull` role on ACR automatically. You do not need to run `az aks update --attach-acr` manually.
- **GHCR private packages.** If your GHCR package is private, add an `imagePullSecret` to your AKS deployment manifest and reference it under `spec.template.spec.imagePullSecrets`.
- **Do not commit secrets.** All credentials live in GitHub Secrets. Never put them in `.yml`, `.tf`, or any tracked file.
