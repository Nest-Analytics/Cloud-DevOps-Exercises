# Exercise 3 — Full Pipeline: Key Vault → Kubernetes Secret → AKS | Security & IAM: End-to-End Secrets Flow

---

## Overview

This exercise is the full integration of everything across Weeks 7–9: Docker (containerised image), Terraform (infrastructure as code), Kubernetes (AKS manifests from Week 8), GitHub Actions (CI/CD pipeline), and now Key Vault (secrets managed securely). You are not starting from scratch — you are extending the working AKS deployment from Week 8 to handle secrets properly.

You will wire the GitHub Actions pipeline to inject secrets from GitHub Secrets into AKS as a Kubernetes Secret on every deployment. The complete flow on every push to `main`:

```
GitHub Actions
  → Terraform provisions Key Vault + stores secrets
  → Deploy job reads secrets from Key Vault
  → kubectl creates Kubernetes Secret in AKS
  → Pod reads env vars from Kubernetes Secret
  → App starts with all config values present
```

No secret value touches any file, manifest, or source control at any point.

---

## Prerequisites

- Exercises 1 and 2 completed
- GitHub Actions pipeline from Week 8/9 working (three-job structure)
- Key Vault provisioned with `APP-PASSWORD`, `APP-USERNAME`, `API-KEY` stored

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
      containers:
        - name: tasklineapp
          # Replace with your actual registry path — pipeline will substitute the SHA tag
          image: YOUR_REGISTRY_PATH:v1
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

      - name: Log in to GHCR
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GHCR_TOKEN }}

      - name: Generate image tag
        id: tag
        run: |
          IMAGE_TAG="ghcr.io/${{ github.actor }}/tasklineapp:${{ github.sha }}"
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
      # These are picked up automatically as Terraform variable values
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
          terraform init -input=false \
            -backend-config="resource_group_name=${{ secrets.TF_STATE_RESOURCE_GROUP }}" \
            -backend-config="storage_account_name=${{ secrets.TF_STATE_STORAGE_ACCOUNT }}" \
            -backend-config="container_name=tfstate1" \
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

    env:
      ARM_CLIENT_ID: ${{ secrets.AZURE_CLIENT_ID }}
      ARM_CLIENT_SECRET: ${{ secrets.AZURE_CLIENT_SECRET }}
      ARM_SUBSCRIPTION_ID: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
      ARM_TENANT_ID: ${{ secrets.AZURE_TENANT_ID }}
      # Non-sensitive config from GitHub Variables
      APP_TITLE: ${{ vars.APP_TITLE }}
      VITE_APP_TITLE: ${{ vars.VITE_APP_TITLE }}
      PORT: ${{ vars.PORT }}
      # Sensitive values from GitHub Secrets
      APP_USERNAME: ${{ secrets.APP_USERNAME }}
      APP_PASSWORD: ${{ secrets.APP_PASSWORD }}
      API: ${{ secrets.API }}

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
            --resource-group learn-rg \
            --name aks-taskline-learn \
            --overwrite-existing

      - name: Update image in deployment manifest
        run: |
          sed -i "s|YOUR_REGISTRY_PATH:v1|${{ needs.build-and-push.outputs.image_tag }}|g" \
            k8s/aks/deployment.yaml

      - name: Create or update Kubernetes secret
        run: |
          kubectl create secret generic taskline-secrets \
            --from-literal=APP_TITLE="${APP_TITLE:-Taskline}" \
            --from-literal=APP_USERNAME="${APP_USERNAME}" \
            --from-literal=APP_PASSWORD="${APP_PASSWORD}" \
            --from-literal=API="${API}" \
            --from-literal=VITE_APP_TITLE="${VITE_APP_TITLE:-Taskline}" \
            --from-literal=PORT="${PORT:-3000}" \
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
| Deploy job | Reads `APP_USERNAME`, `APP_PASSWORD`, `API` from GitHub Secrets into runner environment | Runner memory only |
| `kubectl create secret` | Idempotent create/update of `taskline-secrets` in AKS from runner env vars | Kubernetes etcd |
| Pod startup | Reads env vars from `taskline-secrets` via `secretKeyRef` | Pod memory |
| App runtime | Uses values from environment — never reads from a file or hardcoded string | Process memory |

Nothing sensitive is in any committed file at any point in this chain.

---

## Dockerfile

The multi-stage Dockerfile is already provided. It uses `--platform=$BUILDPLATFORM` for the build stage and a slim runtime image. The pipeline builds with `platforms: linux/amd64` to ensure compatibility with AKS nodes.

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

1. Go to your repository → **Actions** tab
2. Confirm all three jobs show green ticks
3. In the **deploy** job logs, find the `Get external IP` step — copy the IP
4. Open `http://<EXTERNAL-IP>` — the app should be live and logged in with your `APP_USERNAME`

---

## Submitting Your Work

See `Acceptance-Exercise-3.md` for what to include and how to submit.

---

## Hints

- `--dry-run=client -o yaml | kubectl apply -f -` is idempotent — safe to run on every push. It creates the secret if it doesn't exist, updates it if it does.
- `APP_TITLE`, `VITE_APP_TITLE`, and `PORT` come from GitHub **Variables** (not Secrets) because they are not sensitive — they are visible in logs, which is fine.
- `APP_USERNAME`, `APP_PASSWORD`, and `API` come from GitHub **Secrets** — they are masked in all logs.
- The Terraform job stores secrets in Key Vault via `TF_VAR_*`. The deploy job injects them into the cluster directly from GitHub Secrets. Both refer to the same values — Key Vault is the durable store; the pipeline is the delivery mechanism.
- If the deploy job fails at `kubectl create secret` with a credential error, check that all six `env` values (`APP_TITLE`, `APP_USERNAME`, `APP_PASSWORD`, `API`, `VITE_APP_TITLE`, `PORT`) are set in GitHub Secrets/Variables.
- `YOUR_REGISTRY_PATH:v1` in the manifest is the placeholder that `sed` replaces with the real SHA-tagged image. Do not change this string — it must match exactly.
