# Week 1 Exercise: Linux Basics + First Cloud VM + Demo App

This exercise is split into two parts:

* **Part A (Local terminal):** get comfortable with basic Linux commands
* **Part B (Cloud VM):** connect to a cloud VM with SSH and clone a demo app

---

## What to Submit (Evidence)

Capture **screenshots or terminal output** showing:

### Part A (Local terminal)

* `pwd`
* `ls -l` showing `notes.txt`
* `cat notes.txt` showing the exact content

### Part B (Cloud VM)

* Your **successful SSH connection**
* Output of:

  * `whoami`
  * `hostname`
  * `pwd`
  * `ls`
* Proof Node and npm are installed:

  * `node -v`
  * `npm -v`
* Proof repo was cloned:

  * `ls` inside the repo
  * `cat README.md`

---

## Part A: Linux Basics (Do This Yourself)

Run the following commands in your terminal and capture evidence.

### 1) See where you are

```bash
pwd
```

### 2) Make a directory and go into it

```bash
mkdir week1-practice
cd week1-practice
```

### 3) Create a notes file

```bash
echo "My first Cloud/DevOps session" > notes.txt
```

### 4) List files

```bash
ls -l
```

### 5) View the file

```bash
cat notes.txt
```

---

## Part C: First Cloud VM + Demo App

### Step 1: What a VM is

A **Virtual Machine (VM)** in the cloud is a virtualised server running in a provider’s data centre. When you create one, you typically choose:

* **OS image** (example: Ubuntu)
* **Size** (CPU/RAM)
* **Username**
* **Access settings** (SSH)

---

### Step 2: Connect to the VM with SSH

You will be given (or you will create) a VM and connect using SSH.

Example:

```bash
ssh -i ~/Downloads/file.pem user@<PUBLIC_IP_OF_VM>
```

Once connected, run:

```bash
whoami
hostname
pwd
ls
```

---

### Step 3: Install Git and Node.js on the VM (Instructor demo, students follow if time allows)

```bash
sudo apt update
sudo apt install -y git curl
curl -fsSL https://deb.nodesource.com/setup_18.x | sudo -E bash -
sudo apt install -y nodejs
node -v
npm -v
```

---

### Step 4: Clone the demo app on the VM

```bash
cd ~
git clone https://github.com/<your-github-username>/cloudtasks.git
cd cloudtasks
ls
cat README.md
```

---

### Step 5: Run the demo app on the VM (demo)

Running the app is optional for students in Week 1, but it will be demonstrated.

```bash
npm install
npm run dev
```

---

## Notes / Common Fixes

* **Permission error on key file (.pem):**

  ```bash
  chmod 400 ~/Downloads/file.pem
  ```
* **SSH connection times out:** check VM is running and inbound rules allow **port 22**.
* **Wrong username:** Ubuntu images often use `ubuntu` as the default user.

---

## Completion Checklist

* [ ] Completed Part A commands locally and captured evidence
* [ ] SSH connected into the VM and captured evidence
* [ ] Installed Git + Node.js and captured `node -v` and `npm -v`
* [ ] Cloned repo and captured `ls` and `cat README.md`
