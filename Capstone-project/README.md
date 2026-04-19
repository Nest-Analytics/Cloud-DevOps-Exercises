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

> **Registry:** You may use GHCR, ACR, or Docker Hub. Declare your choice and be consistent.
