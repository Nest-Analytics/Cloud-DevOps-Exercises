# Exercise: Host Demo App on a Web App (ZIP Deploy + GitHub CI/CD) + Configure UI via Env Vars

## Lab goal
By the end, you should:
- have your own Web App URL running the Demo App
- deploy using **ZIP deploy** and also deploy using **GitHub (CI/CD)**
- use env variables to change the app appearance without redeploying

---

## Task brief
Your task is to get the Demo App running on your own Web App in two ways:
1) ZIP deployment, and  
2) GitHub-based CI/CD deployment from your repo, and then control part of the UI from configuration.

---

## Tasks

## Part A – Create the Web App (once)

### 1) Create a Web App
Create a managed Web App using:
- the resource group you have been using in class
- **Runtime:** Node.js
- **OS:** Linux
- **Plan:** small Dev/Test plan

Naming:
- Web App name format: `demoapp-<firstname>-<unique>`
  - example: `demoapp-sanmi-01`

---

## Part B – Deployment Method 1: ZIP Deploy

### 2) Deploy the Demo App using ZIP deploy
Deploy the Demo App to your Web App using ZIP deploy.

Use either:
- the ZIP file provided in the group, or
- a ZIP you create from your fork of the Demo App

After deployment:
- visit your Web App URL
- confirm the Demo App loads without errors

---

## Part C – Configure UI with Env variables

### 3) Set `APP_TITLE`
In your Web App configuration (App settings), add:
- All the env variables in the repo

Save and reload the app.

Confirm the title changed and other things change as well **without redeploying**.

---

## Part D – Deployment Method 2: GitHub CI/CD (Deploy from GitHub)

### 6) Connect your Web App to GitHub deployment
Set up GitHub deployment from the Web App (Deploy Center / Deployment settings).

Requirements:
- Use your fork of the Demo App in GitHub.
- Configure it so that a push to your repo triggers a deployment to the Web App.

### 7) Trigger a deployment via GitHub
Make a small change in your repo and push it to GitHub, for example:
- update a visible text in the Demo App UI (a label or a footer), or
- add a short line to `README.md` if that is what your deployment is configured to build from (only do this if it affects what is deployed)

Then confirm:
- the Web App redeployed via GitHub
- your change is visible on the live Web App URL

---


### 8) Set the env as before

Reload the app and confirm the banner appears.


---

## What to submit

Submit your Week 4 work in `cloud-devops-submissions` under:

`week-04/`

Include:
- `notes.md`
- `screenshots/`

---

## Notes to include in `notes.md`

1) Your Web App URL

2) ZIP deploy confirmation (short)
- “I deployed using ZIP deploy and the app loaded successfully.”

3) `APP_TITLE` test (required)
- “When I changed `APP_TITLE` from `<X>` to `<Y>` and saved, the page title changed without redeploying.”

4) GitHub CI/CD confirmation (required)
- “I connected GitHub deployment and confirmed a push triggered a deployment.”
- “The change I pushed that proved CI/CD is working: <describe the change>”

If you did the stretch goal, add:
- details of all the env variables

---

## Screenshots required

Add these screenshots to `week-04/screenshots/`:

### Required (ZIP + Env Vars)
1) Web App configuration showing `APP_TITLE` (and `SHOW_BANNER` if used)
2) Demo App page showing the title after the first `APP_TITLE` value
3) Demo App page showing the title after changing `APP_TITLE` (no redeploy)

### Required (GitHub CI/CD)
4) Web App deployment configuration showing GitHub is connected (repo and branch visible)
5) Evidence of a GitHub-triggered deployment (deployment log, deployment status, or similar)
6) Live Demo App page showing the change that confirms the GitHub deployment worked
