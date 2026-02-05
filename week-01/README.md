# Exercise Brief

# Week 1 – Foundations: Cloud Basics, Linux Terminal, First Cloud VM, Demo App Clone

## Outcome for Week 1
By the end of Week 1, you should be able to:
- explain in simple terms what “cloud” means
- open a terminal and run basic Linux commands confidently
- understand what a cloud VM is and connect to it using SSH
- clone the demo app onto a Linux VM (running it is optional in Week 1)

This week is about foundations. The goal is that by the end of this session, you are comfortable opening a terminal, you understand what a server in the cloud is, and you have logged into one and touched a real application.

---

## Part A – Cloud Basics

### What “cloud” means
The cloud is someone else’s data centre that you can rent on demand. You pay for what you use, and you can scale up or down much more easily than with physical hardware.

### Cloud service layers
1) **IaaS (Infrastructure as a Service)**  
The provider gives you virtual machines, storage, and networks. You still manage the operating system and software.  
Example: a Linux VM in the cloud.

2) **PaaS (Platform as a Service)**  
The provider gives you a managed platform to run your app without worrying about the OS.  
Example: a managed web app service for Node or .NET. You deploy code; the platform manages the OS and runtime.

3) **SaaS (Software as a Service)**  
You use an application over the internet.  
Examples: Outlook online, SharePoint online, Google Docs, Salesforce.

### What DevOps means (in this programme)
DevOps combines development (Dev) and operations (Ops). It is a process-oriented approach that unites software development and IT operations teams to shorten the development lifecycle, increase deployment frequency, and ensure high-quality, reliable releases.

In this training, we will write code and then use cloud infrastructure and services to:
- run that code,
- deploy updates reliably, and
- monitor and maintain the system.

This week you will see your first piece of that: a Linux server in the cloud and our demo application.

---

## Part B – Linux Basics (Terminal Essentials)

A big part of Cloud/DevOps work is done in a terminal. The terminal is a way of typing commands to the operating system. It looks intimidating at first, but the basics are not complicated. This week focuses on the commands you will use constantly.

### Core commands to know

**Where am I?**
- `pwd`  
Print working directory. Shows the folder you are currently in.

**What is in this folder?**
- `ls`  
List files and folders in the current directory.
- `ls -la`  
List with details (including hidden files) and permissions.

**Move around**
- `cd <folder>`  
Change directory.
- `cd /tmp`  
Go to `/tmp`.
- `cd ~`  
Go to your home directory.
- `cd ..`  
Go up one level.

**Create folders and files**
- `mkdir devops-training`  
Create a directory.
- `touch notes.txt`  
Create an empty file.

**Write and read files**
- `echo "Hello Cloud/DevOps" > notes.txt`  
Write text into a file (overwrites).
- `cat notes.txt`  
Print the file content.
- `nano notes.txt`  
Edit a file in the terminal editor.

**Who am I / what machine is this?**
- `whoami`  
Show your username.
- `hostname`  
Show the machine name.

**Permissions**
- `ls -l`  
Shows permissions in the left column.
- `chmod +x script.sh`  
Make a script executable.

**Install packages**
- `sudo apt update`  
Update package lists.
- `sudo apt install git -y`  
Install Git. `sudo` runs the command with admin privileges.

**See running processes**
- `ps aux | head`  
Show processes (first lines).
- `top`  
Live view of system activity.

---

## Mini Exercise (Do This Yourself)

Run the following commands and capture evidence for submission:

### 1) See where you are
```bash
pwd

### 2) Make a directory and go into it
mkdir week1-practice
cd week1-practice

### 3) Create a notes file
echo "My first Cloud/DevOps session" > notes.txt

### 4) List files
ls -l

### 5) View the file
cat notes.txt

Part C – First Cloud VM + Demo App
Step 1 – What a VM is

A Virtual Machine in the cloud is a virtualised server that runs in a provider’s data centre. You choose:

an OS image (example: Ubuntu),

a size (CPU/RAM),

a username,

access settings (SSH).

Step 2 – Connect to the VM with SSH

You will be given (or you will create) a VM and connect using SSH.

Example:

ssh -i ~/Downloads/file.pem user@<PUBLIC_IP_OF_VM>


Once connected, run:

whoami
hostname
pwd
ls

Step 3 – Install Git and Node.js on the VM (you demo, students follow if time allows)
sudo apt update
sudo apt install -y git curl
curl -fsSL https://deb.nodesource.com/setup_18.x | sudo -E bash -
sudo apt install -y nodejs
node -v
npm -v

Step 4 – Clone the demo app on the VM
cd ~
git clone https://github.com/<your-github-username>/cloudtasks.git
cd cloudtasks
ls
cat README.md

Step 5 – Run the demo app on the VM (demo)

Running the app is optional for students in Week 1, but you will see it demonstrated.

npm install
npm run dev
