# Resources
---

**GitHub Actions**
- [GitHub Actions — Quickstart](https://docs.github.com/en/actions/quickstart) — best place to start if Actions is new to you
- [Workflow syntax for GitHub Actions](https://docs.github.com/en/actions/using-workflows/workflow-syntax-for-github-actions) — full reference for every YAML key (`on`, `jobs`, `steps`, `needs`, `permissions`, etc.)
- [Encrypted secrets](https://docs.github.com/en/actions/security-guides/encrypted-secrets) — how to create and reference secrets in your workflows
- [Events that trigger workflows](https://docs.github.com/en/actions/using-workflows/events-that-trigger-workflows) — everything that can go in the `on:` block beyond just `push`

**Azure App Service**
- [Deploy to App Service using GitHub Actions](https://learn.microsoft.com/en-us/azure/app-service/deploy-github-actions) — Microsoft's own guide covering both publish profile and OIDC methods
- [Get a publish profile](https://learn.microsoft.com/en-us/visualstudio/azure/how-to-get-publish-profile-from-azure-app-service) — step by step with screenshots
- [azure/webapps-deploy action](https://github.com/Azure/webapps-deploy) — the action used in Challenges 1 and 2, README explains all the input parameters

**OIDC / Workload Identity Federation**
- [Configure OIDC with Azure](https://docs.github.com/en/actions/deployment/security-hardening-your-deployments/configuring-openid-connect-in-azure) — GitHub's guide, covers the federated credential setup end to end
- [azure/login action](https://github.com/Azure/login) — the action used in Challenge 2, explains all three authentication methods it supports
- [Workload identity federation overview](https://learn.microsoft.com/en-us/entra/workload-id/workload-identity-federation) — Microsoft's explanation of why this is more secure than static credentials

**Azure Virtual Machines**
- [Create a Linux VM in the Azure Portal](https://learn.microsoft.com/en-us/azure/virtual-machines/linux/quick-create-portal) — if you need to create your VM from scratch
- [SSH into a Linux VM](https://learn.microsoft.com/en-us/azure/virtual-machines/linux/ssh-from-windows) — connecting from Windows, Mac, or Linux
- [Reset SSH credentials on an Azure VM](https://learn.microsoft.com/en-us/troubleshoot/azure/virtual-machines/reset-ssh) — useful if you lose access or need to upload a new public key

**PM2**
- [PM2 — Getting started](https://pm2.keymetrics.io/docs/usage/quick-start/) — install, start, restart, and save commands
- [PM2 startup script](https://pm2.keymetrics.io/docs/usage/startup/) — how to make PM2 survive a VM reboot (`pm2 startup` + `pm2 save`)
- [Running npm scripts with PM2](https://pm2.keymetrics.io/docs/usage/application-declaration/) — explains the `pm2 start npm -- run dev` pattern used in Challenge 3

**SSH Action**
- [appleboy/ssh-action](https://github.com/appleboy/ssh-action) — the action used in Challenge 3, README covers all input options and common troubleshooting

---

## Concepts to Read Up On

**CI/CD fundamentals**
- [What is CI/CD?](https://www.redhat.com/en/topics/devops/what-is-ci-cd) — Red Hat's plain-English explanation, good background reading
- [GitHub Actions: Understanding the workflow file](https://docs.github.com/en/actions/learn-github-actions/understanding-github-actions) — explains runners, jobs, steps, and how they relate

**YAML**
- [Learn YAML in Y minutes](https://learnxinyminutes.com/docs/yaml/) — fast syntax reference, covers everything you need for writing pipelines
- [YAML Lint](https://www.yamllint.com/) — paste your YAML here to check for indentation errors before pushing

**Security**
- [Security hardening for GitHub Actions](https://docs.github.com/en/actions/security-guides/security-hardening-for-github-actions) — covers secrets, OIDC, and why you never hardcode credentials in YAML
- [Principle of least privilege](https://learn.microsoft.com/en-us/azure/role-based-access-control/best-practices) — why the `permissions` block in Challenge 2 only requests what it needs

---

## Troubleshooting

**Pipeline not triggering**
Make sure your file is in exactly `.github/workflows/` at the root of the repo, and that the branch name in `on: push: branches:` matches your actual branch name.

**Secret not found**
Secret names are case-sensitive. `AZURE_WEBAPP_NAME` and `azure_webapp_name` are different. Double-check the name in your YAML matches exactly what you created in Settings → Secrets.

**OIDC login failing**
The federated credential in Entra ID must match your repo name and branch exactly. Also confirm the App Registration has been assigned a role (Contributor or higher) on the Web App or subscription — without that, login succeeds but deploy will be denied.

**SSH connection refused (Challenge 3)**
Check that port 22 is open on your VM: Azure Portal → VM → Networking → Inbound port rules. Also confirm the public key on the VM matches the private key stored in your secret.

**PM2 app not restarting**
Run `pm2 list` on the VM to check the app name matches your `PM2_APP_NAME` secret exactly. If the app was never started with `pm2 start` first, the `pm2 restart` command will fail — make sure the one-time setup steps were completed.

**Deployment succeeds but site shows old version**
Azure App Service can take a minute or two to swap in the new code. Hard refresh the browser (`Ctrl + Shift + R`) and wait 60 seconds before assuming something is wrong.
