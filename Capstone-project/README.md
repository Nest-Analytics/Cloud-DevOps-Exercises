# Capstone Project

---

## Overview

This capstone brings together everything from the course. You are provided with a full-stack application: a frontend, a backend, and a Dockerfile for each. Your task is to deploy it, secure it, and observe it using the tools and patterns you have learned.

You do not modify the application code. Your work is entirely in the infrastructure, pipeline, manifests, monitoring, and secrets management.

**Path 1 is required for everyone. For the cloud path, choose either Path 2 or Path 3.**

> **No hardcoding.** GitHub Secrets for sensitive values, GitHub Variables for non-sensitive config, `TF_VAR_*` for Terraform variables, `-backend-config` flags for backend connection details. No exceptions.

---

## Path 1 (Required): Full Stack on Local Kubernetes

Deploy the complete application locally on minikube. All three components, frontend, backend, and PostgreSQL, must run as Kubernetes workloads. The backend connects to PostgreSQL via Kubernetes Secrets. Prometheus and Grafana must be installed and scraping the backend.

### Requirements

- Frontend, backend, and PostgreSQL all running on minikube
- PostgreSQL uses a PersistentVolumeClaim so data survives pod restarts
- All credentials stored in Kubernetes Secrets, nothing hardcoded in manifests
- Backend successfully connecting to and reading from PostgreSQL
- Frontend accessible in the browser and communicating with the backend
- Prometheus and Grafana installed via Helm in the `monitoring` namespace
- ServiceMonitor configured for the backend
- Grafana dashboard with at least four panels including one business metric

---

## Path 2 (Choose this or Path 3): Full Stack on AKS

Deploy the entire application on AKS. Frontend and backend run as Kubernetes workloads. PostgreSQL is replaced by Azure Database for PostgreSQL, provisioned by Terraform. Secrets flow from GitHub Secrets through the pipeline into Kubernetes Secrets in the cluster. Everything is automated via GitHub Actions, no manual deployments after initial setup.

### Requirements

- All infrastructure provisioned by Terraform: AKS, Azure Database for PostgreSQL, Key Vault, Log Analytics workspace, Application Insights
- Three-job GitHub Actions pipeline: build and push, terraform, deploy
- Both frontend and backend images built with commit SHA tags and pushed to your chosen registry
- Kubernetes Secrets created by the pipeline from GitHub Secrets, not manually
- Backend connected to Azure Database for PostgreSQL with credentials from Key Vault
- Frontend and backend live at a public URL
- Azure Monitor Container Insights enabled on the AKS cluster
- At least three KQL queries run in Log Analytics showing backend logs
- Application Insights connected to the backend with request telemetry visible

---

## Path 3 (Choose this or Path 2): Frontend on Azure Static Web Apps, Backend on AKS

The frontend is deployed to Azure Static Web Apps, a CDN-backed static hosting service with no container required. The backend runs on AKS. Both deployments are fully automated via the same GitHub Actions pipeline.

### Requirements

- All infrastructure provisioned by Terraform: Azure Static Web App, AKS, Azure Database for PostgreSQL, Key Vault, Log Analytics workspace, Application Insights
- Three-job GitHub Actions pipeline covering both the frontend static deployment and the backend container deployment
- Frontend build output deployed to Static Web Apps, not containerised in this path
- Backend image built with commit SHA tag and pushed to your chosen registry
- Frontend accessible at the Static Web App URL and successfully communicating with the backend
- Backend connected to Azure Database for PostgreSQL with credentials from Key Vault
- Azure Monitor Container Insights enabled on the AKS cluster
- At least three KQL queries run in Log Analytics showing backend logs
- Application Insights connected to the backend with request telemetry visible

---

## Rules

These apply to all paths.

- No credentials, passwords, or connection strings in any committed file. **Instant fail if found.**
- No resource names or registry paths hardcoded in pipeline YAML. Use GitHub Variables.
- Terraform backend block must be empty. Connection details go in `-backend-config` flags sourced from GitHub Secrets.
- Images tagged with git commit SHA, not `latest`
- Images built with `--platform linux/amd64`
- All deployments triggered by a push to `main`, no manual `kubectl` or `az` commands
- Terraform state stored remotely in Azure Blob Storage
- `terraform.tfvars` and `.env` in `.gitignore`

> **Registry:** You may use GHCR, ACR, or Docker Hub. Declare your choice in `REFLECTION.md` and be consistent.

---

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

### REFLECTION.md: Three Required Sections

1. **Architecture decisions** - why you chose your cloud path and how you structured the pipeline
2. **What did not work** - what broke, what took the most time to fix, be specific
3. **What you would improve** - name the specific thing you would change and why

Minimum 400 words.

> **Deadline:** Submit before the end of Week 2. Late submissions accepted up to 48 hours after the deadline with a Pass ceiling. Distinction is not available for late work.
