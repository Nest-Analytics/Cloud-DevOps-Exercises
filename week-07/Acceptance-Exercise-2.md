# Acceptance Criteria — Exercise 2 - Docker + Terraform + GitHub Actions → Azure

---

## Submission Checklist

The student's `main` branch must contain all required files and a `SUBMISSION.md` with the three links listed below.

---

### Required Files

| File | Required |
|---|---|
| `Dockerfile` | Yes |
| `.dockerignore` | Yes |
| `terraform/main.tf` | Yes |
| `terraform/variables.tf` | Yes |
| `terraform/outputs.tf` | Yes |
| `.github/workflows/deploy.yml` | Yes |
| `SUBMISSION.md` | Yes |

---

### `SUBMISSION.md` — Must Contain All Three

- [ ] GitHub repository URL
- [ ] Live Azure app URL (e.g. `https://gameapp.XYZ.azurecontainerapps.io`)
- [ ] Link to a successful GitHub Actions workflow run

---

### Terraform — Must Pass All

- [ ] `terraform/main.tf` uses the `azurerm` provider
- [ ] A `azurerm_resource_group` resource is defined
- [ ] A `azurerm_container_app_environment` resource is defined and references the resource group
- [ ] A `azurerm_container_app` resource is defined with:
  - [ ] `ingress` block with `external_enabled = true` and `target_port = 3000`
  - [ ] `template.container.image` set from a variable (not hardcoded)
  - [ ] `revision_mode` set to `"Single"`
- [ ] `terraform/variables.tf` includes a `docker_image` variable with no default (it must be passed in)
- [ ] `terraform/outputs.tf` outputs the app's public URL

---

### GitHub Actions Workflow — Must Pass All

- [ ] Workflow file is at `.github/workflows/deploy.yml`
- [ ] Triggered on `push` to `main`
- [ ] Workflow has two jobs: one for Docker build/push, one for Terraform deploy
- [ ] Docker job:
  - [ ] Logs in to Docker Hub using `DOCKERHUB_USERNAME` and `DOCKERHUB_TOKEN` secrets
  - [ ] Builds the image using the repository `Dockerfile`
  - [ ] Tags the image with the git commit SHA (not just `latest`)
  - [ ] Pushes the image to Docker Hub
  - [ ] Passes the image tag as a job output to the deploy job
- [ ] Terraform job:
  - [ ] Runs `terraform init`
  - [ ] Runs `terraform plan` passing the image tag via `-var="docker_image=..."`
  - [ ] Runs `terraform apply`
  - [ ] Azure credentials are sourced from `ARM_*` environment variables mapped from GitHub Secrets — not hardcoded

---

### Live App — Must Pass All

- [ ] The URL in `SUBMISSION.md` is reachable and returns the game app (not a 404, 502, or placeholder page)
- [ ] The URL is an `azurecontainerapps.io` domain (or equivalent managed service URL)
- [ ] Assessor can confirm the app is running the correct Week 7 version of the game

---

### GitHub Actions Run — Must Pass All

- [ ] The linked Actions run shows both jobs completing with green ticks (no red failures)
- [ ] The run was triggered by a push to `main` (not manually triggered)
- [ ] The `Print app URL` step in the deploy job log shows the same URL as in `SUBMISSION.md`

---

### Security — Instant Fail Conditions

A submission is **immediately failed** (not returned for resubmission) if any of the following are found committed in the repository:

- Azure client secret, subscription ID, or tenant ID in any `.tf`, `.yml`, `.env`, or other tracked file
- Docker Hub password or access token in any tracked file
- A `.env` file committed to the repository (even if it contains dummy values)

Students must re-read the Week 7 lecture on secrets management and resubmit after a security review conversation with the lecturer.

---

### Not Accepted If (return for resubmission)

- `SUBMISSION.md` is missing or incomplete (no URL, no Actions link)
- `docker_image` variable has a hardcoded default value in `variables.tf`
- Image is tagged only with `latest` — commit SHA tagging is required
- Terraform files are missing `outputs.tf` or the output does not include the app URL
- The live URL is unreachable at time of assessment
- The GitHub Actions workflow was never triggered (no runs visible in the Actions tab)
- Credentials appear in workflow YAML as plaintext (even if they are placeholder strings)

---

## Marking Notes

Exercise 2 is assessed as **pass / not yet**, same as Exercise 1. A submission that passes Exercise 2 implicitly passes all Exercise 1 criteria as well — assessors do not need to mark both separately if the student submits a working Exercise 2.

The stretch goal from the in-class exercise (app live on Azure) is the **minimum bar** for Exercise 2. Students who did not complete the in-class Azure deployment should use this exercise as the opportunity to do so with proper Terraform and pipeline structure.
