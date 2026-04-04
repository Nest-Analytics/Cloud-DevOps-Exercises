# Acceptance Criteria — Exercise 3 | Security & IAM: Full Pipeline → Key Vault → AKS

---

### Required Files

| File | Required |
|---|---|
| `k8s/aks/deployment.yaml` | Yes |
| `k8s/aks/service.yaml` | Yes |
| `.github/workflows/deploy.yml` | Yes |
| `Dockerfile` | Yes |
| `SUBMISSION.md` | Yes |
| `screenshot-pipeline-success.png` | Yes |
| `screenshot-app-live.png` | Yes |

---

### `k8s/aks/deployment.yaml` — Must Pass All

- [ ] `imagePullPolicy: Never` is **absent** — AKS pulls from the registry
- [ ] Image field contains `YOUR_REGISTRY_PATH:v1` placeholder (replaced by `sed` in the pipeline)
- [ ] `secretKeyRef` present for all six keys: `APP_TITLE`, `APP_USERNAME`, `APP_PASSWORD`, `API`, `VITE_APP_TITLE`, `PORT` — all referencing `taskline-secrets`
- [ ] Volume and volumeMount for `/etc/secrets` present, referencing `taskline-secrets`
- [ ] No secret values appear as plain text anywhere in the file

---

### `k8s/aks/service.yaml` — Must Pass All

- [ ] `type: LoadBalancer`
- [ ] `targetPort: 3000`
- [ ] `port: 80`

---

### GitHub Actions Workflow — Must Pass All

- [ ] Three jobs: `build-and-push`, `terraform`, `deploy`
- [ ] `build-and-push` — builds with `platforms: linux/amd64`, tags with `github.sha`, pushes to registry, passes image tag as job output
- [ ] `terraform` job — `TF_VAR_app_username`, `TF_VAR_app_password`, `TF_VAR_api_key` set from GitHub Secrets; `terraform init` uses `-backend-config` flags (no hardcoded values in `backend` block)
- [ ] `deploy` job — `APP_TITLE`, `VITE_APP_TITLE`, `PORT` from GitHub Variables; `APP_USERNAME`, `APP_PASSWORD`, `API` from GitHub Secrets
- [ ] `kubectl create secret` uses `--dry-run=client -o yaml | kubectl apply -f -` (idempotent)
- [ ] Secret creation step runs **before** `kubectl apply` of the manifests
- [ ] `sed` replaces `YOUR_REGISTRY_PATH:v1` with the real SHA image tag from the `build-and-push` job output

---

### Dockerfile — Must Pass All

- [ ] Multi-stage build: separate `build` and `runtime` stages
- [ ] Runtime stage uses `node:24-bookworm-slim` (or equivalent slim image)
- [ ] `ENV NODE_ENV=production` set in runtime stage
- [ ] `EXPOSE 3000` declared

---

### Screenshots — Must Pass All

- [ ] `screenshot-pipeline-success.png` — GitHub Actions run with all three jobs showing green ticks, triggered by a push to `main`
- [ ] `screenshot-app-live.png` — the Taskline app accessible at the AKS `EXTERNAL-IP` in a browser

---

### `SUBMISSION.md` — Must Contain

- [ ] GitHub repo URL
- [ ] Registry used (GHCR / ACR / Docker Hub)
- [ ] Live app URL (`http://EXTERNAL-IP`)
- [ ] Successful pipeline run URL
- [ ] References to both screenshots

---

## Not Accepted If

- `imagePullPolicy: Never` appears in `k8s/aks/deployment.yaml`
- `k8s/aks/service.yaml` uses `NodePort` instead of `LoadBalancer`
- Secret creation step is absent from the pipeline — even if the app runs from a previously created secret
- `--dry-run=client -o yaml | kubectl apply -f -` is not used (non-idempotent creation fails on re-runs)
- Backend config values are hardcoded in `main.tf` instead of passed via `-backend-config` flags
- Image is tagged with `latest` or `v1` only — commit SHA tagging is required

---

## Instant Fail

Any secret value committed to the repository in any tracked file, including the workflow YAML.

---

## Marking Notes

A passing Exercise 3 implicitly satisfies Exercises 1 and 2. Assessors only need to mark those separately if Exercise 3 was not submitted.
