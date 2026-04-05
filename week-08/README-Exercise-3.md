# Exercise 3 — Deploy to AKS with Terraform and GitHub Actions | Kubernetes: Terraform + GitHub Actions + AKS

---

## Overview

This is the capstone exercise for Week 8. You will combine everything from the last three weeks:

- **Docker** — your image from Exercise 1, pushed to a registry in Exercise 2
- **Terraform** — provision the AKS cluster with state stored remotely in Azure
- **Kubernetes YAML** — the manifests that define your Deployment and Service
- **GitHub Actions** — a pipeline that builds the image, pushes it to your registry, applies Terraform, and deploys to AKS on every push to `main`

When it is working, pushing a code change to `main` automatically results in a live, updated app accessible at a public AKS IP.

---

## No Hardcoded Values — How This Exercise Is Structured

Nothing sensitive or environment-specific is hardcoded in any file. Everything flows from GitHub Secrets and Variables. This means:

- You can change any resource name by updating a variable — no file edits
- Nothing sensitive ever touches the repository
- The same files work for any team member or environment

Two mechanisms make this work:

**`TF_VAR_*` environment variables** — any env var prefixed `TF_VAR_` is automatically read by Terraform as the value for the matching input variable. No `-var` flags needed in `plan` or `apply`.

**`-backend-config` flags at `terraform init`** — the `backend "azurerm"` block in `main.tf` is intentionally empty. Connection details are passed as flags at init time, sourced from GitHub Secrets. Nothing about the backend is in any committed file.

---

## Prerequisites

- Exercise 1 completed — manifests written and tested on minikube
- Exercise 2 completed — image pushed to your chosen registry
- GitHub repository set up
- Azure account with an active subscription

---

## GitHub Secrets and Variables

Go to your repository → **Settings → Secrets and variables → Actions**.

### Secrets (sensitive — always masked in logs)

| Secret name | What it contains |
|---|---|
| `AZURE_CLIENT_ID` | Service principal client ID |
| `AZURE_CLIENT_SECRET` | Service principal client secret |
| `AZURE_SUBSCRIPTION_ID` | Your Azure subscription ID |
| `AZURE_TENANT_ID` | Your Azure tenant ID |
| `TF_STATE_RESOURCE_GROUP` | Resource group containing the Terraform state storage account |
| `TF_STATE_STORAGE_ACCOUNT` | Storage account name for Terraform remote state |
| `GHCR_TOKEN` | GitHub PAT with `read/write:packages` — required if using GHCR |
| `DOCKERHUB_USERNAME` | Docker Hub username — required if using Docker Hub |
| `DOCKERHUB_TOKEN` | Docker Hub access token — required if using Docker Hub |
| `ACR_LOGIN_SERVER` | ACR login server — required if using ACR |
| `APP_USERNAME` | Taskline app username |
| `APP_PASSWORD` | Taskline app password |
| `API` | Taskline API key |

### Variables (non-sensitive — visible in logs, that is fine)

Go to the **Variables tab** (not Secrets).

| Variable name | What it contains |
|---|---|
| `REGISTRY` | `ghcr`, `dockerhub`, or `acr` |
| `REGISTRY_PATH` | Full image path prefix, e.g. `ghcr.io/YOUR_USERNAME/tasklineapp` |
| `TF_VAR_resource_group_name` | Azure resource group, e.g. `learn-rg` |
| `TF_VAR_location` | Azure region, e.g. `westeurope` |
| `TF_VAR_aks_cluster_name` | AKS cluster name, e.g. `aks-taskline-learn` |
| `TF_VAR_acr_name` | ACR name if using ACR — leave empty otherwise |
| `APP_TITLE` | `Taskline` |
| `VITE_APP_TITLE` | `Taskline` |
| `PORT` | `3000` |

**Create the Azure service principal:**

```bash
az ad sp create-for-rbac \
  --name "sp-tasklineapp-k8s" \
  --role Contributor \
  --scopes /subscriptions/YOUR_SUBSCRIPTION_ID \
  --sdk-auth
```

The output JSON gives you `clientId`, `clientSecret`, `subscriptionId`, and `tenantId`. Add each to GitHub Secrets.

---

## ⚠️ Before You Start — Set Up the Terraform Remote State Backend

Each pipeline run starts on a fresh runner with no memory of previous runs. Without remote state, Terraform cannot tell what it already created and will try to recreate everything on every push. Create the state storage once manually before your first pipeline run.

### Step 1 — Create a resource group for state

Keep it separate from app infrastructure so it is never accidentally destroyed.

```bash
az group create \
  --name rg-tfstate \
  --location westeurope
```

### Step 2 — Create the storage account

Name must be globally unique, 3–24 characters, lowercase letters and numbers only.

```bash
az storage account create \
  --name YOURTFSTATESTORAGE \
  --resource-group rg-tfstate \
  --location westeurope \
  --sku Standard_LRS \
  --kind StorageV2
```

### Step 3 — Create the blob container

```bash
az storage container create \
  --name tfstate1 \
  --account-name YOURTFSTATESTORAGE
```

### Step 4 — Grant the service principal access

`Contributor` on the subscription lets the service principal create Azure resources but does not grant blob data access. `Storage Blob Data Contributor` on the storage account is required separately — without it `terraform init` fails with a 403.

```bash
STORAGE_ID=$(az storage account show \
  --name YOURTFSTATESTORAGE \
  --resource-group rg-tfstate \
  --query id -o tsv)

SP_ID=$(az ad sp show \
  --id YOUR_AZURE_CLIENT_ID \
  --query id -o tsv)

az role assignment create \
  --assignee $SP_ID \
  --role "Storage Blob Data Contributor" \
  --scope $STORAGE_ID
```

### Step 5 — Add to GitHub Secrets

| Secret name | Value |
|---|---|
| `TF_STATE_RESOURCE_GROUP` | `rg-tfstate` |
| `TF_STATE_STORAGE_ACCOUNT` | your actual storage account name |

### Step 6 — Verify locally (recommended)

```bash
cd terraform

export ARM_CLIENT_ID=YOUR_CLIENT_ID
export ARM_CLIENT_SECRET=YOUR_CLIENT_SECRET
export ARM_SUBSCRIPTION_ID=YOUR_SUBSCRIPTION_ID
export ARM_TENANT_ID=YOUR_TENANT_ID

terraform init \
  -backend-config="resource_group_name=rg-tfstate" \
  -backend-config="storage_account_name=YOURTFSTATESTORAGE" \
  -backend-config="container_name=tfstate1" \
  -backend-config="key=tasklineapp.terraform.tfstate"
```

If this completes without errors the pipeline will work.

---

## Repository Structure

```
tasklineapp/
├── src/
├── package.json
├── Dockerfile
├── .dockerignore
├── k8s/
│   ├── deployment.yaml       ← minikube version (Exercise 1)
│   ├── service.yaml          ← minikube version (Exercise 1)
│   └── aks/
│       ├── deployment.yaml   ← AKS version (this exercise)
│       └── service.yaml      ← AKS version (this exercise)
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

Three things change from the minikube manifests in Exercise 1:

1. `imagePullPolicy: Never` is removed — AKS pulls from the registry
2. `imagePullSecrets` is added — required for GHCR and private registries
3. Service type changes from `NodePort` to `LoadBalancer`
4. All six env vars are wired from the `taskline-secrets` Kubernetes Secret — the pipeline creates this secret before applying the manifests

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
      imagePullSecrets:
        - name: ghcr-secret
      containers:
        - name: tasklineapp
          # Placeholder — replaced by the pipeline sed command at deploy time
          image: REGISTRY_IMAGE_PLACEHOLDER
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 3000
          env:
            - name: APP_TITLE
              valueFrom:
                secretKeyRef:
                  name: taskline-secrets
                  key: APP_TITLE
            - name: APP_USERNAME
              valueFrom:
                secretKeyRef:
                  name: taskline-secrets
                  key: APP_USERNAME
            - name: APP_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: taskline-secrets
                  key: APP_PASSWORD
            - name: API
              valueFrom:
                secretKeyRef:
                  name: taskline-secrets
                  key: API
            - name: VITE_APP_TITLE
              valueFrom:
                secretKeyRef:
                  name: taskline-secrets
                  key: VITE_APP_TITLE
            - name: PORT
              valueFrom:
                secretKeyRef:
                  name: taskline-secrets
                  key: PORT
          volumeMounts:
            - name: taskline-secrets-volume
              mountPath: /etc/secrets
              readOnly: true
          resources:
            requests:
              cpu: "100m"
              memory: "128Mi"
            limits:
              cpu: "250m"
              memory: "256Mi"
      volumes:
        - name: taskline-secrets-volume
          secret:
            secretName: taskline-secrets
```

> `imagePullSecrets: ghcr-secret` — the pipeline creates this secret in AKS before applying manifests. If you are using Docker Hub with a public image or ACR with the `AcrPull` role, this entry is not strictly required but does no harm if present.

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

---

## Part 2 — Terraform

### `terraform/variables.tf`

Key variables have no defaults — they must be supplied explicitly via `TF_VAR_*` env vars. This prevents silent deployment to the wrong resource group.

```hcl
variable "resource_group_name" {
  description = "Azure resource group name"
  type        = string
}

variable "location" {
  description = "Azure region"
  type        = string
}

variable "aks_cluster_name" {
  description = "AKS cluster name"
  type        = string
}

variable "node_count" {
  description = "Number of AKS nodes"
  type        = number
  default     = 2
}

variable "acr_name" {
  description = "ACR name — leave empty if not using ACR"
  type        = string
  default     = ""
}
```

### `terraform/main.tf`

```hcl
terraform {
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 3.0"
    }
  }

  # Empty backend block — all connection details are passed via
  # -backend-config flags at terraform init time.
  backend "azurerm" {}
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
}

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
  description = "ACR login server — N/A if not using ACR"
  value       = var.acr_name != "" ? azurerm_container_registry.main[0].login_server : "N/A"
}
```

---

## Part 3 — GitHub Actions Workflow

Create `.github/workflows/deploy.yml`:

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

      - name: Log in to ACR
        if: vars.REGISTRY == 'acr'
        uses: azure/docker-login@v1
        with:
          login-server: ${{ secrets.ACR_LOGIN_SERVER }}
          username: ${{ secrets.AZURE_CLIENT_ID }}
          password: ${{ secrets.AZURE_CLIENT_SECRET }}

      - name: Log in to Docker Hub
        if: vars.REGISTRY == 'dockerhub'
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}

      - name: Log in to GHCR
        if: vars.REGISTRY == 'ghcr'
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GHCR_TOKEN }}

      - name: Generate image tag
        id: tag
        run: |
          # REGISTRY_PATH is a GitHub Variable set to your full image path prefix:
          # GHCR:       ghcr.io/YOUR_GITHUB_USERNAME/tasklineapp
          # Docker Hub: YOUR_DOCKERHUB_USERNAME/tasklineapp
          # ACR:        acrtasklineapp.azurecr.io/tasklineapp
          IMAGE_TAG="${{ vars.REGISTRY_PATH }}:${{ github.sha }}"
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
      # ARM_* authenticates Terraform to Azure for resource provisioning
      # and remote state blob access
      ARM_CLIENT_ID: ${{ secrets.AZURE_CLIENT_ID }}
      ARM_CLIENT_SECRET: ${{ secrets.AZURE_CLIENT_SECRET }}
      ARM_SUBSCRIPTION_ID: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
      ARM_TENANT_ID: ${{ secrets.AZURE_TENANT_ID }}
      # TF_VAR_* are picked up automatically by Terraform as variable values
      TF_VAR_resource_group_name: ${{ vars.TF_VAR_resource_group_name }}
      TF_VAR_location: ${{ vars.TF_VAR_location }}
      TF_VAR_aks_cluster_name: ${{ vars.TF_VAR_aks_cluster_name }}
      TF_VAR_acr_name: ${{ vars.TF_VAR_acr_name }}

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Set up Terraform
        uses: hashicorp/setup-terraform@v3

      - name: Terraform Init
        working-directory: terraform
        run: |
          # Backend details come from GitHub Secrets — nothing hardcoded in main.tf
          terraform init \
            -backend-config="resource_group_name=${{ secrets.TF_STATE_RESOURCE_GROUP }}" \
            -backend-config="storage_account_name=${{ secrets.TF_STATE_STORAGE_ACCOUNT }}" \
            -backend-config="container_name=tfstate1" \
            -backend-config="key=tasklineapp.terraform.tfstate"

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
            --resource-group ${{ vars.TF_VAR_resource_group_name }} \
            --name ${{ vars.TF_VAR_aks_cluster_name }} \
            --overwrite-existing

      - name: Create GHCR image pull secret
        if: vars.REGISTRY == 'ghcr'
        run: |
          # Creates the ghcr-secret referenced in the deployment manifest.
          # Idempotent — safe to run on every push.
          kubectl create secret docker-registry ghcr-secret \
            --docker-server=ghcr.io \
            --docker-username=${{ github.actor }} \
            --docker-password=${{ secrets.GHCR_TOKEN }} \
            --dry-run=client -o yaml | kubectl apply -f -

      - name: Create or update Kubernetes app secret
        run: |
          # Creates taskline-secrets referenced by secretKeyRef in the manifest.
          # Values come from GitHub Secrets and Variables — nothing hardcoded.
          # Idempotent — safe to run on every push.
          kubectl create secret generic taskline-secrets \
            --from-literal=APP_TITLE="${{ vars.APP_TITLE }}" \
            --from-literal=APP_USERNAME="${{ secrets.APP_USERNAME }}" \
            --from-literal=APP_PASSWORD="${{ secrets.APP_PASSWORD }}" \
            --from-literal=API="${{ secrets.API }}" \
            --from-literal=VITE_APP_TITLE="${{ vars.VITE_APP_TITLE }}" \
            --from-literal=PORT="${{ vars.PORT }}" \
            --dry-run=client -o yaml | kubectl apply -f -

      - name: Update image in deployment manifest
        run: |
          # Replace REGISTRY_IMAGE_PLACEHOLDER with the real SHA-tagged image from Job 1
          sed -i "s|REGISTRY_IMAGE_PLACEHOLDER|${{ needs.build-and-push.outputs.image_tag }}|g" \
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

---

## How the Pipeline Works End to End

When you push to `main`:

1. **build-and-push** — the `REGISTRY` variable selects the correct login step; `REGISTRY_PATH` constructs the full image tag with the commit SHA; image is built for `linux/amd64` and pushed; the full tag is passed as a job output
2. **terraform** — `ARM_*` secrets authenticate to Azure; `TF_VAR_*` variables supply all resource names automatically; `terraform init` connects to the remote state via `-backend-config` flags; `plan` and `apply` provision or update the cluster. First run takes 5–8 minutes; subsequent runs are fast
3. **deploy** — logs in to Azure; gets AKS credentials; creates `ghcr-secret` if using GHCR; creates or updates `taskline-secrets` with all six app values; replaces `REGISTRY_IMAGE_PLACEHOLDER` in the manifest with the real image tag; applies both manifests; waits for the rollout; prints the external IP

---

## Verifying It Worked

1. Repository → **Actions** tab → latest workflow run → all three jobs green
2. In the **deploy** job logs → `Get external IP` step → copy the IP
3. Open `http://<EXTERNAL-IP>` — the app should be live

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
- **Registry used:** GHCR / Docker Hub / ACR
- **Full image reference:** YOUR_REGISTRY/tasklineapp:SHA
- **Terraform state storage account:** YOURTFSTATESTORAGE
- **Resource group used:** learn-rg
- **AKS cluster name:** aks-taskline-learn
- **Live app URL:** http://EXTERNAL_IP
- **Successful Actions run URL:** https://github.com/YOUR_USERNAME/YOUR_REPO/actions/runs/XXXXXXX
- **Screenshot:** screenshot-aks.png
```

Include `screenshot-aks.png` — browser screenshot of the app at the AKS external IP.

---

## Hints

- **`container_name=tfstate1`** — this must match the container you created in Step 3 of the backend setup. If you used a different name, update both places to match.
- **`REGISTRY_IMAGE_PLACEHOLDER` must be exact** — the `sed` command targets this exact string. Do not change it in the manifest.
- **`TF_VAR_*` env vars are automatic** — Terraform picks them up without any `-var` flags in `plan` or `apply`.
- **First run takes time** — AKS cluster creation takes 5–8 minutes. The pipeline will appear to hang at Terraform Apply — this is normal.
- **Subsequent runs are fast** — Terraform compares state and only changes what has changed. If the cluster exists, apply completes in seconds.
- **`Storage Blob Data Contributor` is separate from `Contributor`** — both are required. Missing the blob role gives a 403 at `terraform init`.
- **Both Kubernetes secrets are idempotent** — `--dry-run=client -o yaml | kubectl apply -f -` creates if missing, updates if present. Safe on every push.
- **GHCR packages must be public or the pull secret must exist** — the pipeline creates `ghcr-secret` before applying manifests, so private packages work. If you make the package public, the pull secret does no harm.
- **Do not commit secrets** — all credentials in GitHub Secrets, all non-sensitive config in GitHub Variables. Nothing sensitive in any committed file.
