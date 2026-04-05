# Exercise 3 — Full Pipeline: Key Vault → Kubernetes Secret → AKS | Security & IAM: End-to-End Secrets Flow

---

## Overview

This exercise is the full integration of everything across Weeks 7–9: Docker (containerised image), Terraform (infrastructure as code), Kubernetes (AKS manifests from Week 8), GitHub Actions (CI/CD pipeline), and now Key Vault (secrets managed securely). You are not starting from scratch — you are extending the working AKS deployment from Week 8 to handle secrets properly.

You will wire the GitHub Actions pipeline to inject secrets into AKS as a Kubernetes Secret on every deployment. The complete flow on every push to `main`:

```
GitHub Actions
  → Terraform provisions Key Vault + stores secrets
  → Deploy job reads secrets from GitHub Secrets into runner
  → kubectl creates Kubernetes Secret in AKS
  → Pod reads env vars from Kubernetes Secret
  → App starts with all config values present
```

No secret value touches any file, manifest, or source control at any point.

---

## Prerequisites

- Exercises 1 and 2 completed
- GitHub Actions pipeline from Week 8 working (three-job structure)
- Key Vault provisioned with `APP-PASSWORD`, `APP-USERNAME`, `API-KEY` stored

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
| `TF_STATE_CONTAINER` | Blob container name for Terraform state, e.g. `tfstate1` |
| `GHCR_TOKEN` | GitHub PAT with `read/write:packages` — required if using GHCR |
| `DOCKERHUB_USERNAME` | Docker Hub username — required if using Docker Hub |
| `DOCKERHUB_TOKEN` | Docker Hub access token — required if using Docker Hub |
| `ACR_LOGIN_SERVER` | ACR login server — required if using ACR |
| `APP_USERNAME` | Taskline app username |
| `APP_PASSWORD` | Taskline app password |
| `API` | Taskline API key |

### Variables (non-sensitive — visible in logs, that is fine)

Go to the **Variables tab**.

| Variable name | What it contains |
|---|---|
| `REGISTRY` | `ghcr`, `dockerhub`, or `acr` — controls which registry login step runs |
| `REGISTRY_PATH` | Full image path prefix, e.g. `ghcr.io/YOUR_USERNAME/tasklineapp` |
| `TF_VAR_resource_group_name` | Azure resource group, e.g. `learn-rg` |
| `TF_VAR_aks_cluster_name` | AKS cluster name, e.g. `aks-taskline-learn` |
| `TF_VAR_location` | Azure region, e.g. `westeurope` |
| `TF_VAR_key_vault_name` | Key Vault name, e.g. `kv-tasklineapp` |
| `APP_TITLE` | `Taskline` |
| `VITE_APP_TITLE` | `Taskline` |
| `PORT` | `3000` |

---

## Part 1 — AKS Kubernetes Manifests

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
        - name: ghcr-secret    # only needed if using GHCR — pipeline creates this conditionally
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

## Part 2 — GitHub Actions Workflow

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
          # REGISTRY_PATH is a GitHub Variable — set it to your full image path prefix:
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
      ARM_CLIENT_ID: ${{ secrets.AZURE_CLIENT_ID }}
      ARM_CLIENT_SECRET: ${{ secrets.AZURE_CLIENT_SECRET }}
      ARM_SUBSCRIPTION_ID: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
      ARM_TENANT_ID: ${{ secrets.AZURE_TENANT_ID }}
      # TF_VAR_* are picked up automatically as Terraform variable values
      TF_VAR_resource_group_name: ${{ vars.TF_VAR_resource_group_name }}
      TF_VAR_location: ${{ vars.TF_VAR_location }}
      TF_VAR_aks_cluster_name: ${{ vars.TF_VAR_aks_cluster_name }}
      TF_VAR_key_vault_name: ${{ vars.TF_VAR_key_vault_name }}
      TF_VAR_app_username: ${{ secrets.APP_USERNAME }}
      TF_VAR_app_password: ${{ secrets.APP_PASSWORD }}
      TF_VAR_api_key: ${{ secrets.API }}

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Set up Terraform
        uses: hashicorp/setup-terraform@v3

      - name: Terraform Init
        working-directory: terraform
        run: |
          # All backend details come from GitHub Secrets — nothing hardcoded
          terraform init -input=false \
            -backend-config="resource_group_name=${{ secrets.TF_STATE_RESOURCE_GROUP }}" \
            -backend-config="storage_account_name=${{ secrets.TF_STATE_STORAGE_ACCOUNT }}" \
            -backend-config="container_name=${{ secrets.TF_STATE_CONTAINER }}" \
            -backend-config="key=tasklineapp.terraform.tfstate"

      - name: Terraform Plan
        working-directory: terraform
        run: terraform plan -input=false -out=tfplan

      - name: Terraform Apply
        working-directory: terraform
        run: terraform apply -input=false -auto-approve tfplan

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

      - name: Create GHCR image pull secret
        if: vars.REGISTRY == 'ghcr'
        run: |
          kubectl create secret docker-registry ghcr-secret \
            --docker-server=ghcr.io \
            --docker-username=${{ github.actor }} \
            --docker-password=${{ secrets.GHCR_TOKEN }} \
            --dry-run=client -o yaml | kubectl apply -f -

      - name: Update image in deployment manifest
        run: |
          # Replace placeholder with the real SHA-tagged image from Job 1
          sed -i "s|REGISTRY_IMAGE_PLACEHOLDER|${{ needs.build-and-push.outputs.image_tag }}|g" \
            k8s/aks/deployment.yaml

      - name: Create or update Kubernetes secret
        run: |
          # Values come from GitHub Secrets and Variables — nothing hardcoded
          # Idempotent — safe to run on every push
          kubectl create secret generic taskline-secrets \
            --from-literal=APP_TITLE="${{ vars.APP_TITLE }}" \
            --from-literal=APP_USERNAME="${{ secrets.APP_USERNAME }}" \
            --from-literal=APP_PASSWORD="${{ secrets.APP_PASSWORD }}" \
            --from-literal=API="${{ secrets.API }}" \
            --from-literal=VITE_APP_TITLE="${{ vars.VITE_APP_TITLE }}" \
            --from-literal=PORT="${{ vars.PORT }}" \
            --dry-run=client -o yaml | kubectl apply -f -

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

## How the Secrets Flow Works

| Step | What happens | Where secrets live |
|---|---|---|
| Terraform job | Creates Key Vault, stores `APP-PASSWORD`, `APP-USERNAME`, `API-KEY` from `TF_VAR_*` env vars | Azure Key Vault |
| Deploy job | Reads `APP_USERNAME`, `APP_PASSWORD`, `API` from GitHub Secrets directly | Runner memory only |
| `kubectl create secret` | Idempotent create/update of `taskline-secrets` in AKS | Kubernetes etcd |
| Pod startup | Reads env vars from `taskline-secrets` via `secretKeyRef` | Pod memory |
| App runtime | Uses values from environment — never reads from a file or hardcoded string | Process memory |

Nothing sensitive is in any committed file at any point in this chain.

---

## Dockerfile

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

---

## Verifying It Worked

1. Repository → **Actions** tab → latest workflow run → all three jobs green
2. **deploy** job logs → `Get external IP` step → copy the IP
3. Open `http://<EXTERNAL-IP>` — the app should be live

---

## Submitting Your Work

See `SUBMISSION.md` for what to include and how to submit.

---

## Hints

- `--dry-run=client -o yaml | kubectl apply -f -` is idempotent — creates the secret if it doesn't exist, updates it if it does. Safe on every push.
- `APP_TITLE`, `VITE_APP_TITLE`, and `PORT` come from GitHub **Variables** — not sensitive, visible in logs.
- `APP_USERNAME`, `APP_PASSWORD`, and `API` come from GitHub **Secrets** — masked in all logs.
- `TF_VAR_resource_group_name` and `TF_VAR_aks_cluster_name` are GitHub **Variables** and are reused in the deploy job to get AKS credentials — no duplication, same source of truth.
- `REGISTRY_IMAGE_PLACEHOLDER` must match exactly in the manifest — the `sed` command targets this string precisely.
- If `terraform apply` fails on Key Vault secrets with a permissions error, wait 60 seconds and re-run — RBAC propagation takes a moment.
- Do not commit secrets — all credentials in GitHub Secrets, all non-sensitive config in GitHub Variables.
