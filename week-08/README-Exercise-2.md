# Exercise 2 — Push the Image to a Container Registry - Kubernetes: Container Registry

---

## Overview

Before you can deploy to a cloud Kubernetes cluster, your container image needs to live somewhere that cluster can reach. In this exercise you will push the taskline app image to a container registry of your choice — Azure Container Registry (ACR), GitHub Container Registry (GHCR), or Docker Hub — and confirm it is accessible.

This exercise is the bridge between local (Exercise 1) and cloud (Exercise 3).

---

## What You Need

- Exercise 1 completed — `tasklineapp:v1` built and working on minikube
- An account with your chosen registry (see options below)
- Azure CLI installed (required for ACR only)

---

## Choose Your Registry

Pick **one** of the three options below and follow it end to end. All three are valid for Exercise 3.

| Registry | Recommended if... |
|---|---|
| **ACR** (Azure Container Registry) | You are deploying to AKS — native integration, no credentials to manage in YAML |
| **GHCR** (GitHub Container Registry) | You are already using GitHub and want everything in one place |
| **Docker Hub** | You want the simplest setup and are comfortable with a public repo |

---

## ⚠️ Platform Note

Always build with `--platform linux/amd64` before pushing to any registry intended for AKS. AKS nodes run Linux AMD64 — an ARM image will fail to start.

```bash
docker build --platform linux/amd64 -t tasklineapp:v1 .
```

If you already built in Exercise 1 with this flag, you do not need to rebuild.

---

## Option A — Azure Container Registry (ACR)

### Step 1 — Create the registry

The ACR name must be globally unique, lowercase, and contain no hyphens.

```bash
az acr create \
  --resource-group rg-tasklineapp \
  --name acrtasklineapp \
  --sku Basic
```

### Step 2 — Log in

```bash
az acr login --name acrtasklineapp
```

### Step 3 — Tag and push

```bash
# Tag the image with the ACR login server address
docker tag tasklineapp:v1 acrtasklineapp.azurecr.io/tasklineapp:v1

# Push
docker push acrtasklineapp.azurecr.io/tasklineapp:v1
```

### Step 4 — Confirm

```bash
az acr repository list --name acrtasklineapp -o table
az acr repository show-tags --name acrtasklineapp --repository tasklineapp -o table
```

You should see `tasklineapp` listed with tag `v1`.

### Step 5 — Attach ACR to AKS

This step grants your AKS cluster permission to pull from ACR without needing credentials in every manifest. Run this once.

```bash
az aks update \
  --resource-group rg-tasklineapp \
  --name aks-tasklineapp \
  --attach-acr acrtasklineapp
```

> If your AKS cluster does not exist yet, do this step when you create it in Exercise 3. The Terraform configuration in Exercise 3 handles this automatically.

---

## Option B — GitHub Container Registry (GHCR)

### Step 1 — Create a GitHub Personal Access Token

Go to GitHub → Settings → Developer settings → Personal access tokens → Tokens (classic) → Generate new token.

Scopes required: `read:packages`, `write:packages`, `delete:packages`.

Save the token — you will not see it again.

### Step 2 — Log in

```bash
echo YOUR_GITHUB_PAT | docker login ghcr.io -u YOUR_GITHUB_USERNAME --password-stdin
```

### Step 3 — Tag and push

```bash
docker tag tasklineapp:v1 ghcr.io/YOUR_GITHUB_USERNAME/tasklineapp:v1
docker push ghcr.io/YOUR_GITHUB_USERNAME/tasklineapp:v1
```

### Step 4 — Make the package public (recommended for this exercise)

Go to github.com → Your profile → Packages → tasklineapp → Package settings → Change visibility → Public.

This avoids needing an `imagePullSecret` in your Kubernetes manifests. If you keep the package private, you will need to add an `imagePullSecret` to your deployment manifest in Exercise 3.

### Step 5 — Confirm

```bash
# Pull it back to confirm it is accessible
docker pull ghcr.io/YOUR_GITHUB_USERNAME/tasklineapp:v1
```

---

## Option C — Docker Hub

### Step 1 — Log in

```bash
docker login
```

Enter your Docker Hub username and password (or access token — recommended over password).

### Step 2 — Tag and push

```bash
docker tag tasklineapp:v1 YOUR_DOCKERHUB_USERNAME/tasklineapp:v1
docker push YOUR_DOCKERHUB_USERNAME/tasklineapp:v1
```

### Step 3 — Confirm

Go to hub.docker.com → your repositories. You should see `tasklineapp` with tag `v1`.

For a public repository, AKS can pull without any credentials. For a private repository, you will need to create an `imagePullSecret` in your manifest.

---

## Submitting Your Work

Push the following to your GitHub repository:

```
SUBMISSION.md       ← updated with Exercise 2 section
```

**Add an Exercise 2 section to `SUBMISSION.md`:**

```markdown
## Exercise 2 Submission

- **Registry used:** ACR / GHCR / Docker Hub  (delete as appropriate)
- **Full image reference:** acrtasklineapp.azurecr.io/tasklineapp:v1  (your actual image path)
- **Registry confirmation screenshot:** screenshot-registry.png
```

Also include `screenshot-registry.png` — a screenshot showing the image listed in your registry (ACR repository list, GHCR packages page, or Docker Hub repository page).

---

## Hints

- For ACR: if `az acr login` fails with an auth error, run `az login` first to refresh your Azure credentials.
- For GHCR: if `docker push` is denied, double-check that your PAT has `write:packages` scope and that you are using `--password-stdin` not pasting the token as a positional argument.
- For Docker Hub: use an access token rather than your account password. Generate one at hub.docker.com → Account Settings → Security → New Access Token.
- The image tag `v1` is fixed for this exercise. In a real pipeline you would use the git commit SHA as the tag (as in Week 7) to make every push traceable.
