## Assessment Rubric

| Criterion | Pass | Distinction |
|---|---|---|
| **Local deployment** | All three components running on minikube. App accessible in browser. Backend connected to database. | PersistentVolumeClaim confirmed working. Frontend, backend, and database correctly networked end to end. |
| **Local monitoring** | Prometheus and Grafana installed. ServiceMonitor applied. At least one PromQL query returning data. | Four-panel Grafana dashboard with meaningful data. At least one business metric tracked beyond HTTP counters. |
| **Secrets management** | No hardcoded values anywhere. Kubernetes Secrets used for all credentials. `terraform.tfvars` in `.gitignore`. | Key Vault used in the cloud path as the durable source of truth. All secrets rotatable without touching any committed file. |
| **Infrastructure as code** | All required resources provisioned by Terraform. No hardcoded values in `.tf` files. Empty backend block. | Terraform state stored remotely. `outputs.tf` exposes key values. `apply` runs cleanly on re-run. |
| **CI/CD pipeline** | Three-job pipeline triggered by push to `main`. Images tagged with commit SHA. No hardcoded values in YAML. | Both images handled correctly. Kubernetes secret creation is idempotent. Rollout status confirmed before success. |
| **Cloud deployment** | App live at a public URL. Database connected. At least one successful pipeline run with all three jobs green. | App recovers correctly after a pod restart. Zero-downtime rollout demonstrated. |
| **Cloud monitoring** | Container Insights enabled. Three KQL queries screenshotted. Application Insights connected to backend. | KQL timechart showing request rate over time. App Insights showing request traces and at least one exception. |
| **Reflection** | Covers all three required sections. Minimum 400 words. Honest about what did not work. | Explains why decisions were made, not just what was done. Proposes specific improvements with justification. |

---

## Presentation

You will present your project to the class for **15 minutes**. Structure your presentation as follows:

1. **What you built** (2 minutes) - a brief walkthrough of the running app
2. **Architecture** (4 minutes) - show your deployment path, infrastructure, and pipeline. Explain why you made the key decisions you made.
3. **Live demo** (5 minutes) - trigger a pipeline run or show the running app, monitoring dashboard, and at least one KQL query returning data
4. **What you would do differently** (4 minutes) - be honest about what broke and what you would change with more time

Slides are optional. The live demo is not.

---

## Submission

Submit a single GitHub repository containing all your work. The repository must be accessible to your assessor.

| What | Details |
|---|---|
| GitHub repository URL | Public or shared with assessor |
| Live URL | Cloud path, must be accessible at time of assessment |
| Actions run URL | The successful run that produced the live cloud deployment |
| `REFLECTION.md` | In the repository root |

### Three Required Sections

1. **Architecture decisions** - why you chose your cloud path and how you structured the pipeline
2. **What did not work** - what broke, what took the most time to fix, be specific
3. **What you would improve** - name the specific thing you would change and why

Minimum 400 words.

> **Deadline:** Submit before the end of Week 2. Late submissions accepted up to 48 hours after the deadline with a Pass ceiling. Distinction is not available for late work.
