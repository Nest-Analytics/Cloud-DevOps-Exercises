# Exercise 2 — Azure Monitoring | Monitoring: Container Insights & Log Analytics

---

## Overview

This exercise connects the Taskline app running on AKS to Azure's cloud monitoring stack. You will use two services:

- **Part 1 — Container Insights**: cluster, node, and pod observability configured through Terraform
- **Part 2 — Log Analytics**: query your structured JSON logs using KQL (Kusto Query Language)

By the end, you will have live metrics and logs flowing into Azure Monitor and be able to query them.

---

## Prerequisites

- Exercise 1 completed — app running on AKS
- Terraform from Week 8/9 updated to include Log Analytics workspace and `oms_agent`
- App deployed and generating traffic

---

## Part 1 — Container Insights

### Step 1 — Confirm Terraform includes Log Analytics and oms_agent

Your `terraform/main.tf` should already have these resources from the Week 9 Terraform configuration. Confirm they are present:

```hcl
resource "azurerm_log_analytics_workspace" "main" {
  name                = "law-tasklineapp"
  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name
  sku                 = "PerGB2018"
  retention_in_days   = 30
}
```

And inside `azurerm_kubernetes_cluster`:

```hcl
oms_agent {
  log_analytics_workspace_id      = azurerm_log_analytics_workspace.main.id
  msi_auth_for_monitoring_enabled = true
}
```

If these are not present, add them and run `terraform apply` via the pipeline or locally.

---

### Step 2 — Verify in the Azure Portal

1. Go to the [Azure Portal](https://portal.azure.com)
2. Navigate to your AKS cluster → **Monitoring → Insights**
3. You should see:
   - **Cluster** tab: node count, CPU and memory utilisation
   - **Nodes** tab: per-node resource usage
   - **Containers** tab: per-container CPU, memory, and restart count
   - **Live Logs** tab: real-time stdout from running containers

If Insights shows "Monitoring not enabled", the `oms_agent` block is missing or the Terraform change has not been applied yet.

---

### Step 3 — Generate Traffic and Watch Container Insights

Generate some traffic against the AKS app:

```bash
# Get the external IP
kubectl get svc tasklineapp-service

AKS_URL=http://EXTERNAL_IP

curl -X POST "$AKS_URL/api/demo/tasks" \
  -H 'Content-Type: application/json' \
  -d '{"title":"Monitor this"}'

curl -X POST "$AKS_URL/login" \
  -H 'Content-Type: application/json' \
  -d '{"username":"ayo","password":"wrong"}'

curl "$AKS_URL/api/demo/error"
```

Go back to the Portal → AKS → **Monitoring → Insights → Containers** tab. Find the `tasklineapp` container. You should see CPU and memory updating. Click **Live Logs** to see stdout from the container in real time.

Take a screenshot of:
- The Containers tab showing the `tasklineapp` container with CPU/memory data
- The Live Logs view showing JSON log lines from the app

---

## Part 2 — Log Analytics: KQL Queries

### Step 1 — Open Log Analytics

In the Portal: navigate to your AKS cluster → **Monitoring → Logs**

This opens the Log Analytics query editor scoped to your cluster. You query logs using **KQL (Kusto Query Language)** — it reads left to right, pipe-separated, similar in concept to LogQL from Grafana/Loki.

---

### Step 2 — Run These Queries

Run each query and take a screenshot of the results.

**All container logs from the last hour:**

```kql
ContainerLog
| where TimeGenerated > ago(1h)
| where ContainerName contains "tasklineapp"
| project TimeGenerated, ContainerName, LogEntry
| order by TimeGenerated desc
```

**Only error-level logs (pino level 50 = error):**

```kql
ContainerLog
| where TimeGenerated > ago(1h)
| where ContainerName contains "tasklineapp"
| where LogEntry contains '"level":50'
| project TimeGenerated, LogEntry
| order by TimeGenerated desc
```

**Only warning-level logs (pino level 40 = warn):**

```kql
ContainerLog
| where TimeGenerated > ago(1h)
| where ContainerName contains "tasklineapp"
| where LogEntry contains '"level":40'
| project TimeGenerated, LogEntry
```

**Request count per minute — renders as a time chart:**

```kql
ContainerLog
| where ContainerName contains "tasklineapp"
| where LogEntry contains "request completed"
| summarize RequestCount = count() by bin(TimeGenerated, 1m)
| render timechart
```

**Task creation events:**

```kql
ContainerLog
| where ContainerName contains "tasklineapp"
| where LogEntry contains "task created"
| project TimeGenerated, LogEntry
| order by TimeGenerated desc
```

> If queries return no results, logs may still be ingesting — Log Analytics can take 3–5 minutes after traffic is generated. Generate more traffic and wait, then re-run.

---

### Step 3 — Why This Works

The queries above parse your structured pino JSON logs directly. Because the app emits `"level":50` for errors and `"msg":"task created"` for business events, KQL can filter on those values without any additional instrumentation. This is why structured logging matters — unstructured text like `Error: something failed` cannot be reliably filtered at scale.

---

### Step 4 — Save a Query (optional but encouraged)

After running the request count timechart, click **Save → Save as query**. Give it a name like `Taskline request rate`. This makes it available as a saved query you can share with the team.

---

## Submitting Your Work

See `Acceptance-Exercise-2.md` for what to include.

Add an Exercise 2 section to `SUBMISSION.md`:

```markdown
## Exercise 2 — Week 9

- **GitHub repo URL:** https://github.com/YOUR_USERNAME/YOUR_REPO

### Part 1 — Container Insights
- **screenshot-container-insights.png** — AKS Containers tab showing tasklineapp with CPU/memory
- **screenshot-live-logs.png** — Live Logs view showing JSON log lines

### Part 2 — Log Analytics
- **screenshot-kql-all-logs.png** — ContainerLog query returning results
- **screenshot-kql-errors.png** — Error-level log query results
- **screenshot-kql-timechart.png** — Request count timechart rendered
```

---

## Hints

- If Container Insights shows "Monitoring not enabled", check that `oms_agent` is in your Terraform AKS resource and that `terraform apply` has been run.
- Log Analytics ingestion takes 3–5 minutes. If queries return nothing immediately, generate more traffic, wait a few minutes, and retry.
- KQL is case-sensitive for string comparisons. `"level":50` must match exactly what pino writes — check a raw log line if queries return nothing.
- The request count timechart is the most visually impressive query — use it for the screenshot.
- Container Insights and Log Analytics are separate views but both use the same Log Analytics workspace as the backend.
