# 03 Compute Instance Creation and First Login

## Goal

Create the Ubuntu compute instance after the local project setup and OCI networking are ready.

This note covers:

```text
Compute instance creation
Image and shape selection
SSH key attachment
Cloud-init attachment
Storage settings
First SSH login
First server validation
Docker and Nginx checks
VS Code Remote SSH setup
```

Project setup belongs in:

```text
notes/01-project-setup-ssh-cloud-init.md
```

Networking setup belongs in:

```text
notes/02-networking-vcn-subnet-gateway.md
```

Troubleshooting belongs in:

```text
notes/04-troubleshooting.md
```

---

# 1. Required Setup Before Creating Compute

Before creating the compute instance, these should already exist:

```text
Project folder
SSH key pair
cloud-init/cloud-init.yaml
VCN
Public subnet
Internet Gateway
Route table with 0.0.0.0/0 → Internet Gateway
Security rule allowing TCP 22
```

Minimum checklist:

```text
[ ] SSH private key exists at ~/.ssh/oci_ubuntu_server_lab
[ ] SSH public key exists at ~/.ssh/oci_ubuntu_server_lab.pub
[ ] cloud-init file exists
[ ] VCN exists
[ ] Public subnet exists
[ ] Public subnet allows public IP addresses
[ ] Internet Gateway exists and is enabled
[ ] Route table has 0.0.0.0/0 → Internet Gateway
[ ] Security List or NSG allows TCP 22
```

---

# 2. Create the Compute Instance

Go to:

```text
Oracle Cloud Console → Compute → Instances → Create Instance
```

Use:

```text
Name: oci-ubuntu-server-lab-01
Compartment: Your selected compartment
Availability Domain: Try AD-1 first
Capacity Type: On-demand
Fault Domain: Let Oracle choose
```

If capacity fails, try:

```text
AD-2
AD-3
```

Do not manually choose a fault domain unless there is a specific reason.

---

# 3. Choose Image

Recommended image:

```text
Canonical Ubuntu 24.04
```

Why:

```text
Ubuntu 24.04 is a modern Linux server OS.
It is good for learning Linux, Docker, Python, Nginx, Git, and data engineering tools.
```

For this lab, the default SSH user is:

```text
ubuntu
```

---

# 4. Choose Shape

Recommended shape:

```text
VM.Standard.A1.Flex
```

Recommended starting resources:

```text
OCPU: 1
RAM: 4–6 GB
```

Good first setup:

```text
OCPU: 1
RAM: 6 GB
```

If capacity is limited, try:

```text
OCPU: 1
RAM: 4 GB
```

Why:

```text
A1.Flex is ARM-based and Always Free eligible.
1 OCPU and 4–6 GB RAM is enough for Linux, SSH, Docker, Python, Nginx, and PostgreSQL client practice.
```

---

# 5. Management Settings

Use:

```text
Instance metadata authorization header: Enabled
```

For cloud-init:

```text
Cloud initialization script: Paste cloud-init/cloud-init.yaml
```

Important:

```text
The first line of cloud-init must be:
#cloud-config
```

Make sure line breaks are preserved.

Do not paste the cloud-init file as one long line.

---

# 6. Networking Settings

Choose:

```text
Primary Network: Select existing virtual cloud network
VCN: oci-ubuntu-lab-vcn
Subnet: public-subnet-01
Private IPv4 Address: Automatically assigned
Public IPv4 Address: Automatically assigned
IPv6: Off
```

Important:

```text
Public IPv4 Address must be enabled.
```

Without a public IPv4 address, direct SSH from the Mac will not work.

After creation, the instance should show:

```text
Public IPv4 address: Present
Private IPv4 address: Present
Subnet: public-subnet-01
Route table: Public route table
```

---

# 7. SSH Key Settings

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

Do not paste the private key.

Correct key usage:

```text
Public key  → goes into Oracle Cloud
Private key → stays on Mac in ~/.ssh/
```

---

# 8. Storage Settings

Recommended boot volume:

```text
Boot volume: Default size or 50 GB
Use in-transit encryption: Enabled
Customer-managed key: Off
Block volume: None
```

Why:

```text
The boot volume stores Ubuntu, packages, Docker, Python tools, logs, and project files.
Default storage is enough for the first lab.
Extra block volumes can be added later for databases or datasets.
```

For the first lab, avoid adding extra block volumes unless needed.

---

# 9. Security Reminder Before Create

At minimum, the network must allow:

```text
TCP 22 = SSH
```

Later, only open these when needed:

```text
TCP 80  = HTTP
TCP 443 = HTTPS
```

Do not expose PostgreSQL publicly:

```text
TCP 5432 should not be open to 0.0.0.0/0.
```

Better database access options:

```text
SSH tunneling
Private subnet
VPN
Bastion host
Docker internal network
```

---

# 10. Create the Instance

Click:

```text
Create
```

Wait until the instance state becomes:

```text
Running
```

Record these values in private notes:

```text
Instance name
Public IPv4 address
Private IPv4 address
Subnet name
Route table name
Image
Shape
OCPU and RAM
Boot volume size
```

Do not put real public IPs, OCIDs, or sensitive configuration details in a public repository unless you intentionally want them visible.

---

# 11. Test Port 22 Before SSH

From Mac Terminal, test whether SSH traffic can reach the instance:

```bash
nc -vz <PUBLIC_IP> 22
```

Good result:

```text
Connection to <PUBLIC_IP> port 22 succeeded
```

Bad result:

```text
Operation timed out
```

If the test times out, check networking first.

Common causes:

```text
Missing route rule
Security rule missing
NSG blocking SSH
No public IP
Subnet is private
Internet Gateway missing or disabled
Instance still booting
```

---

# 12. Connect from Mac

Set private key permissions:

```bash
chmod 600 ~/.ssh/oci_ubuntu_server_lab
```

SSH into Ubuntu:

```bash
ssh -i ~/.ssh/oci_ubuntu_server_lab ubuntu@<PUBLIC_IP>
```

For Ubuntu images, use:

```text
ubuntu
```

For Oracle Linux images, use:

```text
opc
```

For this lab, use:

```bash
ssh -i ~/.ssh/oci_ubuntu_server_lab ubuntu@<PUBLIC_IP>
```

When asked:

```text
Are you sure you want to continue connecting?
```

Type:

```text
yes
```

---

# 13. Confirm Login

After login, run:

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

Expected user:

```text
ubuntu
```

Check operating system:

```bash
cat /etc/os-release
```

Check routing from inside the server:

```bash
ip route
```

---

# 14. Check Cloud-Init Status

Run:

```bash
cloud-init status --long
```

If cloud-init is still running, wait before changing too much.

Check logs:

```bash
sudo tail -n 100 /var/log/cloud-init.log
sudo tail -n 100 /var/log/cloud-init-output.log
```

Search for errors:

```bash
sudo grep -i error /var/log/cloud-init.log
sudo grep -i failed /var/log/cloud-init.log
```

Common cloud-init issues:

```text
Bad YAML indentation
Missing #cloud-config first line
Broken package repository
Network not ready during first boot
Docker repository setup failed
```

---

# 15. Check Installed Tools

Check Git:

```bash
git --version
```

Check Python:

```bash
python3 --version
pip3 --version
```

Check PostgreSQL client:

```bash
psql --version
```

Check Nginx:

```bash
systemctl status nginx
```

Check Docker:

```bash
docker --version
docker compose version
systemctl status docker
```

---

# 16. Allow Ubuntu User to Run Docker

If Docker was installed but the `ubuntu` user cannot run Docker without `sudo`, run:

```bash
sudo usermod -aG docker ubuntu
```

Then log out:

```bash
exit
```

Log back in:

```bash
ssh -i ~/.ssh/oci_ubuntu_server_lab ubuntu@<PUBLIC_IP>
```

Test Docker:

```bash
docker run hello-world
```

If that works, Docker is installed and the user has Docker access.

---

# 17. Optional SSH Shortcut

On Mac, edit SSH config:

```bash
nano ~/.ssh/config
```

Add:

```sshconfig
Host oci-ubuntu-lab
    HostName <PUBLIC_IP>
    User ubuntu
    IdentityFile ~/.ssh/oci_ubuntu_server_lab
```

Then connect with:

```bash
ssh oci-ubuntu-lab
```

This is cleaner than typing the full SSH command every time.

---

# 18. VS Code Remote SSH

Install the VS Code extension:

```text
Remote - SSH
```

Then connect to:

```text
oci-ubuntu-lab
```

Expected flow:

```text
VS Code
→ Remote SSH
→ oci-ubuntu-lab
→ Ubuntu server
```

This allows editing files on the server from VS Code without installing a desktop GUI on the server.

---

# 19. First Server Update

After confirming login, run:

```bash
sudo apt update
sudo apt upgrade -y
```

Then reboot:

```bash
sudo reboot
```

After reboot, reconnect:

```bash
ssh -i ~/.ssh/oci_ubuntu_server_lab ubuntu@<PUBLIC_IP>
```

or, if SSH config was created:

```bash
ssh oci-ubuntu-lab
```

---

# 20. Basic Server Health Checks

After reconnecting, run:

```bash
uptime
df -h
free -h
systemctl --failed
```

Check whether important services are running:

```bash
systemctl status ssh
systemctl status nginx
systemctl status docker
```

Check open listening ports:

```bash
sudo ss -tulpn
```

---

# 21. Optional Nginx Test

If Nginx is installed, test locally from the server:

```bash
curl -I http://localhost
```

Expected result should include something like:

```text
HTTP/1.1 200 OK
```

To view Nginx from a browser using the public IP, OCI security rules must allow:

```text
TCP 80
```

For HTTPS later, allow:

```text
TCP 443
```

Do not open these until needed.

---

# 22. Optional Docker Test

Test Docker:

```bash
docker run hello-world
```

Check Docker system information:

```bash
docker info
```

Check Docker Compose:

```bash
docker compose version
```

---

# 23. What to Document Privately

In private notes, record:

```text
Instance public IP
Instance private IP
Compartment
VCN name
Subnet name
Route table name
Security List or NSG name
Image
Shape
Boot volume size
SSH key name
Date created
```

Avoid putting these in a public repo unless intentionally public:

```text
Real public IP
OCIDs
Account-specific identifiers
Private keys
Oracle config files
API keys
```

---

# 24. Final Target Setup

The desired VM setup is:

```text
Cloud: Oracle Cloud Infrastructure
OS: Canonical Ubuntu 24.04
Shape: VM.Standard.A1.Flex
CPU: 1 OCPU
RAM: 4–6 GB
Storage: Default boot volume or 50 GB
Network: Public subnet
Public IP: Yes
Access: SSH only at first
GUI: None
Purpose: Linux, Docker, PostgreSQL, Python, Nginx, data engineering lab
```

---

# 25. What This Build Teaches

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

---

# 26. Summary

The compute instance works when these pieces are correct:

```text
1. Ubuntu image selected
2. Shape selected
3. Public subnet selected
4. Public IPv4 assigned
5. Public SSH key pasted into OCI
6. Private SSH key kept on Mac
7. TCP 22 open in security rules
8. Route table points 0.0.0.0/0 to Internet Gateway
```

Final login command:

```bash
ssh -i ~/.ssh/oci_ubuntu_server_lab ubuntu@<PUBLIC_IP>
```

After optional SSH config setup:

```bash
ssh oci-ubuntu-lab
```
