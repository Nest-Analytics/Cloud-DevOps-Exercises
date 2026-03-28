# Acceptance Criteria — Exercise 1 - Kubernetes: Local Deployment with minikube

---

## Submission Checklist

The following must be present in the student's GitHub repository for the submission to be accepted.

---

### Required Files

| File | Required |
|---|---|
| `k8s/deployment.yaml` | Yes |
| `k8s/service.yaml` | Yes |
| `screenshot-local.png` | Yes |
| `SUBMISSION.md` | Yes |

---

### `k8s/deployment.yaml` — Must Pass All

- [ ] `apiVersion: apps/v1` and `kind: Deployment` are correct
- [ ] `metadata.name` is set
- [ ] `spec.replicas` is set to 2 or more
- [ ] `spec.selector.matchLabels` matches `spec.template.metadata.labels` exactly — same key and value
- [ ] `spec.template.spec.containers[0].image` references `demoapp:v1` (or equivalent local tag)
- [ ] `imagePullPolicy: Never` is present — required for minikube local image usage
- [ ] `containerPort: 3000` is declared
- [ ] `resources.requests` and `resources.limits` are defined

---

### `k8s/service.yaml` — Must Pass All

- [ ] `apiVersion: v1` and `kind: Service` are correct
- [ ] `spec.type: NodePort`
- [ ] `spec.selector` matches the pod labels in `deployment.yaml` exactly
- [ ] `spec.ports[0].targetPort: 3000`
- [ ] `spec.ports[0].nodePort` is set to a value in the valid range (30000–32767)

---

### `SUBMISSION.md` — Must Contain All

- [ ] GitHub repository URL
- [ ] The minikube URL used to access the app
- [ ] Reference to the screenshot file

---

### Screenshot — Must Pass All

- [ ] Screenshot is included in the submission
- [ ] The app UI is visible and loaded — not a blank page, error page, or browser connection error
- [ ] The URL bar shows a `127.0.0.1` or `localhost` address (confirming it is the local minikube URL, not a cloud URL)

---

## Not Accepted If

- `imagePullPolicy: Never` is missing — pods will enter `ImagePullBackOff` on minikube
- `spec.selector.matchLabels` does not match `spec.template.metadata.labels` — pods will never be owned by the Deployment
- `spec.type` on the Service is `LoadBalancer` instead of `NodePort` — this stays pending on minikube indefinitely
- Screenshot shows a Kubernetes error page, `ERR_CONNECTION_REFUSED`, or a cloud URL
- `k8s/` folder is missing or manifest files are empty
- `SUBMISSION.md` is missing

