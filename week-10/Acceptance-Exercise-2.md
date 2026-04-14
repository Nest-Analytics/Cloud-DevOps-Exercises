# Acceptance-Exercise-2 | Monitoring: Container Insights & Log Analytics

---

### Required Screenshots

| File | What it must show |
|---|---|
| `screenshot-container-insights.png` | AKS Portal → Monitoring → Insights → Containers tab showing the `tasklineapp` container with CPU and memory data |
| `screenshot-live-logs.png` | AKS Portal → Monitoring → Insights → Live Logs showing JSON log lines from the app |
| `screenshot-kql-all-logs.png` | Log Analytics query editor showing `ContainerLog` query returning rows with `tasklineapp` logs |
| `screenshot-kql-errors.png` | KQL query filtering to `"level":50` returning at least one error log row |
| `screenshot-kql-timechart.png` | Request count timechart rendered as a chart (not a table) |

---

### Part 1 — Container Insights — Must Pass All

- [ ] `azurerm_log_analytics_workspace` resource is present in `terraform/main.tf`
- [ ] `oms_agent` block is present inside `azurerm_kubernetes_cluster` in `terraform/main.tf`
- [ ] AKS Portal → Monitoring → Insights is enabled (not showing "Monitoring not enabled")
- [ ] Containers tab screenshot shows `tasklineapp` container — not blank or showing only system containers

---

### Part 2 — Log Analytics — Must Pass All

- [ ] At least three KQL queries run and returning results:
  - [ ] `ContainerLog` general query returning `tasklineapp` log rows
  - [ ] Error-level filter query (`"level":50`) returning at least one row
  - [ ] Request count timechart rendered as a chart
- [ ] Log entries visible in query results are JSON structured logs — not plain text
- [ ] Timechart screenshot shows the chart rendered, not an empty chart or table view

---

### `SUBMISSION.md` — Must Contain

- [ ] GitHub repo URL
- [ ] All five screenshot filenames referenced under the correct part

---

## Not Accepted If

- Container Insights tab shows "Monitoring not enabled" — `oms_agent` is missing from Terraform or not applied
- KQL queries return zero rows — no traffic was generated or ingestion hasn't completed (wait 5 minutes and retry)
- Timechart screenshot shows table view instead of chart — click `render timechart` or select the chart tab in the query editor
- Log entries in screenshots are plain text, not JSON — the app image may be outdated

---

## Marking Notes

Pass / not yet. Part 1 and Part 2 are both required. A submission with only one part is returned for completion.
