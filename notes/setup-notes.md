# OCI Ubuntu Server Lab Setup Notes

Project: `oci-ubuntu-server-lab`

Purpose: Build a headless Ubuntu 24.04 server on Oracle Cloud Infrastructure for learning Linux, SSH, Docker, PostgreSQL, Python, Git, Nginx, and data engineering infrastructure.

Official references:
- Oracle Compute instance creation docs
- Oracle VCN/networking docs
- Oracle SSH key-pair docs
- Oracle Always Free resources docs

## 1. Create the GitHub repository

Repository name:

```text
oci-ubuntu-server-lab
````

Description:

```text
Headless Ubuntu 24.04 server lab on Oracle Cloud for learning Linux, SSH, Docker, PostgreSQL, and data engineering infrastructure.
```

Visibility:

```text
Public
```

Only keep it public if no secrets, private keys, passwords, `.env` files, or Oracle credentials are committed.

Initialize with:

```text
README: On
.gitignore: Python
License: MIT License
```

## 2. Add secret protection to `.gitignore`

At the bottom of `.gitignore`, add:

```gitignore
# =========================================================
# Cloud / SSH / Infrastructure secrets
# =========================================================

# SSH private keys and key files
*.pem
*.key
id_rsa
id_rsa.pub
id_ed25519
id_ed25519.pub
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
The repo stores documentation and config templates.
The private SSH key stays outside the repo in ~/.ssh/.
```

## 3. Create the local project folder

On Mac Terminal:

```bash
mkdir oci-ubuntu-server-lab
cd oci-ubuntu-server-lab

mkdir cloud-init notes
touch README.md
touch cloud-init/cloud-init.yaml
touch notes/setup-notes.md
```

## 4. Generate a dedicated SSH key

Run this on the Mac, not inside the repo:

```bash
ssh-keygen -t ed25519 -C "tonderai-oci-ubuntu-server-lab" -f ~/.ssh/oci_ubuntu_server_lab
```

This creates:

```text
~/.ssh/oci_ubuntu_server_lab
~/.ssh/oci_ubuntu_server_lab.pub
```

Meaning:

```text
oci_ubuntu_server_lab      = private key, never share
oci_ubuntu_server_lab.pub  = public key, paste into Oracle Cloud
```

Show the public key:

```bash
cat ~/.ssh/oci_ubuntu_server_lab.pub
```

Copy it to clipboard:

```bash
pbcopy < ~/.ssh/oci_ubuntu_server_lab.pub
```

Oracle Linux/Ubuntu instances use SSH key pairs instead of passwords. The private key stays on the computer, and the public key is provided when creating the instance.

## 5. Save the cloud-init file

Create:

```text
cloud-init/cloud-init.yaml
```

Paste this:

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

Why this exists:

```text
cloud-init automatically prepares the server at first boot.
It installs the basic Linux tools, Python tools, Nginx, PostgreSQL client, and Docker.
```

## 6. Create the VCN first

Because the direct compute page may not correctly enable public IPv4, create the network separately first.

Go to:

```text
Oracle Cloud Console → Networking → Virtual Cloud Networks
```

Choose:

```text
Start VCN Wizard
```

Select:

```text
Create VCN with Internet Connectivity
```

Use:

```text
VCN name: oci-ubuntu-lab-vcn
VCN CIDR: 10.0.0.0/16
Public subnet CIDR: 10.0.0.0/24
Private subnet CIDR: 10.0.1.0/24
```

Why:

```text
The VM needs a public subnet, internet gateway, route rules, and security rules so it can be reached by SSH from the Mac.
```

## 7. Create the compute instance

Go to:

```text
Oracle Cloud Console → Compute → Instances → Create Instance
```

Use:

```text
Name: oci-ubuntu-server-lab-01
Compartment: Falconx
Availability Domain: try AD-1 first
Capacity type: On-demand
Fault domain: Let Oracle choose
```

If capacity fails, try:

```text
AD-3
AD-2
```

Do not manually choose a fault domain.

## 8. Choose image and shape

Use:

```text
Image: Canonical Ubuntu 24.04
Shape: VM.Standard.A1.Flex
OCPU: 1
RAM: 6 GB
```

Why:

```text
Ubuntu 24.04 is a modern Linux server OS.
A1.Flex is Oracle's ARM-based Always Free eligible shape.
1 OCPU / 6 GB RAM is enough for learning Linux, SSH, Docker, Python, Nginx, and PostgreSQL basics.
```

Oracle’s Always Free Ampere A1 resources are equivalent to 4 OCPUs and 24 GB RAM total, which can be split across smaller VMs.

## 9. Management settings

Use:

```text
Instance metadata authorization header: Enabled
Cloud initialization script: Paste cloud-init/cloud-init.yaml
```

Why:

```text
The metadata authorization header is a stronger security setting.
The cloud-init script installs the first set of tools automatically.
```

Make sure the cloud-init preview keeps line breaks. The first line must be:

```yaml
#cloud-config
```

## 10. Networking settings

Choose:

```text
Primary network: Select existing virtual cloud network
VCN: oci-ubuntu-lab-vcn
Subnet: public subnet
Private IPv4: Automatically assigned
Public IPv4: Automatically assigned
IPv6: Off
```

Important:

```text
Public IPv4 must be Yes.
Without a public IPv4 address, SSH from the Mac will not work directly.
```

## 11. SSH key settings

Choose:

```text
Paste public key
```

Paste the output from:

```bash
cat ~/.ssh/oci_ubuntu_server_lab.pub
```

Do not choose:

```text
No SSH keys
```

Why:

```text
This is a headless VM. SSH is the main way to log in.
```

## 12. Storage settings

Use:

```text
Boot volume: default 46.6 GB or custom 50 GB
Use in-transit encryption: Enabled
Customer-managed key: Off
Block volume: None
```

Why:

```text
The boot volume stores Ubuntu, packages, Docker, Python tools, and project files.
Default storage is enough for the first learning server.
Extra block volumes can be added later for databases or larger datasets.
```

## 13. Security rules

At first, only allow:

```text
22/tcp = SSH
```

Later, only open these when needed:

```text
80/tcp   = HTTP web traffic
443/tcp  = HTTPS web traffic
```

Do not expose PostgreSQL publicly.

```text
5432/tcp should usually stay private or be accessed through SSH tunneling.
```

## 14. Create the instance

Click:

```text
Create
```

If Oracle returns:

```text
Out of capacity for shape VM.Standard.A1.Flex
```

Then:

```text
Try another availability domain.
Keep capacity type as On-demand.
Keep fault domain as Let Oracle choose.
Try smaller RAM such as 1 OCPU / 4 GB if needed.
```

This is an Oracle capacity issue, not a setup mistake.

## 15. Connect from Mac

After the instance is running, copy the public IPv4 address.

Then run:

```bash
ssh -i ~/.ssh/oci_ubuntu_server_lab ubuntu@YOUR_PUBLIC_IP
```

Example:

```bash
ssh -i ~/.ssh/oci_ubuntu_server_lab ubuntu@152.xxx.xxx.xxx
```

If permissions are too open, fix them:

```bash
chmod 600 ~/.ssh/oci_ubuntu_server_lab
```

Then try SSH again.

## 16. First commands after login

Run:

```bash
whoami
hostname
pwd
ls -la
uname -a
df -h
free -h
ip addr
```

Check cloud-init status:

```bash
cloud-init status --long
```

Check Docker:

```bash
docker --version
docker compose version
systemctl status docker
```

Check Nginx:

```bash
systemctl status nginx
```

## 17. What this build teaches

This project teaches:

```text
Cloud VM creation
Linux server administration
SSH key authentication
Public/private networking
VCNs and subnets
Security rules and ports
Cloud-init automation
Package management
Systemd services
Docker installation
Nginx basics
PostgreSQL client tools
Infrastructure documentation
Git-based infrastructure notes
```

## 18. Final target setup

The final desired VM is:

```text
Cloud: Oracle Cloud Infrastructure
OS: Canonical Ubuntu 24.04
Shape: VM.Standard.A1.Flex
CPU: 1 OCPU
RAM: 6 GB
Storage: 46.6 GB or 50 GB
Network: Public subnet
Public IP: Yes
Access: SSH only
GUI: None
Purpose: Linux, Docker, PostgreSQL, Python, Nginx, data engineering lab
```

```

The big correction from your failed attempt is this: **create the VCN first using the “VCN with Internet Connectivity” wizard**, then create the VM using that existing public subnet. Oracle documents that the VCN wizard creates the VCN and supporting internet-connectivity pieces, while regular manual VCN creation requires you to add subnets, gateways, route rules, and security settings yourself. :contentReference[oaicite:0]{index=0}

For the VM itself, your selected shape is right: Oracle describes shapes as templates that determine OCPUs, memory, and resources, and its Always Free Ampere A1 allowance is equivalent to **4 OCPUs and 24 GB memory** total. :contentReference[oaicite:1]{index=1}

For SSH, keep the private key on your Mac and paste only the public key into Oracle; that is Oracle’s documented key-pair model for Linux instances. :contentReference[oaicite:2]{index=2}
::contentReference[oaicite:3]{index=3}
```
