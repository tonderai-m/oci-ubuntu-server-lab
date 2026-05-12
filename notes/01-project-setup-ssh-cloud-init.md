# 01 Project Setup, SSH Key, and Cloud-Init

## Goal

Prepare the local project structure before creating cloud resources in Oracle Cloud Infrastructure.

This note covers:

```text
GitHub repository setup
Local project folder setup
Secret protection
Dedicated SSH key generation
Cloud-init file creation
Pre-networking checklist
```

Networking setup belongs in:

```text
notes/02-networking-vcn-subnet-gateway.md
```

Compute instance creation belongs in:

```text
notes/03-compute-instance-first-login.md
```

Troubleshooting belongs in:

```text
notes/04-troubleshooting.md
```

---

# 1. Project Purpose

Project name:

```text
oci-ubuntu-server-lab
```

Purpose:

```text
Build a headless Ubuntu server on Oracle Cloud Infrastructure for learning Linux, SSH, Docker, PostgreSQL, Python, Git, Nginx, and data engineering infrastructure.
```

This project is for learning:

```text
Cloud VM creation
Linux server administration
SSH key authentication
Public/private networking
VCNs and subnets
Security rules and ports
Cloud-init automation
Docker installation
Nginx basics
PostgreSQL client tools
Git-based infrastructure documentation
```

---

# 2. Recommended Repository Structure

Use this structure:

```text
oci-ubuntu-server-lab/
├── README.md
├── .gitignore
├── cloud-init/
│   └── cloud-init.yaml
└── notes/
    ├── 01-project-setup-ssh-cloud-init.md
    ├── 02-networking-vcn-subnet-gateway.md
    ├── 03-compute-instance-first-login.md
    └── 04-troubleshooting.md
```

---

# 3. Create the GitHub Repository

Create a new GitHub repository.

Repository name:

```text
oci-ubuntu-server-lab
```

Description:

```text
Headless Ubuntu server lab on Oracle Cloud Infrastructure for learning Linux, SSH, Docker, PostgreSQL, Python, Git, Nginx, and data engineering infrastructure.
```

Recommended visibility:

```text
Public
```

Only keep it public if the repository contains no secrets.

Initialize with:

```text
README: Yes
.gitignore: Python
License: MIT License
```

The repository should store:

```text
Documentation
Safe setup notes
Cloud-init templates
Troubleshooting notes
Example configuration files
```

The repository should not store:

```text
Private SSH keys
Oracle Cloud credentials
API keys
Passwords
.env files
Database passwords
Access tokens
Real public IPs if you do not want them visible
OCIDs or account-specific identifiers if you do not want them visible
```

---

# 4. Create the Local Project Folder

On Mac Terminal:

```bash
mkdir oci-ubuntu-server-lab
cd oci-ubuntu-server-lab
```

Create the folder structure:

```bash
mkdir cloud-init notes

touch README.md
touch .gitignore

touch cloud-init/cloud-init.yaml

touch notes/01-project-setup-ssh-cloud-init.md
touch notes/02-networking-vcn-subnet-gateway.md
touch notes/03-compute-instance-first-login.md
touch notes/04-troubleshooting.md
```

Check the structure:

```bash
find . -maxdepth 3 -type f | sort
```

Expected files:

```text
./.gitignore
./README.md
./cloud-init/cloud-init.yaml
./notes/01-project-setup-ssh-cloud-init.md
./notes/02-networking-vcn-subnet-gateway.md
./notes/03-compute-instance-first-login.md
./notes/04-troubleshooting.md
```

---

# 5. Add Secret Protection to `.gitignore`

Open `.gitignore` and add:

```gitignore
# =========================================================
# Cloud / SSH / Infrastructure secrets
# =========================================================

# SSH private keys and key files
*.pem
*.key
id_rsa
id_ed25519
oci_ubuntu_server_lab

# Public keys
# Public keys are not private secrets, but avoid committing them unless needed.
*.pub

# Environment secrets
.env
.env.*
.envrc

# Oracle Cloud credentials
.oci/
oci_config
oracle_config*
oci_api_key*
*.oci

# Terraform secret variables
*.tfvars
*.tfvars.json

# Local config files that may contain secrets
config
credentials
secrets/
*.secret
*.secrets

# Mac system files
.DS_Store

# VS Code local settings
.vscode/
```

Important rule:

```text
The repo stores documentation and safe templates.
The private SSH key stays outside the repo in ~/.ssh/.
```

---

# 6. Generate a Dedicated SSH Key

Run this on your Mac, not inside the project repository:

```bash
ssh-keygen -t ed25519 -C "oci-ubuntu-server-lab" -f ~/.ssh/oci_ubuntu_server_lab
```

This creates two files:

```text
~/.ssh/oci_ubuntu_server_lab        # Private key
~/.ssh/oci_ubuntu_server_lab.pub    # Public key
```

Meaning:

```text
oci_ubuntu_server_lab      = private key, keep on Mac
oci_ubuntu_server_lab.pub  = public key, paste into Oracle Cloud
```

Show the public key:

```bash
cat ~/.ssh/oci_ubuntu_server_lab.pub
```

Copy the public key to clipboard:

```bash
pbcopy < ~/.ssh/oci_ubuntu_server_lab.pub
```

Important:

```text
Only paste the .pub key into Oracle Cloud.
Never upload, commit, or share the private key.
```

---

# 7. Set Safe SSH Key Permissions

Set permissions on the private key:

```bash
chmod 600 ~/.ssh/oci_ubuntu_server_lab
```

Check the files:

```bash
ls -la ~/.ssh/oci_ubuntu_server_lab*
```

Expected files:

```text
~/.ssh/oci_ubuntu_server_lab
~/.ssh/oci_ubuntu_server_lab.pub
```

The private key should be readable only by the current user.

---

# 8. Create the Cloud-Init File

Create this file:

```text
cloud-init/cloud-init.yaml
```

Paste this content:

```yaml
#cloud-config
package_update: true
package_upgrade: true

packages:
  - git
  - curl
  - wget
  - unzip
  - htop
  - tmux
  - build-essential
  - python3
  - python3-pip
  - python3-venv
  - postgresql-client
  - nginx

runcmd:
  - apt-get install -y ca-certificates gnupg lsb-release
  - install -m 0755 -d /etc/apt/keyrings
  - curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
  - chmod a+r /etc/apt/keyrings/docker.asc
  - echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo $VERSION_CODENAME) stable" > /etc/apt/sources.list.d/docker.list
  - apt-get update
  - apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
  - systemctl enable docker
  - systemctl start docker
```

---

# 9. Why Cloud-Init Exists

Cloud-init prepares the server automatically during first boot.

This cloud-init file installs:

```text
Git
curl
wget
unzip
htop
tmux
build tools
Python 3
pip
venv
PostgreSQL client
Nginx
Docker
Docker Compose plugin
```

Cloud-init is useful because it makes the server setup repeatable.

Instead of manually installing the same tools every time, the server can install them during first boot.

---

# 10. Validate the Cloud-Init File

The first line must be:

```yaml
#cloud-config
```

Common cloud-init mistakes:

```text
Missing #cloud-config
Bad YAML indentation
Pasting the file as one long line
Using tabs instead of spaces
Broken repository command
```

Basic local check:

```bash
head -n 5 cloud-init/cloud-init.yaml
```

Expected:

```text
#cloud-config
package_update: true
package_upgrade: true
```

---

# 11. Pre-Networking Checklist

Before moving to OCI networking, confirm:

```text
[ ] GitHub repository exists
[ ] Local project folder exists
[ ] .gitignore protects secrets
[ ] SSH key pair exists in ~/.ssh/
[ ] Private key is not inside the repo
[ ] Public key can be copied with pbcopy
[ ] cloud-init/cloud-init.yaml exists
[ ] cloud-init starts with #cloud-config
```

---

# 12. What Comes Next

Next file:

```text
notes/02-networking-vcn-subnet-gateway.md
```

That file creates the cloud network:

```text
VCN
Public subnet
Internet Gateway
Route table
Security rules
```

Then:

```text
notes/03-compute-instance-first-login.md
```

That file creates the Ubuntu instance and logs in through SSH.

---

# 13. Summary

The local setup is ready when these are complete:

```text
1. Project repository created
2. Secret-protecting .gitignore added
3. Dedicated SSH key generated
4. Public key ready to paste into OCI
5. Private key safely stored in ~/.ssh/
6. Cloud-init file created
```

The key rule:

```text
Public key goes into Oracle Cloud.
Private key stays on your Mac.
```
