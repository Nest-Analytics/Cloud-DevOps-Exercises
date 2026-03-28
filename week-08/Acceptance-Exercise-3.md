# Acceptance Criteria — Kubernetes: Terraform + GitHub Actions + AKS

---

## Submission Checklist

All required files must be present on the `main` branch and `SUBMISSION.md` must contain all three links.

---

### Required Files

| File | Required |
|---|---|
| `k8s/aks/deployment.yaml` | Yes |
| `k8s/aks/service.yaml` | Yes |
| `terraform/main.tf` | Yes |
| `terraform/variables.tf` | Yes |
| `terraform/outputs.tf` | Yes |
| `.github/workflows/deploy.yml` | Yes |
| `SUBMISSION.md` (Exercise 3 section) | Yes |
| `screenshot-aks.png` | Yes |

---

### `k8s/aks/deployment.yaml` — Must Pass All

- [ ] `imagePullPolicy: Never` is **absent** — this must not appear in the AKS manifest
- [ ] `spec.template.spec.containers[0].image` uses a full registry path — not a bare local tag
- [ ] Image path matches the registry chosen in Exercise 2 (ACR, GHCR, or Docker Hub format)
- [ ] `spec.replicas` is 2 or more
- [ ] `spec.selector.matchLabels` matches `spec.template.metadata.labels` exactly
- [ ] `containerPort: 3000` is declared

---

### `k8s/aks/service.yaml` — Must Pass All

- [ ] `spec.type: LoadBalancer` — not NodePort
- [ ] `spec.selector` matches the pod labels in `k8s/aks/deployment.yaml`
- [ ] `spec.ports[0].port: 80` (external)
- [ ] `spec.ports[0].targetPort: 3000` (container)

---

### Terraform — Must Pass All

- [ ] `terraform/main.tf` uses the `azurerm` provider version `~> 3.0`
- [ ] `azurerm_resource_group` resource is defined
- [ ] `azurerm_kubernetes_cluster` resource is defined with:
  - [ ] `default_node_pool` configured
  - [ ] `identity { type = "SystemAssigned" }` present
- [ ] If ACR was chosen in Exercise 2:
  - [ ] `azurerm_container_registry` resource is defined
  - [ ] `azurerm_role_assignment` granting `AcrPull` to the AKS kubelet identity is defined
- [ ] `terraform/outputs.tf` outputs at minimum `aks_cluster_name` and `resource_group_name`

---

### GitHub Actions Workflow — Must Pass All

- [ ] Workflow file is at `.github/workflows/deploy.yml`
- [ ] Triggered on `push` to `main`
- [ ] Workflow has three jobs: build-and-push, terraform, deploy
- [ ] **build-and-push job:**
  - [ ] Authenticates to the chosen registry using GitHub Secrets
  - [ ] Builds image with `platforms: linux/amd64`
  - [ ] Tags image with the git commit SHA (not just `v1` or `latest`)
  - [ ] Pushes image and passes the full image tag as a job output
- [ ] **terraform job:**
  - [ ] Runs after `build-and-push` (declared in `needs`)
  - [ ] Azure credentials sourced from `ARM_*` environment variables mapped from GitHub Secrets
  - [ ] Runs `terraform init`, `terraform plan`, `terraform apply`
- [ ] **deploy job:**
  - [ ] Runs after both `build-and-push` and `terraform` (declared in `needs`)
  - [ ] Retrieves AKS credentials via `az aks get-credentials`
  - [ ] Updates the image reference in the manifest with the SHA-tagged image from Job 1
  - [ ] Applies `k8s/aks/deployment.yaml` and `k8s/aks/service.yaml`
  - [ ] Waits for rollout with `kubectl rollout status`

---

### Live App — Must Pass All

- [ ] The URL in `SUBMISSION.md` is reachable and returns the app (not a 404, 502, or Kubernetes default page)
- [ ] URL is a raw IP address or domain associated with an Azure load balancer
- [ ] `screenshot-aks.png` shows the app UI with the AKS external IP or domain visible in the browser address bar

---

### GitHub Actions Run — Must Pass All

- [ ] The linked Actions run shows all three jobs completing with green ticks
- [ ] The run was triggered by a push to `main` — not manually triggered
- [ ] The `Get external IP` step in the deploy job log shows the same IP as in `SUBMISSION.md`

---

### `SUBMISSION.md` — Must Contain All

- [ ] GitHub repo URL
- [ ] Registry used (ACR / GHCR / Docker Hub)
- [ ] Full image reference including the SHA tag used in the pipeline
- [ ] Live app URL
- [ ] Successful GitHub Actions run URL
- [ ] Reference to `screenshot-aks.png`

---

### Security — Instant Fail Conditions

A submission is **immediately failed** if any of the following are found committed to the repository:

- Azure client secret, subscription ID, or tenant ID in any tracked file
- Docker Hub password, GHCR token, or ACR credentials in any tracked file
- A `.env` file containing real credentials committed to the repo

The student must review secrets management, rotate any exposed credentials, and resubmit after a conversation with the lecturer.

---

## Not Accepted If (return for resubmission)

- `imagePullPolicy: Never` present in `k8s/aks/deployment.yaml` — this breaks AKS deployments
- Service type is `NodePort` in the AKS manifest — LoadBalancer is required
- Image is tagged only as `latest` or `v1` in the pipeline — commit SHA tagging is required
- Terraform does not provision AKS (uses `kubectl apply` only without any Terraform resources)
- Three separate jobs are not present in the workflow — the build/terraform/deploy separation is required
- Live URL is unreachable at time of assessment
- `screenshot-aks.png` shows the minikube local URL, not a cloud IP

---

## Notes

**Exercise 3 is the capstone for Weeks 6–8.** A passing Exercise 3 submission implicitly satisfies Exercises 1 and 2 for this week. 
The three-job pipeline structure (build / terraform / deploy) is non-negotiable — it reflects real-world separation of concerns and is the pattern students will use in production environments.
