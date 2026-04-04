# Acceptance Criteria — Exercise 2 | Security & IAM: Docker Image + Azure Key Vault

---

### Required Files

| File | Required |
|---|---|
| `Dockerfile` | Yes |
| `terraform/main.tf` | Yes |
| `terraform/variables.tf` | Yes |
| `terraform/outputs.tf` | Yes |
| `screenshot-kv-portal.png` | Yes |
| `screenshot-kv-cli.png` | Yes |
| `screenshot-kv-iam.png` | Yes |
| `SUBMISSION.md` | Yes |

---

### Dockerfile — Must Pass All

- [ ] Multi-stage build with a `build` stage and a `runtime` stage
- [ ] Runtime stage uses a slim Node image (e.g. `node:24-bookworm-slim`)
- [ ] `ENV NODE_ENV=production` set in runtime stage
- [ ] `EXPOSE 3000` declared
- [ ] No secret values or environment variable values hardcoded in the Dockerfile

---

### Terraform — Must Pass All

- [ ] `backend "azurerm" {}` block is present and **empty** — no values hardcoded inside it
- [ ] `azurerm_key_vault` resource is present with `rbac_authorization_enabled = true`
- [ ] `azurerm_role_assignment.kv_sp_secrets_officer` assigns `Key Vault Secrets Officer` to `data.azurerm_client_config.current.object_id` (the running identity — service principal or user)
- [ ] `azurerm_role_assignment.kv_aks_secrets_user` assigns `Key Vault Secrets User` to the AKS kubelet identity — and only `Secrets User`, not a broader role
- [ ] All three secret resources present: `APP-PASSWORD`, `APP-USERNAME`, `API-KEY`
- [ ] Each secret resource has `depends_on = [azurerm_role_assignment.kv_sp_secrets_officer]`
- [ ] `app_password`, `app_username`, `api_key` variables declared with `sensitive = true`
- [ ] `azurerm_log_analytics_workspace` present and referenced by `oms_agent` in the AKS cluster
- [ ] No secret values appear as literals in any `.tf` file

---

### Screenshots — Must Pass All

- [ ] `screenshot-kv-portal.png` — Azure Portal Key Vault → Secrets showing `APP-PASSWORD`, `APP-USERNAME`, and `API-KEY` all listed and enabled
- [ ] `screenshot-kv-cli.png` — `az keyvault secret list` output showing all three secret names
- [ ] `screenshot-kv-iam.png` — Azure Portal Key Vault → Access control (IAM) → Role assignments showing the service principal with `Key Vault Secrets Officer` and the AKS identity with `Key Vault Secrets User`

---

### `SUBMISSION.md` — Must Contain

- [ ] GitHub repo URL
- [ ] Key Vault name used
- [ ] Confirmation that both role assignments are in place
- [ ] References to all three screenshots

---

## Not Accepted If

- Dockerfile is missing — it is required before the pipeline in Exercise 3 can build anything
- `backend "azurerm"` block contains hardcoded values — they must come from `-backend-config` flags
- Key Vault created without `rbac_authorization_enabled = true`
- AKS managed identity has `Key Vault Secrets Officer` or higher instead of `Secrets User`
- `depends_on` missing from secret resources
- `sensitive = true` absent from credential variables
- Fewer than three secrets stored

---

## Instant Fail

Any secret value committed to the repository in any tracked file.

---

## Marking Notes

Pass / not yet. Exercise 2 is a prerequisite for Exercise 3. The IAM screenshot (`screenshot-kv-iam.png`) is the primary evidence that least-privilege access is correctly configured — the assessor should check both role names and the principals they are assigned to.
