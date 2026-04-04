# Acceptance Criteria — Exercise 1 | Security & IAM: Kubernetes Secrets on minikube

---

### Required Files

| File | Required |
|---|---|
| `k8s/local/deployment.yaml` | Yes |
| `k8s/local/service.yaml` | Yes |
| `screenshot-pods-running.png` | Yes |
| `screenshot-exec-env.png` | Yes |
| `screenshot-crash-error.png` | Yes |
| `SUBMISSION.md` | Yes |

---

### `k8s/local/deployment.yaml` — Must Pass All

- [ ] `secretKeyRef` present for all six keys: `APP_TITLE`, `APP_USERNAME`, `APP_PASSWORD`, `API`, `VITE_APP_TITLE`, `PORT` — each referencing secret name `taskline-secrets`
- [ ] `imagePullPolicy: Never` is present
- [ ] Volume and volumeMount for `/etc/secrets` are present, both referencing `taskline-secrets`
- [ ] No secret values appear as plain text anywhere in the file

---

### `k8s/local/service.yaml` — Must Pass All

- [ ] `type: NodePort`
- [ ] `targetPort: 3000`
- [ ] `nodePort: 30080`

---

### Screenshots — Must Pass All

- [ ] `screenshot-pods-running.png` — `kubectl get pods` showing both replicas `Running` and `READY: 1/1`
- [ ] `screenshot-exec-env.png` — `kubectl exec` output showing `APP_TITLE` and `PORT` values from the secret (non-empty)
- [ ] `screenshot-crash-error.png` — `kubectl describe pod` Events section showing the missing secret error after the secret was deleted

---

### `SUBMISSION.md` — Must Contain

- [ ] GitHub repo URL
- [ ] Secret name used (`taskline-secrets`)
- [ ] All six keys listed
- [ ] References to all three screenshots

---

## Not Accepted If

- Any secret value is set as a plain `value:` in the manifest instead of using `secretKeyRef`
- `imagePullPolicy: Never` is missing from the local manifest
- The crash screenshot is absent — demonstrating the break is required
- Secret name in `secretKeyRef` does not match the created secret (`taskline-secrets`)

---

## Instant Fail

Any secret value committed to the repository in any tracked file.
