# Exercise 1 — Run the App on a Local Kubernetes Cluster - Kubernetes: Local Deployment with minikube

---

## Overview

In this exercise you will take the Docker image you built in Week 7 and run it on a local Kubernetes cluster using minikube. By the end, the app should be accessible in your browser via a NodePort service — running inside Kubernetes on your own machine, not just as a plain Docker container.

---

## What You Need

- Docker Desktop installed and **running**
- The demo app source code with your `Dockerfile` from Week 7
- `kubectl` installed
- `minikube` installed

If you have not installed kubectl or minikube yet, follow the install steps from the Week 8 lecture before starting.

---

## Confirm Your Tools

Run all three checks before proceeding. Do not move on if any of them fail.

```bash
# 1. Docker is running
docker ps

# 2. kubectl is installed
kubectl version --client

# 3. minikube is installed
minikube version
```

---

## Your Task

### Step 1 — Start minikube

```bash
minikube start --driver=docker

# Confirm the cluster is up
minikube status

# Confirm kubectl is pointing at minikube
kubectl config current-context    # should print: minikube
kubectl get nodes                 # should show STATUS: Ready
```

---

### Step 2 — Build the Image

From the root of your project (where your `Dockerfile` lives):

```bash
docker build --platform linux/amd64 -t tasklineapp:v1 .
```

> **Why `--platform linux/amd64`?**
> Kubernetes nodes — even local ones — run Linux AMD64. If you are on a Mac with Apple Silicon and skip this flag, you will get architecture mismatch errors when Kubernetes tries to run the container.

---

### Step 3 — Load the Image into minikube

minikube has its own Docker daemon separate from your local one. You must explicitly load the image into it — otherwise Kubernetes will try to pull it from the internet and fail.

```bash
minikube image load tasklineapp:v1

# Confirm minikube can see it
minikube image ls | grep tasklineapp
```

---

### Step 4 — Write the Kubernetes Manifests

Create a folder called `k8s/` in your project root. Inside it, create two files.

**`k8s/deployment.yaml`**

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
          image: tasklineapp:v1
          imagePullPolicy: Never
          ports:
            - containerPort: 3000
          resources:
            requests:
              cpu: "100m"
              memory: "128Mi"
            limits:
              cpu: "250m"
              memory: "256Mi"
```

> **`imagePullPolicy: Never`** tells Kubernetes to use the locally loaded image and not attempt a pull from any registry. This is correct for minikube only — you will remove this in Exercise 3.

**`k8s/service.yaml`**

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

### Step 5 — Apply the Manifests

```bash
kubectl apply -f k8s/deployment.yaml
kubectl apply -f k8s/service.yaml
```

---

### Step 6 — Confirm Pods Are Running

```bash
kubectl get pods
```

Both pods should show `STATUS: Running` and `READY: 1/1`. If they show `ContainerCreating`, wait 20–30 seconds and run again.

If a pod shows `ImagePullBackOff`, you missed Step 3. Run `minikube image load tasklineapp:v1` and then delete the failing pod to force a restart:

```bash
kubectl delete pod <pod-name>
```

---

### Step 7 — Open the App in Your Browser

```bash
minikube service tasklineapp-service
```

This opens the app in your default browser. The URL will be something like `http://127.0.0.1:XXXXX`. The app should load and be fully functional.

---

### Step 8 — Verify Kubernetes Is Managing It

Delete one of the pods manually and watch Kubernetes replace it:

```bash
# Get the pod name
kubectl get pods

# Delete one pod
kubectl delete pod <pod-name>

# Watch Kubernetes immediately start a replacement
kubectl get pods -w
```

The replica count stays at 2. This is Kubernetes self-healing in action.

---

## Submitting Your Work

Push the following to your GitHub repository:

```
k8s/
├── deployment.yaml
└── service.yaml
screenshot-local.png        ← browser screenshot showing the app at the minikube URL
SUBMISSION.md
```

**`Acceptance-Exercise-1.md`** should contain:

```markdown
## Exercise 1 Submission

- **GitHub repo URL:** https://github.com/YOUR_USERNAME/YOUR_REPO
- **Local minikube URL used:** http://127.0.0.1:XXXXX
- **Screenshot:** screenshot-local.png
```

---

## Hints

- If `minikube start` fails, make sure Docker Desktop is running first.
- If `kubectl get nodes` shows `NotReady`, wait 30 seconds and try again — the node is still initialising.
- If the browser does not open automatically with `minikube service`, copy the URL printed in the terminal and paste it manually.
- Run `kubectl describe pod <pod-name>` first whenever something is wrong — the `Events` section at the bottom tells you exactly what failed.
- Run `kubectl logs <pod-name>` to see the application output if the pod is running but the app is not behaving correctly.
