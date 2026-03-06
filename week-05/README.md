
# Exercise — CI/CD Pipelines with GitHub Actions & Azure

## Overview

You are given a Node.js web app and three incomplete GitHub Actions pipeline files. Your job is to complete each pipeline so that it automatically tests and deploys the app to Azure when you push to `main`.

Each pipeline uses a different deployment method. You are expected to read, understand, and complete the YAML — not just copy answers. The comments in each file explain what every block does and why.

You are free to use any resource: the GitHub Actions docs, Azure docs, lecture notes, the internet. The goal is that by the end, you can read a pipeline file and explain what is happening at each stage.

---

## Setup

### 1. Fork and clone this repository

```bash
git clone https://github.com/YOUR_INSTRUCTOR/week5-cicd-exercise.git
cd week5-cicd-exercise
npm install
npm test   # make sure the tests pass locally before you start
```

### 2. Create the Azure resources you need

You will need the following Azure resources. Use the same ones from Week 4 where possible.

| Resource | Used in |
|---|---|
| Azure Web App (App Service, Node 20, Linux) | Challenge 1 & 2 |
| Azure AD App Registration (for OIDC) | Challenge 2 |
| Azure Virtual Machine (Ubuntu 22.04) | Challenge 3 |

---

## The Three Challenges

The pipeline files are in `.github/workflows/`. Open each one and read it fully before you start filling it in.

---

### Challenge 1 — Deploy using a Publish Profile
**File:** `.github/workflows/1-deploy-publish-profile.yml`

The most straightforward deployment method. Azure issues you a credential file (the publish profile) that contains a username and password. You store it as a GitHub Secret and the pipeline uses it to authenticate.

**Secrets to create** in your repo (Settings → Secrets and variables → Actions):

| Secret name | What it is | How to get it |
|---|---|---|
| `AZURE_WEBAPP_NAME` | The name of your Azure Web App | Azure Portal → App Services → your app → the name shown at the top |
| `AZURE_WEBAPP_PUBLISH_PROFILE` | The full XML content of your publish profile | Azure Portal → App Services → your app → Overview → **Get publish profile** → open the downloaded file → copy all the text |

**What to complete:** trigger branch, runner, checkout action, Node version, install command, test command, publish-profile secret reference.

---

### Challenge 2 — Deploy using OIDC (Workload Identity Federation)
**File:** `.github/workflows/2-deploy-oidc.yml`

The modern approach. Instead of storing a long-lived credential, you configure a trust relationship between GitHub and Azure. GitHub requests a short-lived token for each pipeline run — no static secret is ever stored, so there is nothing to rotate or leak.

**Secrets to create:**

| Secret name | What it is | How to get it |
|---|---|---|
| `AZURE_CLIENT_ID` | The client ID of your Azure AD App Registration | Azure Portal → Microsoft Entra ID → App registrations → your app → Overview → **Application (client) ID** |
| `AZURE_TENANT_ID` | Your Azure AD tenant ID | Azure Portal → Microsoft Entra ID → Overview → **Tenant ID** |
| `AZURE_SUBSCRIPTION_ID` | Your Azure subscription ID | Azure Portal → Subscriptions → your subscription → **Subscription ID** |
| `AZURE_WEBAPP_NAME` | Same Web App as Challenge 1 | Same as above |

**One-time Azure setup you need to do first:**
1. Create an App Registration in Microsoft Entra ID (Azure AD)
2. On the App Registration, go to **Certificates & secrets → Federated credentials → Add credential**
3. Choose scenario: *GitHub Actions deploying Azure resources*
4. Enter your GitHub username/org and repo name, set Entity Type to **Branch**, value `main`
5. Assign the App Registration the **Contributor** role on your Web App (or subscription):
   App Service → Access control (IAM) → Add role assignment → Contributor → select your App Registration

**What to complete:** trigger, runner, checkout, Azure login credential references, Node version, install, test, app-name reference.

---

### Challenge 3 — Deploy to a Virtual Machine using PM2
**File:** `.github/workflows/3-deploy-vm-pm2.yml`

Azure App Service manages the server for you. A VM does not — you are responsible for the OS, Node, the process manager, and keeping the app running. This challenge teaches you how pipelines work when you control the infrastructure.

PM2 is a Node.js process manager. It keeps your app running in the background, restarts it on crashes, and survives server reboots.

**One-time setup on the VM** — SSH in and run these commands before the pipeline runs:

```bash
# Install Node.js 20
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt-get install -y nodejs

# Install PM2 globally
sudo npm install -g pm2

# Clone the repository
git clone https://github.com/YOUR_ORG/YOUR_REPO.git /home/azureuser/cicd-demo-app
cd /home/azureuser/cicd-demo-app
npm install

# Start the app with PM2
pm2 start src/index.js --name cicd-demo-app
pm2 save

# Configure PM2 to start on boot (run the command it prints)
pm2 startup
```

**Secrets to create:**

| Secret name | What it is | How to get it |
|---|---|---|
| `VM_HOST` | Public IP address of your Azure VM | Azure Portal → Virtual Machines → your VM → Overview → **Public IP address** |
| `VM_USERNAME` | Admin username of the VM | The username you set when creating the VM (default: `azureuser`) |
| `VM_SSH_PRIVATE_KEY` | Private SSH key to authenticate with the VM | If you downloaded a `.pem` during VM creation: open the file, copy all contents. If generating fresh: run `ssh-keygen -t rsa -b 4096 -f ~/.ssh/vm_key`, then upload `vm_key.pub` to the VM via Azure Portal → VM → Reset password (Public key option), and use `vm_key` (private) as the secret value |
| `VM_APP_DIR` | Full path to the app on the VM | The path you cloned into, e.g. `/home/azureuser/cicd-demo-app` |
| `PM2_APP_NAME` | The name you gave PM2 when starting the app | The `--name` value from your `pm2 start` command, e.g. `cicd-demo-app` |

**What to complete:** trigger, runner for both jobs, checkout, Node version, install and test commands, `needs:` field, SSH action credential references, script commands (cd, git pull, npm install, pm2 restart).

---

## Making a Change to Test Your Pipelines

Once a pipeline is working, prove it by making a visible change:

1. Open `src/index.js`
2. Change the `<h1>` text to something unique — your name, a message, anything
3. Commit and push to `main`
4. Watch the pipeline run in the **Actions** tab
5. Visit your deployed URL and confirm the change is live

---

## Tips

- **Read the comments.** Every TODO has a hint. Every section has an explanation of what it does and why.
- **Start with Challenge 1.** It is the simplest. Once it works, Challenges 2 and 3 will make more sense.
- **Check the Actions tab first when something fails.** Click into the failed step — the logs tell you exactly what went wrong.
- **Secret names are case-sensitive.** `AZURE_WEBAPP_NAME` and `azure_webapp_name` are different. If the pipeline can't find a secret, check spelling.
- **Challenge 3 needs the VM set up first.** The pipeline will fail if the app directory doesn't exist on the VM or PM2 has never been started. Do the one-time setup before you push.
- **OIDC (Challenge 2) requires the federated credential to be configured in Azure before it will work.** The pipeline will fail with an authentication error until the trust is set up.

---

## Submission

See [ACCEPTANCE.md](./ACCEPTANCE.md) for what you need to submit.
