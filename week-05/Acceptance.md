# Acceptance Criteria

Submit your as usual in the Submission repo. The repository must contain your completed pipeline files and evidence of successful runs.

---

## What to Submit

### 1. Your completed repository (GitHub link)

The repo must contain all three completed pipeline files with no remaining `TODO` placeholders:
- `.github/workflows/1-deploy-publish-profile.yml`
- `.github/workflows/2-deploy-oidc.yml`
- `.github/workflows/3-deploy-vm-pm2.yml`

---

### 2. Screenshots — one per challenge

For each challenge, take a screenshot of the **GitHub Actions run** showing all steps passing (every step should show a green checkmark). The screenshot must show:

- The name of the workflow
- All steps expanded or at least visible
- A green tick on the final Deploy step

**Challenge 1 screenshot** — publish profile deploy run
**Challenge 2 screenshot** — OIDC deploy run
**Challenge 3 screenshot** — VM + PM2 deploy run

---

### 3. Screenshots — live deployed app

For each challenge, take a screenshot of your deployed app in the browser. The browser address bar must be visible so the URL is shown.

**Challenge 1** — Azure Web App URL (publish profile deploy)
**Challenge 2** — Azure Web App URL (OIDC deploy, same app is fine)
**Challenge 3** — VM public IP or domain, port 3000 (e.g. `http://20.x.x.x:3000`)

---

### 4. Written answers (add a file called `ANSWERS.md` to your repo)

Answer each question in your own words. A sentence or two is enough — this is not an essay. The purpose is to check you understand what you built, not to test your writing.

**Question 1**
Look at your Challenge 1 pipeline. If the `npm test` step fails, what happens to the Deploy step? Why?

**Question 2**
What is the difference between how Challenge 1 and Challenge 2 authenticate with Azure? Why is the OIDC approach considered more secure?

**Question 3**
In Challenge 3, why do we use PM2 instead of just running `npm run dev` directly on the VM?

**Question 4**
Look at the `permissions` block in Challenge 2. What do `id-token: write` and `contents: read` allow, and why would the pipeline fail without the `id-token: write` permission?

---

## Checklist Before You Submit

- [ ] All three `.yml` files have no remaining `TODO` placeholders
- [ ] All three pipelines have at least one successful run in the Actions tab
- [ ] 3 Actions screenshots included (or linked) in your repo
- [ ] 3 live app screenshots included (or linked) in your repo
- [ ] `ANSWERS.md` file exists with all 4 questions answered
- [ ] Repository is set to public (or you have added your instructor as a collaborator)
