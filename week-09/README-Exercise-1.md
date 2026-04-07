# Exercise 1 — Kubernetes Secrets on minikube | Security & IAM: Local Secrets

---

## Overview

You will create a Kubernetes Secret containing the Taskline app's config values, deploy the app to minikube using the provided manifests, and confirm it runs correctly. You will then delete the secret and redeploy to observe what breaks — and why.

**Note that you will be using the same app but the week-09 branch**

---

## What You Need

- minikube running: `minikube start --driver=docker`
- kubectl pointing at minikube: `kubectl config current-context` should print `minikube`
- The Taskline app source code

---

## Step 1 — Point Docker at minikube and Build

```bash
eval $(minikube docker-env)

# --platform linux/amd64 ensures the image runs correctly on the cluster node.
# On a Mac with Apple Silicon (M1/M2/M3), Docker defaults to ARM — always set this flag.
docker build --platform linux/amd64 -t tasklineapp:latest .

docker images | grep tasklineapp
```

---

## Step 2 — Create the Kubernetes Secret

Create the secret with all the values the app needs. Fill in your own values where indicated:

```bash
kubectl create secret generic taskline-secrets \
  --from-literal=APP_TITLE='Taskline' \
  --from-literal=APP_USERNAME='<your-username>' \
  --from-literal=APP_PASSWORD='<your-password>' \
  --from-literal=API='<your-api-key>' \
  --from-literal=VITE_APP_TITLE='Taskline' \
  --from-literal=PORT='3000'
```

Confirm the secret exists — keys are shown, values are not:

```bash
kubectl get secret taskline-secrets
kubectl describe secret taskline-secrets
```

To prove that base64 is not encryption:

```bash
kubectl get secret taskline-secrets \
  -o jsonpath='{.data.APP_PASSWORD}' | base64 --decode
```

---

## Step 3 — Write the Manifests

Create `k8s/local/deployment.yaml`:

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
          image: tasklineapp:latest
          imagePullPolicy: Never
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

Create `k8s/local/service.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: tasklineapp-service
spec:
  type: NodePort
  selector:
    app: tasklineapp
  ports:
    - protocol: TCP
      port: 80
      targetPort: 3000
      nodePort: 30080
```

---

## Step 4 — Deploy and Verify

```bash
kubectl apply -f k8s/local/deployment.yaml
kubectl apply -f k8s/local/service.yaml

# Watch pods come up
kubectl get pods -w
```

Both pods should reach `STATUS: Running` and `READY: 1/1`.

Confirm the secret values are reaching the container:

```bash
kubectl exec -it \
  $(kubectl get pod -l app=tasklineapp -o jsonpath='{.items[0].metadata.name}') \
  -- sh -c 'echo APP_TITLE=$APP_TITLE && echo PORT=$PORT'
```

Open the app in the browser:

```bash
minikube service tasklineapp-service
```

---

## Step 5 — Demonstrate Without the Secret

Delete the secret and force a pod restart:

```bash
kubectl delete secret taskline-secrets
kubectl rollout restart deployment/tasklineapp
kubectl get pods -w
```

The new pods will enter `CrashLoopBackOff` or `CreateContainerConfigError`. Inspect the reason:

```bash
kubectl describe pod \
  $(kubectl get pod -l app=tasklineapp -o jsonpath='{.items[0].metadata.name}')
```

Look at the `Events` section — it will name exactly which secret key is missing. Take a screenshot of this error state.

Recreate the secret (Step 2) and restart the deployment to restore:

```bash
kubectl rollout restart deployment/tasklineapp
```

---

## Submitting Your Work

See `Acceptance-Exercise-1.md` for what to include and how to submit.

---

## Hints

- Create the secret **before** applying the deployment. If pods start before the secret exists they will fail — recreate and restart.
- `CreateContainerConfigError` means a `secretKeyRef` key name doesn't match what's in the secret. `CrashLoopBackOff` means the container started but the app crashed. `kubectl describe pod` tells you which.
- `imagePullPolicy: Never` must stay in the local manifest — minikube won't find the image on any registry.
- To revert Docker to your local daemon after the session: `eval $(minikube docker-env -u)`
