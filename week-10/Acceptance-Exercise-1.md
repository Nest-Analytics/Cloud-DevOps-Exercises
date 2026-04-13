# Acceptance-Exercise-1.md | Monitoring: Prometheus & Grafana

---

### Required Screenshots

| File | What it must show |
|---|---|
| `screenshot-prometheus-targets.png` | Prometheus Status → Targets with `tasklineapp` showing state `UP` |
| `screenshot-promql-queries.png` | Grafana Explore with at least one PromQL query returning data (e.g. `sum(taskline_http_requests_total) by (path, status_code)`) |
| `screenshot-grafana-dashboard.png` | Four-panel Grafana dashboard with all panels showing data — not empty or "No data" |
| `screenshot-aks-metrics.png` | Grafana connected to the AKS cluster showing `taskline_http_requests_total` returning results |

---

### Prometheus — Must Pass All

- [ ] kube-prometheus-stack installed in the `monitoring` namespace
- [ ] All monitoring pods in `Running` state (`kubectl get pods -n monitoring`)
- [ ] ServiceMonitor applied — `kubectl get servicemonitor -n monitoring` shows `tasklineapp-monitor`
- [ ] Prometheus target for `tasklineapp` shows `UP` in the Prometheus UI

---

### PromQL Queries — Must Pass All

- [ ] At least two queries run in Grafana Explore returning data:
  - [ ] `sum(taskline_http_requests_total) by (path, status_code)`
  - [ ] `histogram_quantile(0.95, sum(rate(taskline_http_request_duration_seconds_bucket[5m])) by (le, path))`

---

### Grafana Dashboard — Must Pass All

- [ ] Dashboard saved with all four panels:
  - [ ] Panel 1: requests per second — `sum(rate(taskline_http_requests_total[5m])) by (path)`
  - [ ] Panel 2: p95 latency — `histogram_quantile(0.95, ...)`
  - [ ] Panel 3: error rate — `sum(rate(taskline_errors_total[5m])) by (path, type)`
  - [ ] Panel 4: tasks created — `increase(taskline_tasks_created_total[15m])`
- [ ] All four panels show data — not "No data"

---

### AKS — Must Pass All

- [ ] kubectl context switched to AKS cluster
- [ ] kube-prometheus-stack installed on AKS
- [ ] ServiceMonitor applied to AKS cluster
- [ ] Grafana on AKS shows `taskline_http_requests_total` returning results

---

### `SUBMISSION.md` — Must Contain

- [ ] GitHub repo URL
- [ ] All four screenshot filenames referenced

---

## Not Accepted If

- Prometheus target shows `DOWN` — ServiceMonitor is not correctly configured
- Grafana dashboard panels show "No data" — no traffic was generated or scraping is not working
- AKS screenshot shows minikube data instead of AKS cluster data
- `/metrics` endpoint returns HTML — old image, not rebuilt after code changes

---

## Marking Notes

Pass / not yet. Exercise 1 is a prerequisite for Exercise 2.
