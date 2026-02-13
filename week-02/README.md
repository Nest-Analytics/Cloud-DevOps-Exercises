# Week 2 – Student Assessment (Linux, PM2, Git)

Your Week 2 practice is split into three parts: **Linux**, **PM2**, and **Git**. Keep it realistic and simple. Your goal is to build confidence with the terminal, basic process/service checks, and a standard workflow for running and managing an app.

---

## What you must submit

Submit your work in `cloud-devops-submissions` under:

`week-02/`

Include:
- `notes.md` (required)
- `commands.txt` (required)
- `screenshots/` (required)

Suggested structure:
```text
week-02/
  notes.md
  commands.txt
  screenshots/
````

---

## Part 1 – Linux: Processes, Services, Logs (on your VM)

Run the commands below on your VM and capture evidence.

### 1) Processes

Run:

```bash
ps aux | head
ps aux | grep ssh
top
```

### 2) SSH service status

Run:

```bash
systemctl status ssh
```

### 3) SSH logs (last 30 minutes)

Run:

```bash
sudo journalctl -u ssh --since "30 minutes ago"
```

### 4) Write-up (add to `notes.md`)

Write down:

* **One thing you noticed** in the `systemctl status ssh` output
  (example: service is active/running, uptime, PID, warnings, restart count)
* **One thing you observed** from the `journalctl` output
  (example: successful logins, failed attempts, IP addresses, timestamps, key authentication messages)

---

## Part 2 – Script + PM2 (on your VM)

Your goal is to start the app using a script, then manage it with PM2.

### 1) Create a start script

From your VM, create this file:

`/opt/Velo-learn/start_velo_learn.sh`

Use the same content you used in class (adjust paths if needed). Example template:

```bash
#!/usr/bin/env bash
cd /opt/Velo-learn
npm install
npm run dev
```

### 2) Make it executable and run it once

```bash
chmod +x /opt/Velo-learn/start_velo_learn.sh
/opt/Velo-learn/start_velo_learn.sh
```

Stop the app after confirming it starts (Ctrl + C), then continue.

### 3) Install PM2

```bash
sudo npm install -g pm2
```

### 4) Start the app with PM2

Use this standard command:

```bash
cd /opt/Velo-learn
pm2 start npm --name velo-learn -- run dev
```

Then verify:

```bash
pm2 list
pm2 logs velo-learn
```

### 5) Configure PM2 to start on boot

Run:

```bash
pm2 startup
```

Copy and run the command PM2 prints out (it will be a `sudo ...` command).

Then run:

```bash
pm2 save
```

### Optional challenge

Reboot your VM and confirm the app is still managed by PM2:

```bash
pm2 list
```

---

## Part 3 – Git Basics (Fork + Branch + PR)

You can do this on your laptop or on the VM.

### 1) Fork the repository

Fork the `Velo-learn` repository into your own GitHub account.

### 2) Clone your fork

```bash
git clone https://github.com/<your-username>/Velo-learn.git
cd Velo-learn
```

### 3) Create a feature branch

```bash
git checkout -b feature/add-name-to-readme
```

### 4) Edit README.md

Add your name under a section named:

* `Contributors` or
* `Training student`

Example:

```md
## Training student
- Your Name
```

### 5) Commit and push

```bash
git add README.md
git commit -m "Add my name to README"
git push -u origin feature/add-name-to-readme
```

### 6) Open a Pull Request and merge it

Open a Pull Request from:
`feature/add-name-to-readme` → `main` (in your own fork)

Merge it after checks pass.

---

## Evidence required (screenshots)

Add these screenshots into `week-02/screenshots/`:

1. Output showing:

```bash
ps aux | grep ssh
```

2. Output showing:

```bash
pm2 list
```

3. Proof that your Pull Request was merged in your own GitHub repo
   (screenshot of the merged PR page)

---

## Notes and command log requirements

### `commands.txt`

Paste the commands you ran (in order). Do not paste passwords or private keys.

### `notes.md`

Include:

* your Part 1 observations (systemctl + journalctl)
* confirmation that PM2 is running the app
* confirmation that you created a PR and merged it in your fork
