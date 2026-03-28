# Acceptance Criteria — Exercise 2 - Week 8 | Kubernetes: Container Registry

---

## Submission Checklist

---

### Required Files

| File | Required |
|---|---|
| `SUBMISSION.md` (Exercise 2 section added) | Yes |
| `screenshot-registry.png` | Yes |

---

### Image Push — Must Pass All

- [ ] Image is pushed to exactly one of: ACR, GHCR, or Docker Hub
- [ ] Image is tagged `v1` (or a valid semantic/SHA tag — not `latest` alone)
- [ ] Image was built with `--platform linux/amd64` before pushing (assessor should verify this via the registry manifest if needed)
- [ ] Full image reference is recorded in `SUBMISSION.md` in the correct format for the chosen registry:
  - ACR: `NAME.azurecr.io/demoapp:v1`
  - GHCR: `ghcr.io/USERNAME/demoapp:v1`
  - Docker Hub: `USERNAME/demoapp:v1`

---

### `SUBMISSION.md` — Must Contain All

- [ ] Registry used is identified (ACR / GHCR / Docker Hub)
- [ ] Full image reference is present and correctly formatted
- [ ] Reference to `screenshot-registry.png`

---

### Screenshot — Must Pass All

- [ ] Screenshot shows the image listed in the registry UI or CLI output
- [ ] The image name (`demoapp`) is visible
- [ ] The tag (`v1`) is visible
- [ ] For ACR: `az acr repository show-tags` output or the Azure Portal ACR page
- [ ] For GHCR: the Packages page on github.com showing the package
- [ ] For Docker Hub: the repository page on hub.docker.com

---

### ACR-Specific (if ACR was chosen)

- [ ] `az aks update --attach-acr` has been run, OR the student notes in `SUBMISSION.md` that this will be completed as part of Exercise 3 Terraform provisioning

---

## Not Accepted If

- Image is tagged only as `latest` with no other tag
- Image was not built with `--platform linux/amd64` (will cause failures in Exercise 3)
- Full image reference in `SUBMISSION.md` is incorrect or uses a placeholder that was not updated
- Screenshot does not show the image in the registry — a screenshot of a terminal build output is not sufficient
- Registry is not one of the three accepted options

---

## Notes

Exercise 2 is a prerequisite for Exercise 3 — the image reference from this submission is what goes into the Kubernetes manifests and Terraform configuration in Exercise 3.
