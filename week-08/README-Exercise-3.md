# Exercise 3 — Deploy to AKS with Terraform and GitHub Actions - Kubernetes: Terraform + GitHub Actions + AKS

---

## Overview

This is the capstone exercise for this week. You will combine everything from the last three weeks:

- **Docker** — your image from Exercise 1, pushed to a registry in Exercise 2
- **Terraform** — provision the AKS cluster and (if using ACR) the container registry, with state stored remotely in Azure
- **Kubernetes YAML** — the manifests that define your Deployment and Service
- **GitHub Actions** — a pipeline that builds the image, pushes it to your registry, applies Terraform, and deploys updated manifests to AKS on every push to `main`

When it is working, pushing a code change to `main` automatically results in a live, updated app accessible at a public AKS IP.

---

## No Hardcoded Values — How This Exercise Is Structured

You will notice that none of the Terraform files, manifests, or pipeline YAML in this exercise contain hardcoded resource names, storage account names, or registry paths. Everything is driven by environment variables and GitHub Secrets.

This is intentional. It means:

- You can rename any resource just by updating a secret or variable — no file edits
- Nothing sensitive ever touches the repository
- The same files work for any team member or any environment

There are two mechanisms at play:

**1. Terraform variables via `TF_VAR_*` environment variables**

Any environment variable prefixed with `TF_VAR_` is automatically picked up by Terraform as the value for the matching variable. For example, setting `TF_VAR_resource_group_name=rg-tasklineapp` in the pipeline is equivalent to passing `-var="resource_group_name=rg-tasklineapp"` on the command line. No changes to `.tf` files needed.

**2. Terraform backend config via `-backend-config` flags**

The `backend "azurerm"` block in `main.tf` is intentionally left empty — just a placeholder. The actual backend connection details (storage account, container, key) are passed at `terraform init` time using `-backend-config` flags. Those values come from GitHub Secrets injected as environment variables in the pipeline. This is the standard pattern for keeping backend config flexible and out of source control.

---

## Prerequisites

- Exercise 1 completed — manifests written and tested on minikube
- Exercise 2 completed — image pushed to your chosen registry
- GitHub repository set up
- Azure account with an active subscription
- The following secrets and variables configured in your GitHub repository (Settings → Secrets and variables → Actions):

### GitHub Secrets

| Secret name | What it contains |
|---|---|
| `AZURE_CLIENT_ID` | Azure service principal client ID |
| `AZURE_CLIENT_SECRET` | Azure service principal client secret |
| `AZURE_SUBSCRIPTION_ID` | Your Azure subscription ID |
| `AZURE_TENANT_ID` | Your Azure tenant ID |
| `TF_STATE_RESOURCE_GROUP` | Resource group containing the Terraform state storage account |
| `TF_STATE_STORAGE_ACCOUNT` | Storage account name for Terraform remote state |
| `DOCKERHUB_USERNAME` | Docker Hub username — required if using Docker Hub |
| `DOCKERHUB_TOKEN` | Docker Hub access token — required if using Docker Hub |
| `GHCR_TOKEN` | GitHub PAT with `read/write:packages` — required if using GHCR |
| `ACR_LOGIN_SERVER` | ACR login server, e.g. `acrtasklineapp.azurecr.io` — required if using ACR |

### GitHub Variables (not secrets — these are non-sensitive config)

Go to Settings → Secrets and variables → Actions → **Variables tab**.

| Variable name | What it contains |
|---|---|
| `REGISTRY` | `acr`, `dockerhub`, or `ghcr` — controls which registry login step runs |
| `REGISTRY_PATH` | Full image path prefix, e.g. `acrtasklineapp.azurecr.io/tasklineapp` |
| `TF_VAR_resource_group_name` | Azure resource group for the app, e.g. `rg-tasklineapp` |
| `TF_VAR_location` | Azure region, e.g. `westeurope` |
| `TF_VAR_aks_cluster_name` | AKS cluster name, e.g. `aks-tasklineapp` |
| `TF_VAR_acr_name` | ACR name if using ACR, e.g. `acrtasklineapp` — leave empty otherwise |

> Using Variables (not Secrets) for non-sensitive config like resource names means they are visible in your workflow logs, making debugging easier, while genuine credentials stay in Secrets and are always masked.

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

Terraform stores its state file so it knows what infrastructure already exists. In a pipeline, local state does not work — each runner starts fresh with no memory of previous runs. Without shared state, Terraform cannot tell what it already created and will try to recreate everything on every push.

The solution is remote state: store the `terraform.tfstate` file in an Azure Storage blob container. Every pipeline run reads and writes to the same file. This must be created **manually, once**, before your first pipeline run.

### Step 1 — Create a dedicated resource group for state

Keep state storage in its own resource group, separate from the app infrastructure. If you ever tear down the app, the state is not accidentally deleted with it.

```bash
az group create \
  --name rg-tfstate \
  --location westeurope
```

### Step 2 — Create the storage account

The name must be globally unique, 3–24 characters, lowercase letters and numbers only. Choose your own name.

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
  --name tfstate \
  --account-name YOURTFSTATESTORAGE
```

### Step 4 — Grant the service principal access to the storage account

Your service principal needs the `Storage Blob Data Contributor` role on the storage account — this is separate from the `Contributor` role on the subscription.

> **Why two role assignments?** `Contributor` on the subscription lets the service principal create Azure resources. It does not grant permission to read or write blob data — that is controlled by a separate data-plane permission. Without `Storage Blob Data Contributor`, `terraform init` will fail with a 403 error when trying to access the state file.

```bash
# Get the storage account resource ID
STORAGE_ID=$(az storage account show \
  --name YOURTFSTATESTORAGE \
  --resource-group rg-tfstate \
  --query id -o tsv)

# Get the service principal object ID
SP_ID=$(az ad sp show \
  --id YOUR_AZURE_CLIENT_ID \
  --query id -o tsv)

# Assign the role
az role assignment create \
  --assignee $SP_ID \
  --role "Storage Blob Data Contributor" \
  --scope $STORAGE_ID
```

### Step 5 — Add state details to GitHub Secrets

| Secret name | Value |
|---|---|
| `TF_STATE_RESOURCE_GROUP` | `rg-tfstate` |
| `TF_STATE_STORAGE_ACCOUNT` | your actual storage account name |

### Step 6 — Verify the backend locally (optional but recommended)

Before triggering the pipeline, confirm the backend is accessible by running `terraform init` locally with the backend values passed as flags. This catches role assignment issues before they appear in the pipeline.

```bash
cd terraform

# Authenticate as the service principal
export ARM_CLIENT_ID=YOUR_CLIENT_ID
export ARM_CLIENT_SECRET=YOUR_CLIENT_SECRET
export ARM_SUBSCRIPTION_ID=YOUR_SUBSCRIPTION_ID
export ARM_TENANT_ID=YOUR_TENANT_ID

# Init with backend config passed as flags — no hardcoded values in main.tf
terraform init \
  -backend-config="resource_group_name=rg-tfstate" \
  -backend-config="storage_account_name=YOURTFSTATESTORAGE" \
  -backend-config="container_name=tfstate" \
  -backend-config="key=tasklineapp.terraform.tfstate"
```

If this completes without errors, the pipeline will be able to use the same backend.

---

## Repository Structure

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

The AKS manifests are almost identical to the minikube ones from Exercise 1. Three things change:

1. The image reference uses the full registry path — injected by the pipeline at deploy time, not hardcoded here
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
          # Placeholder — the pipeline replaces this with the real image tag at deploy time
          image: REGISTRY_IMAGE_PLACEHOLDER
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

> The `image: REGISTRY_IMAGE_PLACEHOLDER` value is replaced by the pipeline's `sed` command using the `REGISTRY_PATH` variable and the commit SHA. Nothing is hardcoded here — the manifest works regardless of which registry you chose.

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

All resource names are defined as variables with no defaults — they must be supplied at plan time. In the pipeline, they are supplied automatically via `TF_VAR_*` environment variables set from GitHub Variables.

```hcl
variable "resource_group_name" {
  description = "Azure resource group name for the application"
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
  description = "Azure Container Registry name — leave empty if not using ACR"
  type        = string
  default     = ""
}
```

> **No defaults on the key variables.** `resource_group_name`, `location`, and `aks_cluster_name` have no defaults. This forces them to be explicitly set — either via `TF_VAR_*` env vars in the pipeline, or manually on the command line locally. You will never accidentally deploy to a wrong resource group because a default was silently used.

### `terraform/main.tf`

The `backend "azurerm"` block is intentionally empty. The connection details are passed at `terraform init` time using `-backend-config` flags — sourced from GitHub Secrets in the pipeline, or from your local environment when testing manually. Nothing about the backend is stored in the file.

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
  # See the pipeline terraform init step and the local verification
  # instructions in the Before You Start section.
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

# Grant AKS permission to pull from ACR — only when ACR is being used
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
  description = "ACR login server — N/A if ACR not used"
  value       = var.acr_name != "" ? azurerm_container_registry.main[0].login_server : "N/A"
}
```

---

## Part 3 — GitHub Actions Workflow

Create `.github/workflows/deploy.yml`.

The pipeline has three jobs. Notice how resource names, registry paths, and backend details are all sourced from GitHub Secrets and Variables — nothing is hardcoded in the YAML.

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

      # ACR login — runs only when REGISTRY variable is set to 'acr'
      - name: Log in to ACR
        if: vars.REGISTRY == 'acr'
        uses: azure/docker-login@v1
        with:
          login-server: ${{ secrets.ACR_LOGIN_SERVER }}
          username: ${{ secrets.AZURE_CLIENT_ID }}
          password: ${{ secrets.AZURE_CLIENT_SECRET }}

      # Docker Hub login — runs only when REGISTRY variable is set to 'dockerhub'
      - name: Log in to Docker Hub
        if: vars.REGISTRY == 'dockerhub'
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}

      # GHCR login — runs only when REGISTRY variable is set to 'ghcr'
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
          # REGISTRY_PATH is a GitHub Variable — set it to your full image prefix:
          # ACR:        acrtasklineapp.azurecr.io/tasklineapp
          # Docker Hub: YOUR_DOCKERHUB_USERNAME/tasklineapp
          # GHCR:       ghcr.io/YOUR_GITHUB_USERNAME/tasklineapp
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
      # ARM_* variables authenticate Terraform to Azure for both
      # resource provisioning and remote state blob access
      ARM_CLIENT_ID: ${{ secrets.AZURE_CLIENT_ID }}
      ARM_CLIENT_SECRET: ${{ secrets.AZURE_CLIENT_SECRET }}
      ARM_SUBSCRIPTION_ID: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
      ARM_TENANT_ID: ${{ secrets.AZURE_TENANT_ID }}

      # TF_VAR_* variables are automatically read by Terraform as input variable values.
      # These come from GitHub Variables (non-sensitive config) so they are visible in logs.
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
          # Backend connection details are passed as -backend-config flags.
          # The backend "azurerm" block in main.tf is empty — all config comes from here.
          # TF_STATE_RESOURCE_GROUP and TF_STATE_STORAGE_ACCOUNT come from GitHub Secrets.
          terraform init \
            -backend-config="resource_group_name=${{ secrets.TF_STATE_RESOURCE_GROUP }}" \
            -backend-config="storage_account_name=${{ secrets.TF_STATE_STORAGE_ACCOUNT }}" \
            -backend-config="container_name=tfstate" \
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
          # Resource group and cluster name come from GitHub Variables — no hardcoding
          az aks get-credentials \
            --resource-group ${{ vars.TF_VAR_resource_group_name }} \
            --name ${{ vars.TF_VAR_aks_cluster_name }} \
            --overwrite-existing

      - name: Update image in deployment manifest
        run: |
          # Replace the placeholder in the manifest with the real SHA-tagged image
          # from Job 1. REGISTRY_PATH is a GitHub Variable.
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

1. **build-and-push** — the `REGISTRY` variable selects the correct login step; `REGISTRY_PATH` variable constructs the full image tag with the commit SHA; image is built for `linux/amd64` and pushed; the full image tag is passed as a job output to downstream jobs
2. **terraform** — `ARM_*` secrets authenticate to Azure for both resource provisioning and state blob access; `TF_VAR_*` variables supply all resource names to Terraform automatically; `terraform init` uses `-backend-config` flags from secrets to connect to the remote state container; `plan` and `apply` run against the live state; on first run AKS creation takes 5–8 minutes, subsequent runs are fast
3. **deploy** — logs in to Azure; retrieves AKS credentials using `TF_VAR_resource_group_name` and `TF_VAR_aks_cluster_name` variables; `sed` replaces `REGISTRY_IMAGE_PLACEHOLDER` in the manifest with the real image tag from Job 1; applies both manifests; waits for the rollout; polls for the external IP and prints it

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
- **Terraform state storage account:** YOURTFSTATESTORAGE
- **Resource group used:** rg-tasklineapp
- **AKS cluster name:** aks-tasklineapp
- **Live app URL:** http://EXTERNAL_IP
- **Successful Actions run URL:** https://github.com/YOUR_USERNAME/YOUR_REPO/actions/runs/XXXXXXX
- **Screenshot:** screenshot-aks.png
```

Also include `screenshot-aks.png` — a browser screenshot showing the app at the AKS external IP.

---

## Hints

- **Set up the storage backend before the first push.** The pipeline fails at `terraform init` if the storage account or container does not exist. Follow the backend setup steps above first.
- **`TF_VAR_*` env vars are automatic.** You do not need `-var` flags in `terraform plan` or `apply`. Any environment variable prefixed `TF_VAR_` is picked up by Terraform automatically as the value for that variable.
- **Empty `backend "azurerm" {}` is correct.** The block must exist but can be empty. All connection details are supplied via `-backend-config` flags at `terraform init`. If you omit the block entirely, Terraform defaults to local state.
- **`Storage Blob Data Contributor` is separate from `Contributor`.** The service principal needs both: `Contributor` on the subscription to create Azure resources, and `Storage Blob Data Contributor` on the storage account to access the state blob. Missing the second gives a 403 at `terraform init`.
- **Terraform first run takes time.** AKS cluster creation takes 5–8 minutes. The pipeline will appear to hang at the Terraform Apply step — this is normal.
- **`REGISTRY_IMAGE_PLACEHOLDER` must match exactly.** The `sed` command in the deploy job targets this exact string in `k8s/aks/deployment.yaml`. Do not change it.
- **Terraform on second run.** If the AKS cluster already exists, Terraform detects no changes and apply completes in seconds. This is correct behaviour — state tells it what already exists.
- **ACR pull permission.** The `azurerm_role_assignment` resource in `main.tf` grants AKS the `AcrPull` role automatically when `TF_VAR_acr_name` is set. You do not need `az aks update --attach-acr` manually.
- **GHCR private packages.** If your GHCR package is private, add an `imagePullSecret` to `k8s/aks/deployment.yaml` under `spec.template.spec.imagePullSecrets`.
- **Do not commit secrets.** All credentials and sensitive values live in GitHub Secrets. Resource names live in GitHub Variables. Nothing sensitive goes in `.yml`, `.tf`, or any tracked file.
