# Exercise 1 — Prometheus & Grafana | Monitoring: Metrics on minikube and AKS

---

## Overview

You will install Prometheus and Grafana on your local minikube cluster, confirm the app's `/metrics` endpoint is being scraped, run PromQL queries against live data, and build a four-panel Grafana dashboard. You then verify the same metrics stack is working on your AKS cluster.

---

## What You Need

- minikube running with the Taskline app deployed from the Week 9 demo
- Helm installed
- The AKS cluster from Week 9 still provisioned

---

## Part 1 — Local: Prometheus & Grafana on minikube

### Step 1 — Install the monitoring stack

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

helm install monitoring prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace

# Wait for everything to come up
kubectl get pods -n monitoring -w
```

All pods should reach `Running`. This installs Prometheus, Grafana, Alertmanager, Node Exporter, and kube-state-metrics in one command.

---

### Step 2 — Apply the ServiceMonitor

The ServiceMonitor tells Prometheus to scrape the Taskline app's `/metrics` endpoint.

```bash
kubectl apply -f k8s/local/servicemonitor.yaml
```

Confirm the app's `/metrics` endpoint is returning Prometheus text, not HTML:

```bash
# Terminal 1 — keep this running to hold the tunnel open
minikube service tasklineapp-service --url

# Terminal 2 — use the URL printed above
BASE_URL=http://127.0.0.1:YOUR_PORT
curl "$BASE_URL/metrics"
```

The response must be plain text starting with `# HELP` lines. If it returns HTML, the app image is stale — rebuild and restart:

```bash
eval $(minikube docker-env)
docker build --platform linux/amd64 -t tasklineapp:latest .
kubectl rollout restart deployment/tasklineapp
kubectl rollout status deployment/tasklineapp
```

---

### Step 3 — Access Grafana

```bash
# Get the admin password
kubectl get secret -n monitoring monitoring-grafana \
  -o jsonpath="{.data.admin-password}" | base64 --decode; echo

# Port-forward Grafana — run in the background
nohup kubectl port-forward -n monitoring svc/monitoring-grafana 3001:80 \
  > /dev/null 2>&1 &

# Open: http://localhost:3001
# Login: admin / <password from above>
```

---

### Step 4 — Confirm Prometheus is Scraping the App

In Grafana:

1. Go to **Connections → Data Sources** — confirm Prometheus is listed
2. Open a new browser tab and port-forward Prometheus:

```bash
nohup kubectl port-forward -n monitoring svc/monitoring-kube-prometheus-prometheus 9090:9090 \
  > /dev/null 2>&1 &
```

3. Open `http://localhost:9090`
4. Go to **Status → Targets**
5. Find `tasklineapp` in the list — state must show `UP`

---

### Step 5 — Generate Traffic

In Terminal 1, keep the minikube tunnel running. In Terminal 2:

```bash
BASE_URL=http://127.0.0.1:YOUR_PORT

# Create a task
curl -X POST "$BASE_URL/api/demo/tasks" \
  -H 'Content-Type: application/json' \
  -d '{"title":"Buy milk"}'

# Update it (use the id from the response above)
curl -X PATCH "$BASE_URL/api/demo/tasks/<TASK_ID>/status" \
  -H 'Content-Type: application/json' \
  -d '{"status":"done"}'

# Trigger a warn log
curl -X POST "$BASE_URL/login" \
  -H 'Content-Type: application/json' \
  -d '{"username":"ayo","password":"wrong"}'

# Trigger an error log
curl "$BASE_URL/api/demo/error"
```

Repeat these a few times to generate enough data for charts.

> If you get `task not found` when updating, you have 2 replicas with in-memory state and the requests hit different pods. Scale to 1 replica: `kubectl scale deployment tasklineapp --replicas=1`

---

### Step 6 — Run PromQL Queries

In Grafana go to **Explore**, select the Prometheus data source, and run each query:

```promql
taskline_http_requests_total
```

```promql
taskline_tasks_created_total
```

```promql
taskline_errors_total
```

```promql
sum(taskline_http_requests_total) by (path, status_code)
```

```promql
histogram_quantile(0.95, sum(rate(taskline_http_request_duration_seconds_bucket[5m])) by (le, path))
```

Take a screenshot of the last two queries showing results.

---

### Step 7 — Build a Four-Panel Dashboard

In Grafana: **Dashboards → New → New dashboard → Add visualization**

Build each panel using the Prometheus data source:

**Panel 1 — Requests per second**
```promql
sum(rate(taskline_http_requests_total[5m])) by (path)
```
Visualisation: Time series

**Panel 2 — p95 latency**
```promql
histogram_quantile(0.95, sum(rate(taskline_http_request_duration_seconds_bucket[5m])) by (le, path))
```
Visualisation: Time series

**Panel 3 — Error rate**
```promql
sum(rate(taskline_errors_total[5m])) by (path, type)
```
Visualisation: Time series

**Panel 4 — Tasks created in last 15 minutes**
```promql
increase(taskline_tasks_created_total[15m])
```
Visualisation: Stat or Time series

Save the dashboard. Take a screenshot showing all four panels with data.

---

## Part 2 — AKS: Confirm Prometheus is Running on the Cluster

Switch kubectl context to your AKS cluster:

```bash
az aks get-credentials \
  --resource-group $TF_VAR_resource_group_name \
  --name $TF_VAR_aks_cluster_name \
  --overwrite-existing

kubectl config current-context
```

Install the same monitoring stack on AKS:

```bash
helm install monitoring prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace

kubectl get pods -n monitoring -w
```

Apply the ServiceMonitor:

```bash
kubectl apply -f k8s/aks/servicemonitor.yaml
```

Port-forward Grafana from AKS to your local machine:

```bash
kubectl get secret -n monitoring monitoring-grafana \
  -o jsonpath="{.data.admin-password}" | base64 --decode; echo

nohup kubectl port-forward -n monitoring svc/monitoring-grafana 3002:80 \
  > /dev/null 2>&1 &
```

Open `http://localhost:3002`. Go to **Explore → Prometheus**. Run:

```promql
taskline_http_requests_total
```

Confirm results appear. Take a screenshot showing the query returning data from the AKS cluster.

---

## Submitting Your Work

See `Acceptance-Exercise-1.md` for what to include.

Add an Exercise 1 section to `SUBMISSION.md`:

```markdown
## Exercise 1 — Week 10

- **GitHub repo URL:** https://github.com/YOUR_USERNAME/YOUR_REPO
- **screenshot-prometheus-targets.png** — Prometheus Status → Targets showing tasklineapp UP
- **screenshot-promql-queries.png** — Grafana Explore showing PromQL query results
- **screenshot-grafana-dashboard.png** — four-panel Grafana dashboard with data
- **screenshot-aks-metrics.png** — Grafana on AKS showing taskline_http_requests_total results
```

---

## Hints

- `minikube service tasklineapp-service --url` blocks the terminal — leave it running in Terminal 1. Use Terminal 2 for all curl commands.
- If `/metrics` returns HTML, the old image is running. Rebuild with `eval $(minikube docker-env)` first.
- If Prometheus target shows DOWN, check the ServiceMonitor label `release: monitoring` matches the Helm release name.
- If charts show no data, generate more traffic first — `rate()` over a 5-minute window needs at least a few minutes of data.
- Use port 3001 for local Grafana and 3002 for AKS Grafana so they do not clash.
