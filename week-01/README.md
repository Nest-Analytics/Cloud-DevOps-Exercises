## Exercise (Do This Yourself)

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
